# Performance

Non-negotiable GPU and frame-budget rules for every 2.5D build. If a technique from any other reference file can't be implemented within this budget, simplify it — don't ship jank.

## 1. The 60fps Budget, in Numbers

A 60Hz display needs a new frame every **16.7ms** (1000ms / 60). That budget is shared across the whole pipeline:

```
16.7ms total budget
├─ JS (scroll handler, animation logic)     ≤ 4-5ms
├─ Style recalculation                       ≤ 1-2ms
├─ Layout (reflow)                           0ms   ← must not happen at all during scroll
├─ Paint                                     ≤ 3-4ms  ← ideally 0ms (compositor-only)
├─ Composite                                 ≤ 2-3ms
└─ Browser/GPU overhead & buffer             ~3ms
```

If a scroll-driven site targets 120Hz displays (increasingly common — most modern phones), the budget halves to **8.3ms**. Design your JS-per-frame cost assuming the tighter budget where possible; it degrades gracefully to 60Hz but not the reverse.

Dropped-frame math: a single frame that takes 33ms (missing one 16.7ms slot) reads as a visible stutter. On a 3-second scroll-triggered animation at 60fps (180 frames), staying under budget on 175/180 frames is a smooth 97% — but perceptible jank clusters (5 consecutive dropped frames) read far worse than isolated 1-frame drops. Diagnose for *consecutive* drops, not just average frame time.

## 2. Compositor-Only Properties

Only four CSS properties can be animated without triggering layout or paint, because the browser can hand them entirely to the GPU compositor thread:

```
transform   (translate, scale, rotate, skew — never top/left/margin)
opacity
filter
clip-path   (compositor-accelerated in modern Chromium/Firefox/Safari — verify with the "Layers" DevTools panel per browser)
```

**Never animate:** `width`, `height`, `top`, `left`, `margin`, `padding`, `border-width` — these trigger layout (reflow) on every frame, which is the single biggest cause of scroll jank. If you need an element to "grow," use `transform: scale()`; if you need it to "move," use `transform: translate()`; if you need it to "reveal," use `clip-path` or `opacity`, never `height: 0 → auto`.

```css
/* WRONG — triggers layout every frame */
.card { transition: width 0.3s, height 0.3s; }
.card:hover { width: 320px; height: 400px; }

/* RIGHT — compositor-only */
.card { transition: transform 0.3s; transform-origin: center; }
.card:hover { transform: scale(1.08); }
```

### Forcing GPU layer promotion

```css
.gpu-layer {
  transform: translateZ(0); /* or translate3d(0,0,0) — forces its own compositor layer */
  backface-visibility: hidden; /* prevents flicker on some Safari/webkit versions during 3D transforms */
}
```

## 3. `will-change` — Correct Use and Common Pitfalls

`will-change` tells the browser to pre-promote an element to its own GPU layer *before* the animation starts, avoiding a stutter on the first frame.

```css
.depth-3.is-animating { will-change: transform; }
```

**Pitfalls:**

1. **Applying it globally / permanently.** Every layer promoted to the GPU consumes VRAM and compositing overhead. A page with 80 elements all carrying `will-change: transform` at all times can be *slower* than no `will-change` at all, because the browser is maintaining 80 separate GPU layers continuously.

   ```js
   // RIGHT — add right before animating, remove right after
   el.style.willChange = 'transform';
   gsap.to(el, {
     x: 200,
     onComplete: () => { el.style.willChange = 'auto'; }
   });
   ```

2. **Using it on too many elements at once.** Cap simultaneously-promoted layers to what's visible on screen — combine with `IntersectionObserver` so only in-viewport elements carry `will-change`.

3. **Setting it on the `<body>` or a huge wrapping container.** This promotes the *entire subtree* to a single giant composited layer, often forcing a full-page repaint on any change inside it — the opposite of the intended effect. Scope `will-change` to the smallest element that actually animates.

4. **Forgetting mobile memory limits.** Mobile GPUs have far less VRAM headroom than desktop. More than ~15–20 simultaneously composited layers on a mid-range phone is a common cause of the tab crashing or being killed by the OS. Combine with the skill's "reduce effects on `pointer: coarse`" rule.

## 4. The 80-Element Ceiling

The skill's proactive trigger — "more than 80 animated elements → flag performance concern" — comes directly from compositor layer budgets. As a working rule:

- **≤30 simultaneously animating elements**: safe on virtually all devices
- **30–80**: safe on desktop/modern mobile, monitor performance on low-end Android
- **80+**: high risk of dropped frames even on high-end devices; batch/virtualize (only animate elements within/near viewport via `IntersectionObserver`), or consolidate multiple small elements into a single animated container (animate the parent's `transform`, not 40 children's individual transforms).

## 5. `content-visibility: auto` for Off-Screen Sections

```css
.scene {
  content-visibility: auto;
  contain-intrinsic-size: 0 100vh; /* prevents layout jump when content is skipped */
}
```

This tells the browser to skip layout, style, and paint work entirely for sections not near the viewport — critical on a scroll-storytelling page that might have 8-10 full-height scenes. Always pair with `contain-intrinsic-size` matching the scene's expected height, or the page's scrollbar/scroll-position math will jump as sections pop in/out of visibility calculation.

## 6. Throttling Scroll Work with `requestAnimationFrame`

Never run animation logic directly inside a `scroll` event handler — scroll fires far more often than the display can paint.

```js
let ticking = false;
window.addEventListener('scroll', () => {
  if (!ticking) {
    requestAnimationFrame(() => {
      updateParallax();
      ticking = false;
    });
    ticking = true;
  }
}, { passive: true }); // passive:true lets the browser start scrolling immediately, not wait for JS
```

If using GSAP ScrollTrigger, this batching is handled internally — but the same principle applies to any hand-rolled scroll listener.

## 7. IntersectionObserver Over Scroll-Position Math

Don't compute "is this element visible" by comparing `scrollY` to element offsets on every scroll tick — that forces a layout read (`getBoundingClientRect()`) which is comparatively expensive at 60x/sec. Use `IntersectionObserver`, which runs off the main thread's hot path:

```js
const revealObserver = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    entry.target.classList.toggle('is-visible', entry.isIntersecting);
  });
}, { threshold: 0.2, rootMargin: '-10% 0px -10% 0px' });

document.querySelectorAll('[data-reveal]').forEach((el) => revealObserver.observe(el));
```

Reserve `getBoundingClientRect()`-based per-frame math for the parallax/depth system itself (which genuinely needs continuous position, not a boolean), and use `IntersectionObserver` for every binary "has this entered/exited" decision (reveal triggers, animation start/stop, lazy activation).

## 8. Diagnosing Scroll Jank

Checklist, in order of likelihood:

1. **Chrome DevTools → Performance panel → record while scrolling.** Look for red "Layout Shift" / purple "Rendering" bars repeating every frame — this means something is animating a layout property.
2. **DevTools → Rendering tab → "Paint flashing."** Green flashes on elements that shouldn't be repainting on scroll indicates a non-compositor property is in play, or a layer isn't properly promoted.
3. **DevTools → Layers panel.** Count composited layers. If it's in the hundreds, `will-change` is over-applied (see §3).
4. **Check for synchronous `getBoundingClientRect()`/`getComputedStyle()` calls inside scroll or rAF callbacks** that also *write* styles in the same frame — this causes forced synchronous layout ("layout thrashing"). Batch all reads before all writes:

   ```js
   // WRONG — read/write interleaved per element forces layout thrash
   elements.forEach((el) => {
     const top = el.getBoundingClientRect().top; // read
     el.style.transform = `translateY(${top * 0.5}px)`; // write
   });

   // RIGHT — batch reads, then batch writes
   const tops = elements.map((el) => el.getBoundingClientRect().top);
   elements.forEach((el, i) => { el.style.transform = `translateY(${tops[i] * 0.5}px)`; });
   ```

5. **Check image decode cost.** A large, un-resized hero PNG decoding on every repaint (common when the browser can't cache the decoded bitmap, e.g. due to unnecessary CSS filters applied on the same element every frame) can single-handedly blow the paint budget. Move `filter`/`blur` to a wrapping layer, not the raw `<img>`, so decode is cached.
6. **Count active GSAP tweens/ScrollTriggers.** Each `scrub: true` ScrollTrigger runs a calculation on every scroll tick; more than ~20-30 active scrubbed triggers simultaneously is a common real-world jank source. Use `batch()` or consolidate multiple triggers into one timeline where possible.

## 9. Mobile-Specific Reduction

```js
const isCoarsePointer = window.matchMedia('(pointer: coarse)').matches;
if (isCoarsePointer) {
  document.documentElement.classList.add('reduced-fx');
}
```

```css
.reduced-fx .depth-1, .reduced-fx .depth-5 { display: none; }      /* drop pure-decoration layers */
.reduced-fx [data-parallax] { transform: none !important; }        /* disable parallax math entirely */
.reduced-fx .float-loop { animation: none; }                       /* stop ambient loops */
```

Mobile GPUs, thermal throttling, and battery constraints make aggressive multi-layer parallax the single most common source of user-reported "laggy site" complaints — treat this reduction as mandatory, not optional polish.
