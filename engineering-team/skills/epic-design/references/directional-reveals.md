# Directional Reveals

8 "born from top/sides" techniques, all built on `clip-path`, `transform`, or a combination — every one of these is compositor-safe (see `references/performance.md` §2).

## 1. Top-Down Clip Birth

Content appears to be "born" downward from a horizontal seam, as if unrolling out of the top edge.

```css
.clip-birth-top {
  clip-path: inset(0 0 100% 0); /* fully hidden — bottom inset is 100% */
}
```

```js
gsap.to('.clip-birth-top', {
  clipPath: 'inset(0 0 0% 0)',
  duration: 0.9,
  ease: 'var(--ease-out-expo)',
  scrollTrigger: { trigger: '.clip-birth-top', start: 'top 80%' },
});
```

Map to the skill's decision rule: **"section drops down"** → this exact pattern (`inset(0 0 100% 0)` → `inset(0)`).

## 2. Curtain Panel Roll-Up

Two (or more) solid panels cover content and roll upward/outward like theater curtains, revealing what's beneath.

```html
<div class="curtain-scene">
  <div class="curtain-panel curtain-left"></div>
  <div class="curtain-panel curtain-right"></div>
  <div class="curtain-content"><!-- revealed content --></div>
</div>
```

```css
.curtain-scene { position: relative; overflow: hidden; }
.curtain-panel {
  position: absolute; top: 0; bottom: 0; width: 50%;
  background: #111; z-index: 5;
}
.curtain-left { left: 0; transform-origin: left; }
.curtain-right { right: 0; transform-origin: right; }
```

```js
gsap.timeline({ scrollTrigger: { trigger: '.curtain-scene', start: 'top 60%' } })
  .to('.curtain-left', { scaleX: 0, duration: 1, ease: 'var(--ease-out-expo)' })
  .to('.curtain-right', { scaleX: 0, duration: 1, ease: 'var(--ease-out-expo)' }, '<'); // simultaneous
```

For a true vertical "roll up" (theater curtain, not sliding panel):

```js
gsap.to('.curtain-panel', {
  yPercent: -100,
  duration: 1.1,
  ease: 'var(--ease-out-expo)',
  stagger: 0.1,
  scrollTrigger: { trigger: '.curtain-scene', start: 'top 60%' },
});
```

## 3. Circle Iris Expand

A circular `clip-path` expands from a point to reveal full content — classic "camera iris" transition.

```css
.iris-reveal {
  clip-path: circle(0% at 50% 50%);
}
```

```js
gsap.to('.iris-reveal', {
  clipPath: 'circle(75% at 50% 50%)', // 75% covers corners of a typical 16:9 viewport; use 141% for full corner coverage on any aspect ratio
  duration: 1.2,
  ease: 'var(--ease-out-expo)',
  scrollTrigger: { trigger: '.iris-reveal', start: 'top 65%' },
});
```

Map to decision rule: **"circle opens"** → circle iris expand, exactly this.

## 4. Window Pane Iris

Like circle iris but composed of multiple rectangular "panes" (a grid) that each open outward from center — reads as a window shutter or aperture-blade opening.

```html
<div class="pane-grid">
  <!-- generate 3x3 or 4x4 divs via JS/templating -->
  <div class="pane"></div>
  <!-- ...repeat -->
</div>
```

```css
.pane-grid { display: grid; grid-template-columns: repeat(4, 1fr); grid-template-rows: repeat(4, 1fr); position: absolute; inset: 0; z-index: 5; }
.pane { background: #0a0a0a; transform: scale(1); }
```

```js
gsap.to('.pane', {
  scale: 0,
  duration: 0.8,
  ease: 'var(--ease-out-expo)',
  stagger: { amount: 0.5, grid: [4, 4], from: 'center' }, // opens outward from the center pane
  scrollTrigger: { trigger: '.pane-grid', start: 'top 60%' },
});
```

Map to decision rule: **"whole section transforms"** → window pane iris + scrub timeline (swap `stagger` timing for a `scrollTrigger: { scrub: true }` timeline if the reveal should be scroll-scrubbed rather than a one-shot trigger).

## 5. Section Peel

A section appears to peel back like a page corner, revealing the next section beneath.

```css
.peel-section {
  transform-origin: bottom right;
  transform-style: preserve-3d;
}
```

```js
gsap.to('.peel-section', {
  rotationX: -80,
  y: '-40%',
  opacity: 0.4,
  ease: 'none',
  scrollTrigger: { trigger: '.peel-section', start: 'top top', end: 'bottom top', scrub: true },
});
```

Parent needs `perspective` for the rotation to read correctly:

```css
.peel-wrapper { perspective: 1500px; overflow: clip; }
```

## 6. Diagonal Wipe

A diagonal `clip-path` polygon sweeps across to reveal content — more dynamic than a straight horizontal/vertical wipe.

```css
.diagonal-wipe {
  clip-path: polygon(0 0, 0 0, 0 100%, 0 100%); /* collapsed to a line on the left edge */
}
```

```js
gsap.to('.diagonal-wipe', {
  clipPath: 'polygon(0 0, 100% 0, 100% 100%, 0% 100%)', // full rectangle
  duration: 1,
  ease: 'var(--ease-out-expo)',
  scrollTrigger: { trigger: '.diagonal-wipe', start: 'top 75%' },
});
```

For a genuinely diagonal (angled) sweep rather than a straight rectangle expand, animate a polygon with an offset top/bottom edge:

```js
gsap.fromTo('.diagonal-wipe',
  { clipPath: 'polygon(0% 0%, 15% 0%, 0% 100%, 0% 100%)' },
  { clipPath: 'polygon(0% 0%, 100% 0%, 100% 100%, 0% 100%)',
    duration: 1, ease: 'var(--ease-out-expo)',
    scrollTrigger: { trigger: '.diagonal-wipe', start: 'top 75%' } }
);
```

## 7. Clip-Path Birth (General "Rises from Section")

The general-purpose version referenced across the skill's decision engine for "elements between sections" and "rises from section": an element is clipped to nothing at a section seam and grows into full visibility as if emerging from that boundary.

```css
.clip-path-birth {
  clip-path: inset(100% 0 0 0); /* hidden below the bottom edge, i.e. "inside" the section boundary */
  transform: translateY(10%);
}
```

```js
gsap.to('.clip-path-birth', {
  clipPath: 'inset(0% 0 0 0)',
  y: 0,
  duration: 1,
  ease: 'var(--ease-cinematic)',
  scrollTrigger: { trigger: '.clip-path-birth', start: 'top 85%' },
});
```

This is the combinator used with "Inter-section floating product" (see `references/inter-section-effects.md`) — the product clip-birth-reveals at the seam between two sections, then transitions into a floating/scattering behavior once fully visible.

## 8. Offset Directional Fly-In (Sides)

Elements fly in from the left/right edges of the viewport rather than from top/bottom — the base pattern behind "text flies in from sides."

```js
gsap.utils.toArray('[data-fly-from]').forEach((el) => {
  const dir = el.dataset.flyFrom; // 'left' | 'right'
  gsap.from(el, {
    x: dir === 'left' ? -120 : 120,
    opacity: 0,
    duration: 0.8,
    ease: 'var(--ease-cinematic)',
    scrollTrigger: { trigger: el, start: 'top 82%' },
  });
});
```

```html
<h2 data-fly-from="left">Built for the field.</h2>
<p data-fly-from="right">Every detail engineered to disappear.</p>
```

For alternating multi-line headlines (odd lines from left, even from right — a common "Split converge / offset diagonal" companion pattern):

```js
document.querySelectorAll('.alt-line').forEach((line, i) => {
  gsap.from(line, {
    x: i % 2 === 0 ? -150 : 150,
    opacity: 0,
    duration: 0.8,
    ease: 'var(--ease-cinematic)',
    scrollTrigger: { trigger: line, start: 'top 85%' },
  });
});
```
