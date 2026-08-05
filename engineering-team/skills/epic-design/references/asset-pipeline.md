# Asset Pipeline

Governs how images and video get inspected, judged, prepared, and served for scroll-heavy 2.5D sites. Read this before generating any HTML/CSS.

## 1. Inspection Workflow

Run `scripts/inspect-assets.py <file>` on every uploaded image before writing code. The script reports:

```
image.png
  format: PNG
  mode: RGBA
  size: 2400x2400
  alpha_channel: real (rgba varies)
  background: transparent (98.2% edge alpha < 10)
  → ✅ Clean cutout — use directly
```

For JPEGs, the script always reports `alpha_channel: none (JPEG cannot encode alpha)`. Never attempt transparency workflows on a JPEG source — re-export from PNG/TIFF or run background removal first.

### Background classification

| Script verdict | Meaning | Action |
|---|---|---|
| ✅ Clean cutout | Real alpha, edges transparent | Use directly, no processing |
| ⚠️ Solid dark background | Uniform dark fill behind subject | Judge (see below) |
| ⚠️ Solid light/white background | Uniform light fill behind subject | Judge (see below) |
| ⚠️ Complex/scene background | Non-uniform, photographic context | Judge (see below) |

## 2. Judging Whether to Remove a Background

Do not reflexively strip every background. Ask: **what role does this asset play in the scroll story?**

**Remove background when the asset is:**
- An isolated product meant to float free at depth-2/3 (bottle, shoe, gadget)
- A character/figure that needs to sit over other layers
- A logo/icon reused on multiple background colors
- Anything assigned a floating, scattering, or orbiting animation

**Keep background when the asset is:**
- A UI/app/website screenshot (the frame IS the content)
- A full-bleed photograph used as a depth-0 section background
- A complete illustration/poster meant to be read as a whole
- A mockup or device frame ("image inside a card")
- Any depth-0 layer image — depth-0 always keeps its background, just gets blurred and darkened

If genuinely ambiguous, default to **keep** and ask the user — silent background removal is prohibited by the skill's non-negotiable rules.

## 3. User Notification Format

Report every asset in this exact structure before generating any code:

```
📸 Asset Audit

1. hero-bottle.png (2400×2400, PNG, RGBA)
   ✅ Clean cutout — real transparency detected
   → Assigned: depth-3 (hero), resize to 1200px display width

2. background-texture.jpg (3840×2160, JPEG)
   ⚠️ No alpha possible (JPEG) — this is fine, it's a depth-0 background
   → Assigned: depth-0, will apply blur(8px) + scale(0.9) via CSS

3. leaf-accent.png (1800×1800, PNG, RGBA)
   ⚠️ Solid white background detected
   → This looks like a floating accent (depth-2). Recommend removing
     the white background so it doesn't show as a box. Confirm before
     I proceed, or tell me to keep it as-is.
```

Never proceed to build using an asset with an unresolved ⚠️ flag without either user confirmation or explicit reasoning stated to the user.

## 4. CSS Knockout (Lightweight Background Removal)

For solid, near-uniform backgrounds, a full background-removal tool is often unnecessary — `mix-blend-mode` or `filter` tricks can knock out flat white/black without external processing:

```css
/* Knock out a solid WHITE background without external tools */
.knockout-white {
  mix-blend-mode: multiply;   /* white areas disappear over light sections */
  background: transparent;
}

/* Knock out a solid BLACK background */
.knockout-black {
  mix-blend-mode: screen;     /* black areas disappear over dark sections */
}
```

These only work cleanly on solid, non-gradient studio backdrops, and only when the destination section background is compatible (multiply needs a light section, screen needs a dark section). For anything else, treat it as an asset-prep task and ask the user for a proper cutout — don't fake it with filters that will show halos.

## 5. Resize Targets by Depth Role

| Role | Depth | Target display width | Source resolution needed |
|---|---|---|---|
| Hero product | 3 | 50–80vw | 2x display width, min 1600px |
| Companion accent | 2 | 15–25% of hero | 2x display width, ~600–900px |
| Particle/sparkle | 5 | 1–5vw | 200–400px, sprite-sheet if repeated |
| Section background | 0 | 100vw (full-bleed) | 1920px min, 2560px for hero sections |
| Glow/atmosphere blob | 1 | 30–60vw, heavily blurred | Can be tiny (200px) — blur hides pixelation |

Rule of thumb: **never ship a source image larger than 2x its maximum rendered size.** Depth-0/1 layers get blurred anyway, so their source resolution can be dramatically lower than depth-3 hero assets.

## 6. Responsive Images

Every non-decorative image gets a `srcset` with real breakpoints, not arbitrary guesses:

```html
<img
  class="product-hero depth-3"
  src="bottle-800.png"
  srcset="
    bottle-480.png 480w,
    bottle-800.png 800w,
    bottle-1200.png 1200w,
    bottle-1600.png 1600w
  "
  sizes="(max-width: 600px) 90vw, (max-width: 1200px) 60vw, 50vw"
  alt="Product bottle floating over a dark studio background"
  loading="lazy"
  decoding="async"
  fetchpriority="low"
/>
```

For the hero image visible in the first viewport, invert the loading strategy — no `lazy`, and set `fetchpriority="high"` so it doesn't compete with below-fold assets for bandwidth:

```html
<img class="product-hero depth-3" src="bottle-1200.png" fetchpriority="high" alt="…" />
```

### Format priority

1. **AVIF** — best compression, use for photographic depth-0/1 backgrounds
2. **WebP** — near-universal support, good middle ground
3. **PNG** — only for assets that require crisp transparency edges (product cutouts, logos)

```html
<picture>
  <source srcset="bg-2560.avif" type="image/avif">
  <source srcset="bg-2560.webp" type="image/webp">
  <img src="bg-2560.jpg" alt="" class="layer depth-0" aria-hidden="true">
</picture>
```

## 7. Lazy Loading Strategy for Scroll-Heavy Pages

Scroll storytelling pages are long — never eagerly load everything.

```js
// Native lazy-load handles most images; use IntersectionObserver
// for anything that also needs a "wake up" moment (e.g. start a float loop
// only once visible, to avoid animating 60 off-screen elements at once).
const lazyLayers = document.querySelectorAll('[data-lazy-activate]');

const activationObserver = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    if (entry.isIntersecting) {
      entry.target.classList.add('is-active');   // starts CSS animation
      activationObserver.unobserve(entry.target); // one-shot activation
    }
  });
}, { rootMargin: '200px 0px' }); // pre-activate 200px before entering viewport

lazyLayers.forEach((el) => activationObserver.observe(el));
```

Pair this with `content-visibility: auto` (see `references/performance.md`) so off-screen sections skip layout/paint entirely, not just image download.

## 8. Video vs Lottie vs CSS-only — Choosing the Right Medium

| Need | Best medium | Why |
|---|---|---|
| Looping ambient background motion (smoke, particles, gradient flow) | **CSS-only** | Zero network cost after page load, infinitely scalable, no decode cost |
| Complex vector micro-interaction (icon morph, logo build) | **Lottie (JSON)** | Small file size, crisp at any DPI, scrubbable to scroll position |
| Real footage (product demo, lifestyle b-roll) | **Video (MP4/WebM)** | Only option for photographic motion |
| Simple shape/color transition | **CSS-only** | Avoid Lottie/video overhead for anything expressible in `transform`/`filter` |
| Scroll-scrubbed animation (plays exactly in sync with scroll position) | **Lottie** (via `lottie-web` + GSAP) or CSS-only if simple enough | Video scrubbing (`video.currentTime = …`) is CPU-expensive and janky on scroll; Lottie scrubs cheaply |

### Video implementation (only when real footage is required)

```html
<video
  class="layer depth-0"
  autoplay muted loop playsinline
  preload="none"
  poster="video-poster.jpg"
  aria-hidden="true"
>
  <source src="bg-loop-1080.webm" type="video/webm">
  <source src="bg-loop-1080.mp4" type="video/mp4">
</video>
```

Rules for background video:
- Always `muted` + `playsinline` (required for autoplay on iOS)
- Always provide a `poster` — shown while decoding, and to `prefers-reduced-motion` users if you pause the tag
- Keep source under ~8s duration and loop it — long files cost bandwidth for no visual gain
- Cap resolution at 1080p even on 4K displays; blurred/backgrounded video doesn't benefit from more pixels
- Pause when out of viewport:

```js
const bgVideos = document.querySelectorAll('video.layer');
const videoObserver = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    const video = entry.target;
    if (entry.isIntersecting) video.play().catch(() => {});
    else video.pause();
  });
}, { threshold: 0.1 });
bgVideos.forEach((v) => videoObserver.observe(v));
```

### Lottie implementation (scroll-scrubbed)

```js
import lottie from 'lottie-web';

const anim = lottie.loadAnimation({
  container: document.querySelector('.lottie-target'),
  renderer: 'svg',
  loop: false,
  autoplay: false,
  path: 'logo-build.json',
});

// Scrub Lottie frames directly with scroll progress (GSAP ScrollTrigger)
gsap.to(anim, {
  currentFrame: anim.totalFrames - 1,
  ease: 'none',
  scrollTrigger: {
    trigger: '.lottie-section',
    start: 'top top',
    end: 'bottom top',
    scrub: true,
  },
});
```

### CSS-only ambient motion (preferred default)

```css
.glow-blob {
  background: radial-gradient(circle, rgba(120,90,255,0.4), transparent 70%);
  filter: blur(40px);
  animation: drift 14s ease-in-out infinite alternate;
}
@keyframes drift {
  0%   { transform: translate(0, 0) scale(1); }
  100% { transform: translate(4%, -6%) scale(1.08); }
}
```

Default to CSS-only for any effect it can express. Reserve video/Lottie budget for the one or two moments per page that actually need photographic or vector complexity.
