# Object Detection Optimization

Covers detection architecture tradeoffs (YOLO / Faster R-CNN / DETR family), detection-specific tuning (NMS, anchors), and the model-compression path (quantization, pruning, distillation) referenced in SKILL.md's Workflow 2 (Model Optimization and Deployment). This is the detail behind the numbers in SKILL.md's Architecture Selection Guide table.

## Table of Contents
- [Detection Architecture Families](#detection-architecture-families)
- [Anchor-Based vs Anchor-Free](#anchor-based-vs-anchor-free)
- [NMS Tuning and Alternatives](#nms-tuning-and-alternatives)
- [Small Object Detection](#small-object-detection)
- [Quantization](#quantization)
- [Pruning](#pruning)
- [Knowledge Distillation](#knowledge-distillation)
- [Edge Deployment Latency Numbers](#edge-deployment-latency-numbers)
- [Putting It Together: An Optimization Decision Tree](#putting-it-together-an-optimization-decision-tree)

## Detection Architecture Families

### YOLO family (single-stage, anchor-free since v8)

| Model | Params | FLOPs | mAP@50:95 (COCO) | Latency (A100, TensorRT FP16, 640px) |
|---|---|---|---|---|
| YOLOv8n | 3.2M | 8.7G | 37.3 | ~1.2ms |
| YOLOv8s | 11.2M | 28.6G | 44.9 | ~2.1ms |
| YOLOv8m | 25.9M | 78.9G | 50.2 | ~4.2ms |
| YOLOv8l | 43.7M | 165.2G | 52.9 | ~6.8ms |
| YOLOv8x | 68.2M | 257.8G | 53.9 | ~10.1ms |
| YOLOv11m | 20.1M | 68.0G | 51.5 | ~4.0ms |

YOLOv8/v11 dropped the explicit anchor boxes used through v5, moving to an anchor-free, decoupled head (separate branches for classification and box regression) with task-aligned assignment (TAL) for matching predictions to ground truth. This removed the anchor-hyperparameter tuning burden (anchor scale/ratio per dataset) that plagued YOLOv3-v5 on datasets with unusual object aspect ratios. YOLOv11 trims params ~15-20% versus v8 at equivalent mAP through a redesigned C3k2 block and depthwise convs in the neck — worth the migration for edge deployment even without an accuracy gain.

### Faster R-CNN (two-stage, anchor-based)

| Backbone | mAP@50:95 | Latency (V100, PyTorch FP32) |
|---|---|---|
| ResNet-50-FPN | 40.2 | ~46ms |
| ResNet-101-FPN | 42.0 | ~65ms |
| ResNeXt-101-FPN | 43.3 | ~85ms |

Two stages: a Region Proposal Network (RPN) generates ~1000-2000 candidate boxes against a dense anchor grid (typically 3 scales x 3 ratios per location), then an ROI head classifies and refines each proposal. This gives better localization precision than single-stage detectors at the cost of 5-10x latency, because the second stage runs full classification+regression per-proposal rather than densely across the whole feature map once. Still the reference architecture when small, dense, or highly-overlapping objects dominate the dataset and latency isn't the binding constraint (e.g., offline batch processing, medical imaging, satellite imagery).

### DETR family (end-to-end transformer, no NMS)

| Model | mAP@50:95 | Latency | Notes |
|---|---|---|---|
| DETR-R50 | 42.0 | ~50ms (V100) | Original 2020 design, needs 500 epochs to converge |
| Deformable DETR | 46.2 | ~40ms | Deformable attention fixes DETR's slow convergence and poor small-object mAP |
| DINO-4scale | 49.0 | ~85ms | Contrastive denoising query training, current DETR-line SOTA-adjacent |
| RT-DETR-L | 53.0 | ~5.3ms (TensorRT FP16) | Hybrid CNN+transformer encoder, real-time-viable |

DETR reframes detection as direct set prediction: a fixed number of learned object queries attend over image features via a transformer decoder, and a bipartite matching loss (Hungarian algorithm) assigns each query to a ground-truth object or "no object." This eliminates NMS and anchor design entirely, but the original DETR needed ~500 training epochs (vs ~12-36 for Faster R-CNN) because the query-to-object assignment is unstable early in training, and it was notably weak on small objects since it only used the final, coarse feature map. Deformable DETR fixed both by using deformable attention over multi-scale features and cut training to ~50 epochs. RT-DETR is the one to reach for when you want DETR's NMS-free simplicity but need real-time latency — it's the recommended transformer choice in SKILL.md's Step 2 architecture table for exactly this reason.

## Anchor-Based vs Anchor-Free

| Aspect | Anchor-based (Faster R-CNN, YOLOv5) | Anchor-free (YOLOv8+, FCOS, CenterNet) |
|---|---|---|
| Hyperparameters | Anchor scales/ratios must be tuned per dataset (k-means on box sizes is standard practice) | None — head predicts box params directly per location or via TAL assignment |
| Convergence speed | Faster early on (dense priors give more positive samples per image) | Slightly slower early, comparable by convergence |
| Small/extreme aspect-ratio objects | Poor if anchors don't match true box distribution | Generally more robust, no prior box shape assumption |
| Multi-object same-center ambiguity | Handled naturally (different anchors, different objects) | Needs an assignment strategy (FCOS uses center-ness + FPN level assignment; YOLOv8 uses TAL) |

Practical implication: if migrating a Faster R-CNN pipeline that was heavily anchor-tuned for a specific dataset (e.g., aerial imagery with elongated objects), expect an anchor-free replacement to need dataset-specific validation of assignment strategy rather than anchor k-means — the failure mode shifts from "wrong anchor shapes" to "ambiguous center assignment when small objects cluster."

## NMS Tuning and Alternatives

Standard greedy NMS parameters and what they control:

```yaml
conf_thres: 0.25   # discard boxes below this confidence before NMS — lower = more recall, more false positives
iou_thres: 0.45    # boxes overlapping above this IoU with a higher-score box are suppressed
max_det: 300       # cap on detections per image (COCO eval convention)
```

Tuning guidance:
- **Dense/overlapping objects (crowds, cell counting):** lower `iou_thres` to 0.3-0.4 causes over-suppression of genuinely separate overlapping objects — use **Soft-NMS** instead, which decays confidence by overlap rather than hard-discarding (`score *= exp(-iou² / sigma)`, typically `sigma=0.5`), recovering 1-2 mAP on crowded datasets like CrowdHuman.
- **Sparse, well-separated objects:** default `iou_thres=0.45-0.5, conf_thres=0.25` is fine; raising `conf_thres` to 0.4-0.5 trades a little recall for meaningfully fewer false positives in production alerting use cases.
- **Class-agnostic vs per-class NMS:** per-class NMS (default in most frameworks) avoids suppressing a "person" box because it overlaps a "backpack" box at high IoU — always use per-class NMS unless classes are mutually exclusive by construction.
- **DIoU-NMS:** replaces IoU with Distance-IoU in the suppression criterion, slightly better for occluded objects with similar box centers; ~0.3-0.5 mAP gain reported on occlusion-heavy benchmarks, negligible extra compute.
- **NMS-free (DETR family):** by construction there's no NMS step — the bipartite matching loss during training already enforces one query per object. This removes a hand-tuned hyperparameter and a CPU-bound post-processing bottleneck (NMS on CPU can be 2-5ms per image at high object density, which matters when GPU inference itself is under 5ms — NMS becomes the bottleneck).

## Small Object Detection

Small objects (COCO defines "small" as area <32²px) are the most persistent detection failure mode. Approaches, from SKILL.md's Step 2 table and beyond:

- **FPN (Feature Pyramid Network):** fuses high-resolution/low-semantic early features with low-resolution/high-semantic late features via lateral connections + upsampling, so detection heads at multiple scales each see appropriately-sized objects. Standard in Faster R-CNN and YOLO; without it, small-object AP on COCO typically drops 15-20 points relative to overall AP.
- **SAHI (Slicing Aided Hyper Inference):** tiles a large image into overlapping patches, runs the detector on each patch at full resolution, then merges detections back with NMS across the full image. Effective when the base model's input resolution (e.g., 640px) forces small objects in a large image (e.g., 4K aerial photo) to shrink below detectable size after resize. Costs are linear in tile count — a 4x4 tiling means ~16x the inference calls, so this is a latency-for-recall tradeoff usually reserved for offline/batch pipelines.
- **Higher input resolution:** training/inference at 1280px instead of 640px roughly doubles small-object visibility but also roughly doubles compute (quadratic in resolution for CNNs) — YOLOv8m at 1280px runs at roughly a third the FPS of 640px.
- **Multi-scale training/TTA:** random-resize augmentation during training, and multi-scale test-time inference (run at 640/800/1024, merge with NMS) at eval/serving time when latency budget allows ensembling.

## Quantization

| Precision | Relative size | Relative speed (GPU, TensorRT) | Typical mAP drop | Notes |
|---|---|---|---|---|
| FP32 | 100% | 1x | 0% | Baseline, rarely deployed as final target |
| FP16 | 50% | 1.5-2x | <0.5% | Nearly free win on any modern NVIDIA GPU (Tensor Cores); default deployment precision |
| INT8 (PTQ, calibrated) | 25% | 2-4x | 1-3% | Needs 300-1000 representative calibration images; sensitive layers (e.g., detection head's final regression) sometimes kept FP16 in mixed precision |
| INT8 (QAT) | 25% | 2-4x | 0.3-1% | Fine-tunes with fake-quantization ops during training; recovers most of PTQ's accuracy loss at the cost of a retraining pass |
| INT4/mixed | ~12.5% | 3-6x (hardware dependent) | 3-8%+ | Rarely worth it for detection; usable for classification backbones on aggressive edge targets |

Calibration matters more than the algorithm: INT8 PTQ accuracy loss is dominated by how representative the calibration set is of production data distribution (lighting, object scale, background) — a calibration set drawn only from daytime images and deployed on a 24/7 camera feed will show much worse than the "1-3%" figure at night.

## Pruning

- **Unstructured pruning** (zero out individual weights below a magnitude threshold) achieves high sparsity (50-90%) with minimal accuracy loss but doesn't speed up inference on standard GPU kernels without sparse-tensor hardware support (NVIDIA Ampere+ 2:4 structured sparsity is an exception, giving up to 2x speedup with specific 2-out-of-4 sparsity patterns).
- **Structured pruning** (remove entire channels/filters) directly reduces FLOPs and is hardware-agnostic. Typical results on YOLO-scale detectors: 30-50% channel pruning with fine-tuning recovers to within 1-2 mAP of the unpruned model while giving a roughly proportional latency reduction (e.g., 40% channel reduction → ~30-35% latency reduction, sub-linear due to fixed overheads).
- **Practical workflow:** train dense → prune (magnitude or L1-norm based channel importance) → fine-tune for 10-20% of original training epochs → repeat pruning iteratively rather than pruning to target sparsity in one shot, which tends to lose 2-3x more accuracy for the same final sparsity.

## Knowledge Distillation

Distilling a large teacher (e.g., YOLOv8x, mAP 53.9) into a small student (e.g., YOLOv8n, baseline mAP 37.3) via soft-label matching on classification logits and feature-map mimicry on intermediate layers typically recovers 1.5-3 mAP points on the student versus training it from scratch on hard labels alone — i.e., a distilled YOLOv8n might land around 38.5-40 mAP instead of 37.3, for zero extra inference cost since only the student ships. This is the standard lever when the accuracy target requires more than the smallest model gives but the latency budget forbids the next size up.

## Edge Deployment Latency Numbers

Representative numbers (single image, batch=1, FP16 unless noted) for a YOLOv8-class model at 640px:

| Hardware | Precision | Latency | Notes |
|---|---|---|---|
| NVIDIA A100 | FP16 (TensorRT) | 1.2-4.2ms (n to m) | Cloud/server tier |
| NVIDIA Jetson Orin NX | FP16 (TensorRT) | 8-15ms | Common robotics/AMR edge target |
| NVIDIA Jetson Nano | INT8 (TensorRT) | 35-60ms | Legacy low-power edge, often needs YOLOv8n + INT8 to hit real-time |
| Apple M-series (Neural Engine, CoreML) | FP16 | 5-10ms | ANE handles conv-heavy models very efficiently |
| Intel CPU (i7, OpenVINO) | INT8 | 15-30ms | Depends heavily on core count and AVX-512 availability |
| Raspberry Pi 4 (CPU only, TFLite) | INT8 | 200-400ms | Effectively unusable for real-time; needs a coral/NPU accelerator |
| Google Coral Edge TPU | INT8 | 8-12ms | Requires full INT8 quantization, some ops must be TPU-compatible or fall back to CPU |

The consistent pattern: crossing from cloud GPU to any edge target costs roughly 5-15x latency at equal precision, and INT8 quantization is usually mandatory (not optional) to hit real-time (>15 FPS) on anything below a Jetson Orin-class device.

## Putting It Together: An Optimization Decision Tree

1. Is baseline PyTorch FP32 already within budget? If yes, stop — export to ONNX/TensorRT FP16 anyway for headroom, but no further compression needed.
2. Is the gap under 2x? Apply FP16 (near-free) then re-measure.
3. Still short, and target is GPU? Apply INT8 PTQ with a representative calibration set; if accuracy drop exceeds tolerance, switch to QAT.
4. Still short, or target is CPU/mobile/embedded? Consider structured pruning (30-50% channels) + fine-tune, and/or drop to a smaller architecture variant with distillation from the current model as teacher.
5. Still short after all of the above? The architecture choice itself is likely wrong for the constraint — move down SKILL.md's Architecture Selection Guide table (e.g., YOLOv8m → YOLOv8n, or Faster R-CNN → YOLO) rather than continuing to compress a fundamentally oversized model.
