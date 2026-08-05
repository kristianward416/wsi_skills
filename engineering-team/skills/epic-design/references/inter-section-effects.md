# Inter-Section Effects

Techniques for elements and behavior that persist, travel, or interact *across* section boundaries — floating products, cross-section travel, sticky column journeys, and card-stack overlaps.

## 1. Inter-Section Floating Product

A product/hero image appears to rise up from behind one section and persist as the user scrolls into the next — the single most requested "premium" effect ("product rises between sections").

```html
<section class="scene-a">…</section>
<div class="floating-product-wrapper">
  <img class="floating-product" src="bottle.png" alt="Insulated bottle" />
</div>
<section class="scene-b">…</section>
```

```css
.floating-product-wrapper {
  position: sticky;
  top: 10vh;
  height: 0;               /* doesn't consume layout height itself */
  z-index: 10;
  display: flex;
  justify-content: center;
  pointer-events: none;
}
.floating-product {
  width: min(40vw, 500px);
  clip-path: inset(100% 0 0 0); /* born from the seam with scene-a */
}
```

```js
gsap.timeline({
  scrollTrigger: { trigger: '.floating-product-wrapper', start: 'top 90%', end: '+=100%', scrub: true },
})
.to('.floating-product', { clipPath: 'inset(0% 0 0 0)', ease: 'none' }, 0)
.to('.floating-product', { y: -80, scale: 1.1, ease: 'none' }, 0.3)
.to('.floating-product', { y: -160, opacity: 0, ease: 'none' }, 0.75); // scatters/fades before scene-b fully takes over
```

The `position: sticky` wrapper is what lets the product visually "hold" across the section seam rather than scrolling with either section individually — this is the mechanical trick behind the whole effect.

## 2. GSAP Flip — Cross-Section Travel

An element genuinely relocates in the DOM/layout between two different positions (e.g. a product thumbnail in section A becomes the hero image in section B) and Flip animates the transition between those two real layout states, rather than faking it with absolute positioning.

```js
function flipProductToHero() {
  const state = Flip.getState('.product-instance');
  document.querySelector('.hero-slot').appendChild(document.querySelector('.product-instance'));
  Flip.from(state, {
    duration: 1,
    ease: 'var(--ease-cinematic)',
    scale: true,
    absolute: true, // element is absolutely positioned during the flip so surrounding layout doesn't jump
  });
}

ScrollTrigger.create({
  trigger: '.section-b',
  start: 'top 60%',
  onEnter: flipProductToHero,
});
```

Use Flip specifically when the "before" and "after" states are genuinely different layout contexts (different grid, different parent, different size context) — for simple travel within the same coordinate space, plain `transform` tweens (see §3) are cheaper and don't require Flip's state-capture overhead.

## 3. Curved Path Travel

An element travels along a curved (non-linear) path between two points as the user scrolls, rather than a straight line — reads as more "alive."

```js
// Quadratic bezier point at t, given start/control/end points
function bezierPoint(t, p0, p1, p2) {
  const x = (1 - t) ** 2 * p0.x + 2 * (1 - t) * t * p1.x + t ** 2 * p2.x;
  const y = (1 - t) ** 2 * p0.y + 2 * (1 - t) * t * p1.y + t ** 2 * p2.y;
  return { x, y };
}

const start = { x: 0, y: 0 };
const control = { x: 300, y: -400 }; // pulls the arc upward/rightward
const end = { x: 600, y: 0 };

ScrollTrigger.create({
  trigger: '.curved-travel-section',
  start: 'top top',
  end: 'bottom top',
  scrub: true,
  onUpdate: (self) => {
    const { x, y } = bezierPoint(self.progress, start, control, end);
    gsap.set('.traveling-element', { x, y });
  },
});
```

For a simpler declarative version using GSAP's `MotionPathPlugin` (separate CDN import: `MotionPathPlugin.min.js`):

```js
gsap.to('.traveling-element', {
  motionPath: {
    path: '#travel-svg-path', // an actual <path> element defines the curve
    align: '#travel-svg-path',
    autoRotate: false,
  },
  ease: 'none',
  scrollTrigger: { trigger: '.curved-travel-section', start: 'top top', end: 'bottom top', scrub: true },
});
```

## 4. Sticky Column Journey

Content lives in a sticky sidebar while a longer scrolling column of text/media passes by it — the layout behind most "editorial scroll" and product-spec pages.

```css
.sticky-journey {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 4rem;
}
.sticky-journey-media {
  position: sticky;
  top: 15vh;
  height: 70vh;
  align-self: start;
}
.sticky-journey-copy { /* normal flow, taller than the sticky column, drives the scroll distance */ }
```

```js
// Swap the sticky media's content/state as different copy blocks scroll past
gsap.utils.toArray('.copy-block').forEach((block, i) => {
  ScrollTrigger.create({
    trigger: block,
    start: 'top center',
    end: 'bottom center',
    onEnter: () => updateStickyMedia(i),
    onEnterBack: () => updateStickyMedia(i),
  });
});

function updateStickyMedia(index) {
  gsap.to('.sticky-journey-media img', {
    opacity: 0, duration: 0.3, onComplete: () => {
      document.querySelector('.sticky-journey-media img').src = mediaSources[index];
      gsap.to('.sticky-journey-media img', { opacity: 1, duration: 0.3 });
    },
  });
}
```

## 5. Cascading Card Stack

Cards overlap and stack as the user scrolls, each new card sliding partially over the previous one and settling — see also Pattern 7 in `references/motion-system.md` for the entrance timing; this section covers the stacking/overlap mechanics specifically.

```css
.stack-container { position: relative; }
.stack-card {
  position: sticky;
  top: 10vh;
  height: 60vh;
  margin-bottom: -20vh; /* creates the overlap as the next card's sticky position engages */
  border-radius: 24px;
  box-shadow: 0 20px 60px rgba(0,0,0,0.3);
}
```

Each card becomes sticky at the same `top` offset, so as the user scrolls, the next card slides up and settles directly over the previous one (which is now pinned behind it) — pure CSS `position: sticky`, no JS required for the core mechanic. Add a subtle scale-down on the covered card for visual depth:

```js
gsap.utils.toArray('.stack-card').forEach((card, i) => {
  if (i === 0) return;
  ScrollTrigger.create({
    trigger: card,
    start: 'top top',
    end: 'bottom top',
    scrub: true,
    onUpdate: (self) => {
      gsap.set(card.previousElementSibling, {
        scale: 1 - self.progress * 0.05,
        filter: `brightness(${1 - self.progress * 0.2})`,
      });
    },
  });
});
```

## 6. Section Overlap (Non-Card, Full-Bleed)

Two full-bleed sections overlap at their seam rather than transitioning cleanly — the next section slides up and over the tail end of the previous one.

```css
.overlap-section {
  position: relative;
  margin-top: -15vh; /* pulls this section up over the previous one's bottom edge */
  z-index: 2;
  border-radius: 32px 32px 0 0;
  background: var(--section-bg);
}
```

Combine with a subtle entrance so the overlap doesn't feel abrupt:

```js
gsap.from('.overlap-section', {
  y: 60,
  ease: 'var(--ease-cinematic)',
  duration: 1,
  scrollTrigger: { trigger: '.overlap-section', start: 'top 90%' },
});
```

## 7. Floating Elements Between Sections (Decorative, Non-Product)

Smaller decorative floats (not the hero product) that drift across the seam independently of either section's own parallax — typically depth-1/2, `aria-hidden`.

```js
gsap.to('.inter-float', {
  y: () => -window.innerHeight * 0.4,
  x: () => gsap.utils.random(-40, 40),
  rotate: () => gsap.utils.random(-15, 15),
  ease: 'none',
  scrollTrigger: {
    trigger: '.inter-float',
    start: 'top bottom',
    end: 'bottom top',
    scrub: true,
  },
});
```

## 8. DJI Scale-In Pin at a Section Boundary

When the "DJI"-style dramatic scale-in (see `references/motion-system.md` Pattern 5) is used specifically to *transition into* the next section (rather than within a single hero), release the pin directly into the next section's entrance so there's no dead scroll space:

```js
ScrollTrigger.create({
  trigger: '.dji-transition-scene',
  start: 'top top',
  end: '+=1000',
  pin: true,
  onLeave: () => gsap.set('.next-section', { zIndex: 3 }), // ensure next section stacks correctly on release
});
```

Pair with the z-index/stacking guidance in `references/depth-system.md` §5 — pinned scenes need the *following* scene to have a higher explicit `z-index` or the release will visually glitch for one frame.
