# Reference: Commands and APIs for Computer Vision Engineering

Practical cheat-sheet for the workflows in SKILL.md: dataset prep, training, benchmarking, export, and deployment. Every snippet below is something you'd actually type or paste into a config file, not pseudocode.

## Table of Contents
- [Ultralytics YOLO CLI and Python API](#ultralytics-yolo-cli-and-python-api)
- [Detectron2](#detectron2)
- [MMDetection](#mmdetection)
- [Dataset Format Conversion](#dataset-format-conversion)
- [COCO API Snippets](#coco-api-snippets)
- [Albumentations Config Patterns](#albumentations-config-patterns)
- [ONNX Export and Inspection](#onnx-export-and-inspection)
- [TensorRT (trtexec)](#tensorrt-trtexec)
- [OpenVINO](#openvino)
- [CoreML](#coreml)
- [Benchmarking Snippets](#benchmarking-snippets)
- [Segment Anything (SAM / SAM2)](#segment-anything-sam--sam2)
- [Tracking (ByteTrack / SORT)](#tracking-bytetrack--sort)
- [Experiment Tracking](#experiment-tracking)
- [Common Debugging Commands](#common-debugging-commands)

## Ultralytics YOLO CLI and Python API

Training:

```bash
# Basic training
yolo detect train data=data.yaml model=yolov8m.pt epochs=100 imgsz=640 batch=16

# Resume from checkpoint
yolo detect train resume model=runs/detect/train/weights/last.pt

# Multi-GPU (DDP)
yolo detect train data=data.yaml model=yolov8l.pt epochs=300 imgsz=640 device=0,1,2,3 batch=64

# Hyperparameter overrides commonly tuned in practice
yolo detect train data=data.yaml model=yolov8m.pt epochs=100 \
    lr0=0.01 lrf=0.01 momentum=0.937 weight_decay=0.0005 \
    warmup_epochs=3 box=7.5 cls=0.5 dfl=1.5 \
    mosaic=1.0 mixup=0.1 close_mosaic=10
```

Validation and inference:

```bash
yolo detect val model=best.pt data=data.yaml imgsz=640 conf=0.001 iou=0.6 save_json=True
yolo detect predict model=best.pt source=images/ conf=0.25 iou=0.45 save=True
```

Python API equivalent (preferred when scripting pipelines):

```python
from ultralytics import YOLO

model = YOLO("yolov8m.pt")
results = model.train(
    data="data.yaml", epochs=100, imgsz=640, batch=16,
    device=0, patience=30, cos_lr=True, amp=True,
)
metrics = model.val()  # metrics.box.map, metrics.box.map50
model.export(format="onnx", dynamic=True, simplify=True, opset=17)
```

`data.yaml` structure:

```yaml
path: data/final
train: images/train
val: images/val
test: images/test
names:
  0: car
  1: person
  2: bicycle
  3: dog
  4: cat
```

## Detectron2

```bash
python train_net.py --config-file configs/faster_rcnn_R_50_FPN_1x.yaml \
    --num-gpus 4 \
    SOLVER.IMS_PER_BATCH 16 SOLVER.BASE_LR 0.02 SOLVER.MAX_ITER 90000 \
    OUTPUT_DIR outputs/frcnn_r50

# Eval only
python train_net.py --config-file configs/faster_rcnn_R_50_FPN_1x.yaml \
    --eval-only MODEL.WEIGHTS outputs/frcnn_r50/model_final.pth
```

Register a custom COCO dataset:

```python
from detectron2.data.datasets import register_coco_instances
register_coco_instances(
    "my_train", {}, "data/coco/train.json", "data/coco/train_images/"
)
register_coco_instances(
    "my_val", {}, "data/coco/val.json", "data/coco/val_images/"
)

from detectron2.config import get_cfg
from detectron2 import model_zoo
cfg = get_cfg()
cfg.merge_from_file(model_zoo.get_config_file("COCO-Detection/faster_rcnn_R_50_FPN_3x.yaml"))
cfg.DATASETS.TRAIN = ("my_train",)
cfg.DATASETS.TEST = ("my_val",)
cfg.MODEL.WEIGHTS = model_zoo.get_checkpoint_url("COCO-Detection/faster_rcnn_R_50_FPN_3x.yaml")
cfg.MODEL.ROI_HEADS.NUM_CLASSES = 5
cfg.SOLVER.IMS_PER_BATCH = 8
cfg.SOLVER.BASE_LR = 0.00025
```

## MMDetection

```bash
python tools/train.py configs/rtmdet/rtmdet_l_8xb32-300e_coco.py \
    --work-dir work_dirs/rtmdet_l --gpus 4

python tools/test.py configs/rtmdet/rtmdet_l_8xb32-300e_coco.py \
    work_dirs/rtmdet_l/epoch_300.pth --eval bbox

# Export to ONNX via MMDeploy
python mmdeploy/tools/deploy.py \
    mmdeploy/configs/mmdet/detection/detection_onnxruntime_dynamic.py \
    configs/rtmdet/rtmdet_l_8xb32-300e_coco.py \
    work_dirs/rtmdet_l/epoch_300.pth \
    demo.jpg --work-dir mmdeploy_out --device cuda
```

## Dataset Format Conversion

VOC XML → COCO JSON (core logic, using `pylabel` or manual):

```python
from pylabel import importer
dataset = importer.ImportVOC(path="data/annotations", path_to_images="data/images")
dataset.export.ExportToCoco(output_path="data/coco/train.json")
```

YOLO TXT → COCO JSON:

```python
# YOLO txt: <class_id> <x_center> <y_center> <width> <height>  (all normalized 0-1)
def yolo_to_coco_bbox(x_c, y_c, w, h, img_w, img_h):
    x = (x_c - w / 2) * img_w
    y = (y_c - h / 2) * img_h
    return [x, y, w * img_w, h * img_h]  # COCO bbox = [x_min, y_min, width, height]
```

COCO JSON → YOLO TXT:

```python
def coco_to_yolo_bbox(bbox, img_w, img_h):
    x, y, w, h = bbox
    x_c = (x + w / 2) / img_w
    y_c = (y + h / 2) / img_h
    return x_c, y_c, w / img_w, h / img_h
```

## COCO API Snippets

```python
from pycocotools.coco import COCO
from pycocotools.cocoeval import COCOeval

coco_gt = COCO("data/coco/val.json")
coco_dt = coco_gt.loadRes("predictions.json")  # list of {image_id, category_id, bbox, score}

evaluator = COCOeval(coco_gt, coco_dt, iouType="bbox")
evaluator.evaluate()
evaluator.accumulate()
evaluator.summarize()
# Prints AP@[0.5:0.95], AP@0.5, AP@0.75, AP for small/medium/large objects, AR@1/10/100
```

Quick dataset sanity check:

```python
coco = COCO("data/coco/train.json")
print(f"Images: {len(coco.imgs)}, Annotations: {len(coco.anns)}, Categories: {len(coco.cats)}")
cat_counts = {c["name"]: 0 for c in coco.cats.values()}
for ann in coco.anns.values():
    cat_counts[coco.cats[ann["category_id"]]["name"]] += 1
print(cat_counts)  # check for severe class imbalance before training
```

## Albumentations Config Patterns

```python
import albumentations as A
from albumentations.pytorch import ToTensorV2

train_transform = A.Compose([
    A.RandomResizedCrop(640, 640, scale=(0.5, 1.0), p=1.0),
    A.HorizontalFlip(p=0.5),
    A.OneOf([
        A.RandomBrightnessContrast(brightness_limit=0.2, contrast_limit=0.2),
        A.HueSaturationValue(hue_shift_limit=20, sat_shift_limit=30),
    ], p=0.5),
    A.MotionBlur(blur_limit=3, p=0.1),
    A.Normalize(mean=(0.485, 0.456, 0.406), std=(0.229, 0.224, 0.225)),
    ToTensorV2(),
], bbox_params=A.BboxParams(format="coco", label_fields=["labels"], min_visibility=0.3))
```

## ONNX Export and Inspection

```bash
# Verify structural validity
python -c "import onnx; m = onnx.load('model.onnx'); onnx.checker.check_model(m); print('valid')"

# Simplify graph (folds constants, removes redundant nodes) — often 10-30% fewer nodes
pip install onnxsim
onnxsim model.onnx model_simplified.onnx

# Inspect input/output shapes and dynamic axes
python -c "
import onnx
m = onnx.load('model.onnx')
for inp in m.graph.input:
    print(inp.name, [d.dim_value or d.dim_param for d in inp.type.tensor_type.shape.dim])
"
```

ONNX Runtime inference with CUDA/TensorRT execution providers:

```python
import onnxruntime as ort

providers = [
    ("TensorrtExecutionProvider", {"trt_fp16_enable": True, "trt_engine_cache_enable": True}),
    "CUDAExecutionProvider",
    "CPUExecutionProvider",
]
sess = ort.InferenceSession("model.onnx", providers=providers)
outputs = sess.run(None, {"images": input_array})
```

## TensorRT (trtexec)

```bash
# FP16 engine build (standard for GPU deployment, ~2x speedup vs FP32, <0.5% mAP loss)
trtexec --onnx=model.onnx --saveEngine=model_fp16.engine --fp16 \
    --minShapes=images:1x3x640x640 --optShapes=images:8x3x640x640 --maxShapes=images:16x3x640x640

# INT8 with calibration cache (2-4x speedup vs FP32, needs calibration data)
trtexec --onnx=model.onnx --saveEngine=model_int8.engine --int8 \
    --calib=calibration.cache

# Benchmark an existing engine
trtexec --loadEngine=model_fp16.engine --batch=8 --iterations=200 --avgRuns=10
```

Python TensorRT runtime (pycuda):

```python
import tensorrt as trt
logger = trt.Logger(trt.Logger.WARNING)
runtime = trt.Runtime(logger)
with open("model_fp16.engine", "rb") as f:
    engine = runtime.deserialize_cuda_engine(f.read())
context = engine.create_execution_context()
```

## OpenVINO

```bash
# Convert ONNX -> OpenVINO IR
ovc model.onnx --output_model openvino_model --compress_to_fp16

# Post-training INT8 quantization via NNCF
python -c "
import nncf, openvino as ov
model = ov.Core().read_model('openvino_model.xml')
quantized = nncf.quantize(model, calibration_dataset)
ov.save_model(quantized, 'openvino_model_int8.xml')
"

# Benchmark
benchmark_app -m openvino_model.xml -d CPU -api async -niter 200
```

## CoreML

```python
import coremltools as ct

model = ct.convert(
    "model.onnx",
    inputs=[ct.ImageType(name="images", shape=(1, 3, 640, 640))],
    compute_units=ct.ComputeUnit.ALL,  # runs on Neural Engine when available
    minimum_deployment_target=ct.target.iOS16,
)
model.save("model.mlpackage")
```

## Benchmarking Snippets

```python
import time, torch

@torch.no_grad()
def benchmark(model, input_tensor, warmup=10, iters=100):
    model.eval()
    for _ in range(warmup):
        model(input_tensor)
    torch.cuda.synchronize()
    start = time.perf_counter()
    for _ in range(iters):
        model(input_tensor)
    torch.cuda.synchronize()
    elapsed = (time.perf_counter() - start) / iters
    return elapsed * 1000  # ms per inference
```

GPU utilization while benchmarking:

```bash
nvidia-smi dmon -s u -c 100          # utilization sampling
nvidia-smi --query-gpu=utilization.gpu,memory.used,power.draw --format=csv -l 1
```

## Segment Anything (SAM / SAM2)

```python
from segment_anything import sam_model_registry, SamPredictor

sam = sam_model_registry["vit_h"](checkpoint="sam_vit_h_4b8939.pth").to("cuda")
predictor = SamPredictor(sam)
predictor.set_image(image)
masks, scores, logits = predictor.predict(
    point_coords=[[500, 375]], point_labels=[1], multimask_output=True
)
```

```bash
# Automatic mask generation (no prompts) — expensive, ~1-2s/image on A100 for vit_h
python scripts/amg.py --checkpoint sam_vit_h_4b8939.pth --model-type vit_h --input images/ --output masks/
```

## Tracking (ByteTrack / SORT)

```python
from ultralytics import YOLO
model = YOLO("yolov8m.pt")
results = model.track(source="video.mp4", tracker="bytetrack.yaml", persist=True)
```

`bytetrack.yaml` key params worth tuning:

```yaml
track_high_thresh: 0.5   # detection conf threshold for first association pass
track_low_thresh: 0.1    # second-pass low-confidence recovery
new_track_thresh: 0.6    # confidence needed to spawn a new track ID
track_buffer: 30         # frames to keep a lost track alive before deletion
match_thresh: 0.8        # IoU threshold for association
```

## Experiment Tracking

```python
import wandb
wandb.init(project="detection-experiments", config={"arch": "yolov8m", "epochs": 100})

# Ultralytics has native W&B integration — just install and login
# pip install wandb && wandb login
# yolo detect train data=data.yaml model=yolov8m.pt epochs=100  # auto-logs to W&B if enabled
```

```bash
mlflow server --host 0.0.0.0 --port 5000 --backend-store-uri sqlite:///mlflow.db
export MLFLOW_TRACKING_URI=http://localhost:5000
```

## Common Debugging Commands

```bash
# Check CUDA / cuDNN / driver alignment (most common training crash cause)
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.backends.cudnn.version())"
nvidia-smi

# Find corrupted images before a training run kills itself mid-epoch
python -c "
from PIL import Image
import glob
for f in glob.glob('data/images/**/*.jpg', recursive=True):
    try:
        Image.open(f).verify()
    except Exception as e:
        print(f, e)
"

# Confirm anchor-free vs anchor-based head shape mismatch when loading a fine-tuned checkpoint
python -c "
import torch
ckpt = torch.load('best.pt', map_location='cpu')
print(ckpt['model'].yaml if hasattr(ckpt['model'], 'yaml') else 'n/a')
"
```
