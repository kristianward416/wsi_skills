# Depth System

The 6-layer depth model is the backbone of every scene in this skill. Every element gets a `data-depth` (0–5) and a matching `.depth-N` class. This file covers the math and the implementation.

## 1. The Canonical Depth Table

```
DEPTH 0 → Far background     | parallax: 0.10x | blur: 8px  | scale: 0.70
DEPTH 1 → Glow/atmosphere    | parallax: 0.25x | blur: 4px  | scale: 0.85
DEPTH 2 → Mid decorations    | parallax: 0.50x | blur: 0px  | scale: 1.00
DEPTH 3 → Main objects       | parallax: 0.80x | blur: 0px  | scale: 1.05
DEPTH 4 → UI / text          | parallax: 1.00x | blur: 0px  | scale: 1.00
DEPTH 5 → Foreground FX      | parallax: 1.20x | blur: 0px  | scale: 1.10
```

Read this as: depth-4 (text/UI) moves 1:1 with scroll — the "true" scroll speed. Everything with a multiplier below 1.0 appears to lag behind (recedes into the background); anything above 1.0 appears to rush ahead (pops into the foreground). This is the entire illusion of depth — no WebGL or actual 3D geometry required.

## 2. The Parallax Offset Formula

For a layer with scroll-speed multiplier `m`, and the page's scroll position `scrollY`:

```
layerOffset = (scrollY - sceneScrollStart) * (m - 1)
```

Subtracting 1 from `m` is the key move: it expresses the layer's motion *relative to* normal document flow (which is already moving at 1x as the user scrolls). A depth-0 layer with `m = 0.10` gets `offset = scrollDelta * -0.90` — i.e. it moves backward relative to the viewport, producing the illusion of recession.

```js
const DEPTH_SPEEDS = { 0: 0.10, 1: 0.25, 2: 0.50, 3: 0.80, 4: 1.00, 5: 1.20 };

function updateParallax(scene) {
  const rect = scene.getBoundingClientRect();
  const sceneProgress = -rect.top; // px scrolled into this scene

  scene.querySelectorAll('[data-depth]').forEach((layer) => {
    const depth = Number(layer.dataset.depth);
    const speed = DEPTH_SPEEDS[depth];
    const offset = sceneProgress * (speed - 1);
    layer.style.transform = `translate3d(0, ${offset}px, 0)`;
  });
}
```

Always use `translate3d` (not `translate`) and never animate `top`/`margin-top` — this keeps the transform on the compositor thread (see `references/performance.md`).

## 3. translateZ / True 3D Depth Variant

For scenes with `perspective` set on the container (true CSS 3D rather than simulated 2D parallax), depth is expressed as literal Z-distance instead of a speed multiplier:

```css
.scene-3d {
  perspective: 1200px;
  perspective-origin: 50% 50%;
}

.depth-0 { transform: translateZ(-600px) scale(1.5); }  /* pushed back, upscaled to compensate for perspective shrink */
.depth-1 { transform: translateZ(-350px) scale(1.29); }
.depth-2 { transform: translateZ(-150px) scale(1.11); }
.depth-3 { transform: translateZ(0px)    scale(1.00); }  /* focal plane */
.depth-4 { transform: translateZ(80px)   scale(0.94); }
.depth-5 { transform: translateZ(200px)  scale(0.85); }
```

### Compensating scale for translateZ

When you push a layer back with `translateZ(-D)`, CSS perspective projection shrinks it visually. To keep its on-screen size matching its 2D-simulated counterpart, compensate with:

```
compensatedScale = perspective / (perspective + D)
```

For `perspective: 1200px` and `D = 600` (translateZ(-600px)):
```
compensatedScale = 1200 / (1200 + 600) = 1200 / 1800 = 0.667  →  scale up by 1/0.667 = 1.5
```
This is why depth-0 above uses `scale(1.5)` — it exactly cancels the perspective shrink so the layer reads at its intended size while still being physically "further away" for parallax/lighting purposes.

Prefer the 2D `translate3d` + speed-multiplier system for most builds — it's cheaper, more predictable across viewport sizes, and doesn't require managing compensating scales. Reserve `perspective`/`translateZ` for scenes that explicitly want a mouse-tilt or true-3D-rotation effect.

## 4. Blur and Scale as Depth Cues

Blur and scale reinforce parallax speed — don't skip them even when the speed multiplier alone "looks fine":

```css
.depth-0 { filter: blur(8px); transform: scale(0.70); }
.depth-1 { filter: blur(4px); transform: scale(0.85); }
.depth-2 { filter: blur(0px); transform: scale(1.00); }
.depth-3 { filter: blur(0px); transform: scale(1.05); } /* main object, slight upscale for presence */
.depth-4 { filter: blur(0px); transform: scale(1.00); }
.depth-5 { filter: blur(0px); transform: scale(1.10); } /* foreground FX, largest — closest to viewer */
```

Note the combined transform must include both parallax offset and depth scale — apply them together, not as separate properties that fight each other:

```js
layer.style.transform =
  `translate3d(0, ${offset}px, 0) scale(${DEPTH_SCALE[depth]})`;
```

## 5. Z-Index / Stacking Context Rules

Depth order must be enforced by real DOM stacking, not just visual convention — otherwise a depth-5 foreground particle can end up rendered behind depth-3 content if `z-index` is wrong.

```css
.layer { position: absolute; inset: 0; }
.depth-0 { z-index: 0; }
.depth-1 { z-index: 1; }
.depth-2 { z-index: 2; }
.depth-3 { z-index: 3; }
.depth-4 { z-index: 4; }
.depth-5 { z-index: 5; }

.scene {
  position: relative;   /* establishes stacking context for the scene */
  isolation: isolate;   /* prevents children from leaking z-index into sibling scenes */
  overflow: clip;       /* contains any layer that scales/translates beyond scene bounds */
  height: var(--scene-height, 100vh);
}
```

`isolation: isolate` is important on every `.scene` — without it, a `z-index: 5` particle in scene A can outrank a `z-index: 4` text layer in scene B if scene B doesn't also create its own stacking context, because both are competing in the same global stacking order.

### Sticky/pinned scenes need an extra stacking layer

When a scene is pinned (`position: sticky` or GSAP ScrollTrigger `pin: true`), it stays on screen while later scenes scroll over it. Give the *next* scene an explicit higher z-index and an opaque background so it visually overtakes the pinned one:

```css
.scene[data-pinned="true"] { z-index: 1; }
.scene[data-pinned="true"] + .scene { z-index: 2; position: relative; background: var(--bg-color); }
```

## 6. Depth Assignment Defaults (Non-Negotiable)

- Product/hero images → **depth-3** by default, always
- Section background images/video → **depth-0**, always with slight blur
- Text/UI/CTAs → **depth-4** (true scroll speed — text should never look like it's swimming)
- Decorative glows/blobs → **depth-1**
- Small accent shapes hugging the hero → **depth-2**
- Sparkles/dust/grain overlays → **depth-5**

Every scene needs a minimum of 3 depth layers (the skill's Rule #1). A minimal valid scene: depth-0 (background), depth-3 (hero image), depth-4 (headline).

## 7. Full Implementation Reference

```css
.layer[data-depth] { will-change: transform; }
```

```js
class DepthScene {
  constructor(sceneEl) {
    this.scene = sceneEl;
    this.layers = [...sceneEl.querySelectorAll('[data-depth]')];
    this.speeds = { 0: 0.10, 1: 0.25, 2: 0.50, 3: 0.80, 4: 1.00, 5: 1.20 };
    this.ticking = false;
    window.addEventListener('scroll', () => this.requestTick(), { passive: true });
  }
  requestTick() {
    if (this.ticking) return;
    this.ticking = true;
    requestAnimationFrame(() => { this.update(); this.ticking = false; });
  }
  update() {
    const rect = this.scene.getBoundingClientRect();
    if (rect.bottom < 0 || rect.top > window.innerHeight) return; // skip off-screen scenes
    const progress = -rect.top;
    this.layers.forEach((layer) => {
      const speed = this.speeds[Number(layer.dataset.depth)];
      const offset = progress * (speed - 1);
      layer.style.transform = `translate3d(0, ${offset.toFixed(2)}px, 0)`;
    });
  }
}
document.querySelectorAll('.scene').forEach((s) => new DepthScene(s));
```

This throttles work to one calculation per animation frame and skips scenes that are entirely out of view — see `references/performance.md` for the full scroll-jank-avoidance rationale behind this pattern.
