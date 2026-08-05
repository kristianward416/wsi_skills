# Computer Vision Architectures: CNNs, ViTs, and Hybrids

A field guide to the classification/backbone architectures referenced throughout this skill's detection and segmentation workflows. This covers the backbones that feed into YOLO, Faster R-CNN, DETR, SAM, and DeepLabV3+/SegFormer heads described in SKILL.md's Architecture Selection Guide.

## Table of Contents
- [Why Backbone Choice Matters](#why-backbone-choice-matters)
- [CNN Architectures](#cnn-architectures)
- [Vision Transformer Architectures](#vision-transformer-architectures)
- [Hybrid CNN-Transformer Architectures](#hybrid-cnn-transformer-architectures)
- [Self-Supervised Backbones](#self-supervised-backbones)
- [Backbone Selection Guide by Constraint](#backbone-selection-guide-by-constraint)
- [Backbone Swaps in Practice](#backbone-swaps-in-practice)

## Why Backbone Choice Matters

Every detection/segmentation head in this skill (YOLO's head, Faster R-CNN's RPN + ROI heads, DETR's transformer decoder, DeepLabV3+'s ASPP module) sits on top of a feature-extracting backbone. The backbone determines the accuracy ceiling, the FLOPs budget, and how well small objects and fine textures survive downsampling. Swapping ResNet-50 for ConvNeXt-T in a Faster R-CNN, for instance, is a common lever for a quick 2-4 mAP gain with roughly comparable latency — before touching the detection head at all.

## CNN Architectures

### ResNet Family (2015, still the default baseline)

| Variant | Params | FLOPs (224²) | ImageNet Top-1 | Notes |
|---|---|---|---|---|
| ResNet-18 | 11.7M | 1.8G | 69.8% | Edge/mobile backbone, rarely used for detection alone |
| ResNet-50 | 25.6M | 4.1G | 76.1% | The default Faster R-CNN / Mask R-CNN backbone (`R_50_FPN`) |
| ResNet-101 | 44.5M | 7.8G | 77.4% | +1-2 mAP over R-50 in detection, ~40% slower |
| ResNeXt-101-32x8d | 88.8M | 16.5G | 79.3% | Grouped convolutions, better accuracy/FLOP ratio than plain ResNet |

Residual connections solved vanishing gradients for deep nets; the architecture is still ubiquitous because it's well-understood, has mature FPN integration, and every framework (Detectron2, MMDetection, torchvision) ships pretrained weights and reference configs for it. Downside: FLOP-for-FLOP, it's been beaten by EfficientNet and ConvNeXt on pure classification, and its inductive bias (local conv + late global pooling) makes it comparatively weak on long-range context versus attention-based backbones.

### EfficientNet (2019) — compound scaling

| Variant | Params | FLOPs | Top-1 | Notes |
|---|---|---|---|---|
| EfficientNet-B0 | 5.3M | 0.39G | 77.1% | Best accuracy/FLOP at the small end |
| EfficientNet-B4 | 19M | 4.2G | 82.9% | Common cloud-serving sweet spot |
| EfficientNet-B7 | 66M | 37G | 84.3% | Diminishing returns, slow to train |

Compound scaling jointly scales depth/width/resolution by a fixed coefficient rather than scaling one dimension arbitrarily — this is why EfficientNet dominated accuracy-per-FLOP leaderboards for years. Caveat that matters in production: depthwise separable convolutions (MBConv blocks) are FLOP-efficient but memory-bandwidth-bound, so real GPU latency doesn't scale down as cleanly as the FLOP counts suggest — EfficientNet-B4 is often *slower* on a GPU than a ResNet-50 with 4x the FLOPs, because of poor kernel utilization on grouped/depthwise ops. It shines on CPU and mobile NPUs where FLOPs correlate better with latency.

### ConvNeXt (2022) — CNN redesigned with ViT lessons

| Variant | Params | FLOPs | Top-1 | Notes |
|---|---|---|---|---|
| ConvNeXt-T | 28.6M | 4.5G | 82.1% | Drop-in ResNet-50 replacement, +6 pts Top-1 at similar FLOPs |
| ConvNeXt-S | 50M | 8.7G | 83.1% | |
| ConvNeXt-B | 89M | 15.4G | 83.8% | |
| ConvNeXt-L | 198M | 34.4G | 84.3% | |

Takes ResNet and applies ViT-inspired design choices without any attention: larger 7x7 depthwise kernels, LayerNorm instead of BatchNorm, GELU instead of ReLU, inverted bottleneck (wide-narrow-wide), fewer activation functions per block. The result matches or beats Swin Transformer at equivalent FLOPs while keeping pure-conv inference simplicity (easier TensorRT/ONNX export, no attention kernel quirks). This is the backbone to reach for when a team wants ViT-level accuracy but needs to stay in familiar conv-export tooling for edge deployment.

## Vision Transformer Architectures

### ViT (2020) — the original

| Variant | Params | FLOPs (224²) | ImageNet Top-1 (pretrained on JFT-300M) |
|---|---|---|---|
| ViT-B/16 | 86M | 17.6G | 84.2% (77.9% if trained on ImageNet-1k only) |
| ViT-L/16 | 307M | 61.6G | 85.3% |

Key limitation to internalize: ViT trained from scratch on ImageNet-1k alone (1.3M images) underperforms ResNet — the plain patch-embedding + global self-attention design has weaker inductive bias than convolution and needs either massive pretraining data (JFT-300M, 300M+ images) or distillation to be competitive. This is why DeiT and hybrid architectures exist. ViT's O(n²) attention over patches also means compute grows quadratically with resolution — a 224² image at patch size 16 gives 196 tokens; doubling resolution to 448² gives 784 tokens and ~16x the attention FLOPs, not 4x.

### DeiT (2021) — data-efficient ViT via distillation

DeiT-B matches ViT-B/16's 84.2% Top-1 while training only on ImageNet-1k (no JFT), by adding a distillation token trained against a strong CNN teacher (typically RegNet). This made ViT-style backbones practical for teams without access to internet-scale pretraining corpora, and is the reason DeiT shows up as the classification backbone of choice in resource-constrained ViT projects.

### Swin Transformer (2021) — hierarchical, shifted windows

| Variant | Params | FLOPs | Top-1 | COCO box mAP (as detection backbone) |
|---|---|---|---|---|
| Swin-T | 29M | 4.5G | 81.3% | 50.5 (Cascade Mask R-CNN) |
| Swin-S | 50M | 8.7G | 83.0% | 51.8 |
| Swin-B | 88M | 15.4G | 83.5% | 51.9 |
| Swin-L | 197M | 34.5G | 86.3% | 53.9 |

Swin computes self-attention within local windows (default 7x7 patches) rather than globally, then shifts the window grid between layers so information flows across window boundaries over depth. This gives linear (not quadratic) complexity in image size and produces a hierarchical multi-scale feature pyramid — which is exactly what FPN-based detectors (Faster R-CNN, Mask R-CNN) expect, unlike plain ViT's single-scale output. This is why Swin, not plain ViT, became the standard transformer backbone for dense prediction tasks (detection, segmentation) — DINO and several Detectron2/MMDetection SOTA configs use Swin-L backbones.

### PVT / Twins / Focal Transformer

Second-generation hierarchical ViTs addressing similar multi-scale needs as Swin with different attention-reduction tricks (spatial reduction attention in PVT, local-global split in Twins). Rarely the first choice today — Swin and ConvNeXt cover most practical needs — but PVTv2 backbones still appear in some segmentation literature (SegFormer's encoder is a PVT-style hierarchical transformer).

## Hybrid CNN-Transformer Architectures

### RT-DETR's hybrid encoder

RT-DETR (used in SKILL.md's real-time transformer detection path) uses a CNN backbone (ResNet or CSPResNet) for multi-scale feature extraction, then a lightweight transformer encoder operating only on the lowest-resolution feature map (to control the O(n²) cost), followed by a CNN-based fusion (CCFM) to redistribute global context back into the higher-resolution feature maps before the decoder. This hybrid split — CNN for local/multiscale, transformer for global context on a cheap subset — is why RT-DETR-L hits 53.0 mAP at 5.3ms, competitive with YOLOv8-scale latency while removing NMS from the pipeline entirely.

### CoAtNet / MobileViT

Interleave conv blocks (early, cheap, strong local inductive bias) with transformer blocks (later, global context on already-downsampled features). MobileViT specifically targets mobile deployment: MobileViT-S gets 78.4% ImageNet Top-1 at ~6M params and ~2 GFLOPs, beating MobileNetV3 at similar budget by giving up some pure speed for the global-context boost transformers provide.

## Self-Supervised Backbones

### DINOv2 (2023)

Self-supervised ViT (ViT-g/14 giant variant at 1.1B params, distilled down to ViT-S/B/L) trained on 142M curated images without labels. Frozen DINOv2 features, with just a linear probe, hit 86.3% ImageNet Top-1 (ViT-g) and produce dense features strong enough for k-NN segmentation and depth estimation without any task-specific fine-tuning. Practical relevance to this skill: DINOv2 backbones are increasingly the default choice when labeled data is scarce — freeze the backbone, train a small detection/segmentation head on top, and get strong results with a few hundred to a few thousand labeled images instead of the 10K+ typically needed for supervised ViT training mentioned in SKILL.md's CNN vs ViT trade-off table.

### CLIP ViT

Contrastively trained on 400M image-text pairs. Not typically used directly as a detection backbone, but its image encoder underlies open-vocabulary detection (Grounding DINO, YOLO-World) — relevant when a detection task's classes aren't fixed at training time.

## Backbone Selection Guide by Constraint

| Constraint | Recommended Backbone | Rationale |
|---|---|---|
| <1K labeled images, need transfer learning | ResNet-50 or ConvNeXt-T pretrained | Strong ImageNet features transfer well with few images |
| Edge/mobile, <5M params | MobileNetV3, EfficientNet-B0, MobileViT-S | FLOP-efficient, quantization-friendly |
| Need multi-scale features for FPN detection head | ResNet+FPN, Swin, ConvNeXt | Native hierarchical stages match FPN's expected pyramid |
| Max accuracy, GPU cloud serving, no latency pressure | Swin-L, ConvNeXt-XL, ViT-L/DINOv2-L | Accept 2-3x latency for +3-5 mAP |
| Real-time GPU detection (YOLO/RT-DETR) | CSPDarknet (YOLO's own), ResNet/CSPResNet (RT-DETR) | Purpose-built for speed, avoid attention overhead in backbone |
| Very limited labels, need strong features "for free" | DINOv2 frozen backbone + linear/light head | Self-supervised pretraining substitutes for missing labeled data |
| CPU-only inference | EfficientNet, MobileNetV3 | FLOPs correlate with CPU latency better than on GPU |
| Open-vocabulary / classes unknown at training time | CLIP ViT-based (Grounding DINO, YOLO-World) | Text-conditioned detection, no fixed class list |

## Backbone Swaps in Practice

Detectron2 config swap (ResNet-50 → stronger backbone requires a matching FPN-compatible config; most teams instead swap within timm-supported backbones via a custom `build_backbone`):

```python
# torchvision / timm backbone swap for a custom detection head
import timm
backbone = timm.create_model("convnext_tiny", pretrained=True, features_only=True,
                              out_indices=(1, 2, 3))  # multi-scale feature maps for FPN
```

Rule of thumb when deciding whether a backbone swap is worth the retraining cost: if the current backbone is a plain ResNet-50 and latency budget allows even 10-20% headroom, ConvNeXt-T is close to a free 2-4 mAP win. If labeled data is under ~2K images, a DINOv2-frozen backbone is worth prototyping before spending budget on more annotation.
