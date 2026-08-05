# Production Vision Systems

Deployment, serving, and monitoring guidance for the vision models produced by SKILL.md's training and optimization workflows. Assumes models have already been exported per `object_detection_optimization.md` (ONNX/TensorRT/OpenVINO/CoreML) and covers what happens after that: serving architecture, throughput math, and keeping a model healthy in production.

## Table of Contents
- [Serving Architecture Choices](#serving-architecture-choices)
- [Triton Inference Server](#triton-inference-server)
- [TorchServe](#torchserve)
- [ONNX Runtime as a Serving Layer](#onnx-runtime-as-a-serving-layer)
- [Batching Strategies](#batching-strategies)
- [GPU Utilization Math](#gpu-utilization-math)
- [Throughput and Capacity Planning](#throughput-and-capacity-planning)
- [Monitoring for Drift and Degradation](#monitoring-for-drift-and-degradation)
- [Production Incident Patterns](#production-incident-patterns)

## Serving Architecture Choices

| Serving option | Best for | Weak point |
|---|---|---|
| Triton Inference Server | Multi-model, multi-framework, GPU-heavy production fleets | Operational complexity (Kubernetes/config-heavy) |
| TorchServe | PyTorch-only shops wanting less setup than Triton | Fewer backend/runtime options, weaker dynamic batching than Triton |
| ONNX Runtime (embedded/standalone) | Single-model services, CPU or mixed CPU/GPU fleets, simplest ops story | No built-in model-management/multi-model routing — you build that layer yourself |
| Custom FastAPI/Flask + TensorRT engine | Ultra-low-latency single model, full control | You own batching, health checks, and versioning yourself |

The choice mostly comes down to fleet shape: one team serving five different detector/segmentation model families across GPU nodes wants Triton's model repository abstraction; a team serving one YOLO model behind a REST API is usually over-engineering by reaching for Triton and better served by ONNX Runtime or TorchServe.

## Triton Inference Server

Model repository layout:

```
model_repository/
└── yolov8_detector/
    ├── config.pbtxt
    └── 1/
        └── model.onnx
```

`config.pbtxt` — the parts that matter in practice:

```protobuf
name: "yolov8_detector"
platform: "onnxruntime_onnx"
max_batch_size: 16

input [
  { name: "images", data_type: TYPE_FP32, dims: [3, 640, 640] }
]
output [
  { name: "output0", data_type: TYPE_FP32, dims: [84, 8400] }
]

dynamic_batching {
  preferred_batch_size: [4, 8, 16]
  max_queue_delay_microseconds: 5000   # cap latency added by waiting to fill a batch
}

instance_group [
  { count: 2, kind: KIND_GPU, gpus: [0] }   # 2 model instances sharing GPU 0 for concurrency
]

optimization {
  execution_accelerators {
    gpu_execution_accelerator: [
      { name: "tensorrt", parameters { key: "precision_mode" value: "FP16" } }
    ]
  }
}
```

Launch and query:

```bash
docker run --gpus=1 -p 8000:8000 -p 8001:8001 -p 8002:8002 \
    -v $(pwd)/model_repository:/models \
    nvcr.io/nvidia/tritonserver:24.05-py3 \
    tritonserver --model-repository=/models

curl -X POST localhost:8000/v2/models/yolov8_detector/infer -d @request.json
```

Key knobs and why they matter:
- `dynamic_batching.max_queue_delay_microseconds` trades latency for throughput — 5ms of queuing delay to opportunistically fill a batch of 8 can 2-3x throughput versus batch=1, but adds a hard latency floor of 5ms per request even under low load. For a P99-latency-sensitive endpoint, this needs to be measured against the SLA, not left at a default.
- `instance_group.count` > 1 on one GPU lets Triton run multiple model copies concurrently (useful when a single instance can't saturate the GPU — small models, low batch sizes), but doesn't help (and can hurt via context-switch/memory pressure) once one instance already saturates GPU compute.
- Model ensembles (`platform: "ensemble"`) chain a detection model → a crop/preprocessing step → a classification model server-side, avoiding a network round-trip between stages of a multi-model pipeline (e.g., detect-then-classify pipelines are common in this skill's segmentation/detection combo workflows).

## TorchServe

```bash
torch-model-archiver --model-name yolov8m --version 1.0 \
    --serialized-file yolov8m.torchscript --handler object_detector \
    --export-path model_store

torchserve --start --model-store model_store --models yolov8m.mar --ts-config config.properties
```

`config.properties` batching config:

```properties
batch_size=8
max_batch_delay=100
number_of_gpu=1
default_workers_per_model=2
```

TorchServe's batching is coarser-grained than Triton's — `max_batch_delay` (ms) is the analog of Triton's queue delay but there's less fine control over per-shape batch preferences. It's the right choice when the team is already fully PyTorch/TorchScript-native and wants to avoid maintaining an ONNX/TensorRT export pipeline just to get decent serving, at the cost of somewhat lower peak throughput than a tuned Triton+TensorRT setup (commonly 20-40% lower GPU throughput at equal batch size, since TorchServe doesn't get TensorRT-level kernel fusion).

## ONNX Runtime as a Serving Layer

For a single-model service, wrapping ONNX Runtime directly in a lightweight API server is often simpler than standing up Triton:

```python
import onnxruntime as ort
from fastapi import FastAPI
import numpy as np

app = FastAPI()
sess = ort.InferenceSession(
    "model.onnx",
    providers=[("TensorrtExecutionProvider", {"trt_fp16_enable": True}),
               "CUDAExecutionProvider", "CPUExecutionProvider"],
)

sess.get_session_options().intra_op_num_threads = 4  # relevant mainly for CPU EP

@app.post("/infer")
def infer(payload: dict):
    arr = np.array(payload["image"], dtype=np.float32)
    outputs = sess.run(None, {"images": arr})
    return {"detections": outputs[0].tolist()}
```

Caveat worth flagging: this pattern has no built-in batching — every request runs as `batch_size=1` unless you add a batching layer yourself (e.g., a queue that accumulates requests for a few ms before calling `sess.run`). Under concurrent load this leaves GPU throughput far below what dynamic batching in Triton/TorchServe would achieve — fine for low-QPS services, a real bottleneck above roughly 20-30 QPS on a single GPU.

## Batching Strategies

| Strategy | Latency impact | Throughput impact | When to use |
|---|---|---|---|
| No batching (batch=1 always) | Lowest, most predictable per-request latency | Poor GPU utilization at low-medium QPS | Strict low-latency SLA (e.g., <20ms P99), or QPS low enough that GPU is never the bottleneck |
| Static batching (fixed batch size, wait to fill) | Adds up to one full batch-fill period of latency | Best throughput at steady high QPS | Offline/batch processing, or steady high-QPS online traffic |
| Dynamic batching (Triton-style, max queue delay) | Bounded added latency (the queue delay cap) | Near-static-batching throughput with much better latency control | Most production online-serving cases — the default recommendation |
| Adaptive/continuous batching | Minimal added latency, high throughput | Complex to implement correctly | High-variance traffic patterns, justifies the complexity only at real scale |

Rule of thumb from the benchmark numbers in SKILL.md's Workflow 2: a YOLOv8m goes from 45.2ms/22.1 FPS at batch=1 to 165.3ms/48.4 FPS at batch=8 — batch=8 more than doubles throughput but each individual request's batch-processing time went up too, so a request unlucky enough to be first-in-batch can wait for the full 165ms if there's no separate low-latency path. This is precisely why bounding `max_queue_delay` matters — a batch size chosen purely by throughput benchmarking can quietly blow a latency SLA.

## GPU Utilization Math

Given a model's measured throughput per batch size, GPU utilization for a target QPS is estimated as:

```
required_gpu_time_per_second = QPS * (per_request_latency_at_chosen_batch_size / batch_size)
gpu_utilization ≈ required_gpu_time_per_second / 1.0 second
```

Worked example using SKILL.md's own benchmark numbers (YOLOv8m, batch=8: 165.3ms for 8 images → 20.66ms per image):

```
Target: 200 QPS
required_gpu_time_per_second = 200 * 0.02066s = 4.13 seconds of GPU-time needed per wall-clock second
→ requires ~5 GPUs (with headroom) running this configuration to sustain 200 QPS, or
→ switch to a smaller model / more aggressive quantization to reduce per-image time
```

A GPU reporting 100% utilization in `nvidia-smi` does not mean it's compute-bound in a way further optimization can't help — memory-bandwidth-bound layers (common in depthwise convs, some attention operators) can max out the "utilization" metric while leaving significant Tensor Core throughput unused. `nsys profile` or Triton's per-model latency breakdown (queue time vs compute time vs output time) is needed to tell the difference between "genuinely compute-saturated, buy more GPUs" and "batching/kernel-launch overhead dominated, fix the pipeline instead."

```bash
# Rough compute-vs-memory-bound diagnosis
nsys profile -o profile_report python infer_loop.py
nsys stats profile_report.nsys-rep --report cuda_gpu_kern_sum
```

## Throughput and Capacity Planning

Checklist for sizing a fleet:
1. Measure P50/P95/P99 latency at the batch size you intend to run in production — not batch=1 numbers from a benchmarking script, which SKILL.md's own optimizer output reports separately from real dynamic-batch behavior.
2. Multiply by expected peak QPS (not average — provisioning for average QPS guarantees P99 SLA violations during traffic spikes).
3. Add 30-50% headroom for autoscaling lag, failed-instance failover, and non-inference overhead (preprocessing, image decode, network).
4. Re-benchmark after every optimization step (quantization, pruning, batching config change) — compounding gains from `object_detection_optimization.md` (e.g., FP16 + structured pruning + better batching) are not simply multiplicative in practice; measure the combined configuration, don't just multiply individual speedup factors.

## Monitoring for Drift and Degradation

A vision model's failure modes in production are quieter than a crash — they show up as slowly declining precision/recall on data that no longer matches training distribution. Concrete signals to track:

- **Confidence score distribution drift:** track the rolling histogram of max-class confidence scores across production predictions. A shift toward the model's decision boundary (e.g., median confidence dropping from 0.85 to 0.6 over weeks) without a code change usually indicates input distribution drift — new camera hardware, lighting changes, a new object variant not in training data.
- **Detection count anomalies:** sudden spikes or drops in average detections-per-image for a fixed camera/scene are often the first visible sign of either a model regression or an environmental change (e.g., a security camera detector suddenly reporting near-zero detections at night after a firmware update changed IR illumination).
- **Class distribution shift:** track the proportion of each class among all detections over time; a shift (e.g., "bicycle" detections dropping from 15% to 2% of all detections) can indicate either a genuine environmental change or the model silently failing on that class due to a subtle preprocessing bug (wrong resize interpolation, channel-order swap).
- **Latency percentile tracking (P50/P95/P99), not just averages:** GPU serving latency has heavier tails than typical web services because of batching-queue effects; P99 latency creeping up while P50 stays flat is a classic sign of an occasional slow path (e.g., dynamic-batch queue starvation, a co-scheduled job stealing GPU time) worth catching before it becomes an SLA breach.
- **Shadow/canary evaluation on a held-out labeled trickle:** periodically route a small sample of production traffic through a human-labeling step (or an active-learning uncertainty sampler that flags low-confidence detections for review) to get a real accuracy signal in production, since raw inference doesn't come with ground truth.
- **Golden-set regression testing on every deploy:** re-run the current model against a fixed, versioned set of ~200-500 representative labeled images before every deploy and diff mAP against the previous version — catches export-pipeline bugs (e.g., an ONNX opset upgrade silently changing NMS behavior) that unit tests on isolated components miss.

Example lightweight drift check using running statistics:

```python
import numpy as np

class ConfidenceDriftMonitor:
    def __init__(self, baseline_median, alert_threshold=0.15):
        self.baseline_median = baseline_median
        self.alert_threshold = alert_threshold
        self.window = []

    def update(self, confidences: list[float]):
        self.window.extend(confidences)
        if len(self.window) >= 1000:
            current_median = np.median(self.window)
            drift = abs(current_median - self.baseline_median) / self.baseline_median
            if drift > self.alert_threshold:
                self._alert(current_median, drift)
            self.window = []

    def _alert(self, current_median, drift):
        print(f"ALERT: confidence median drifted {drift:.1%} "
              f"(baseline={self.baseline_median:.3f}, current={current_median:.3f})")
```

## Production Incident Patterns

Patterns worth pattern-matching against when something breaks:

- **"Accuracy dropped after an ONNX/TensorRT re-export, nothing else changed":** almost always an opset mismatch, an NMS implementation difference between the training framework and the export target, or a preprocessing mismatch (mean/std normalization values or channel order silently differing between the PyTorch preprocessing and the ONNX Runtime input pipeline). Diff outputs layer-by-layer between PyTorch and ONNX on a fixed input before assuming it's a model problem.
- **"Model works in staging, degrades in production within days":** check for a training/serving skew in preprocessing (e.g., staging uses PIL resize, production uses OpenCV resize — different default interpolation) before assuming data drift.
- **"Latency P99 spikes correlate with time-of-day":** usually co-tenancy — another job or autoscaled workload sharing the GPU node during peak hours. `nvidia-smi` process list during the spike window, not just aggregate utilization, is the fastest way to confirm.
- **"INT8 model has much worse accuracy than the 1-3% budgeted":** check the calibration dataset's representativeness first (see `object_detection_optimization.md`'s Quantization section) — this is the single most common root cause, well above "INT8 is fundamentally too lossy for this model."
