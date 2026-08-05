# Motion System

The scroll-architecture patterns, easing library, and timing conventions that every technique in `text-animations.md`, `directional-reveals.md`, and `inter-section-effects.md` is built on. GSAP + ScrollTrigger is the reference implementation; each pattern also notes the CSS-only fallback where one exists.

## 1. Setup

```html
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/gsap.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/ScrollTrigger.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/gsap@3.12.5/dist/Flip.min.js"></script>
<script>
  gsap.registerPlugin(ScrollTrigger, Flip);
  if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) {
    gsap.globalTimeline.timeScale(0.01); // near-instant, doesn't fully disable (see accessibility.md for the full kill-switch)
  }
</script>
```

## 2. Easing Curve Library

Named curves with real `cubic-bezier()` values and their intended use-case. Keep this consistent across a project — mixing arbitrary easing per-element reads as noisy, not premium.

| Name | `cubic-bezier` | Feel | Use for |
|---|---|---|---|
| `ease-cinematic` | `cubic-bezier(0.16, 1, 0.3, 1)` | Fast start, long soft settle | Hero entrances, headline reveals |
| `ease-elastic-soft` | `cubic-bezier(0.68, -0.55, 0.27, 1.55)` | Slight overshoot then settle | Card pop-ins, button hovers, float-loop bounce apex |
| `ease-out-expo` | `cubic-bezier(0.19, 1, 0.22, 1)` | Very fast start, dramatic decel | Clip-path wipes, curtain reveals |
| `ease-in-out-quart` | `cubic-bezier(0.77, 0, 0.175, 1)` | Symmetric, deliberate | Scroll-scrubbed transforms (scrub ties this to scroll position, so symmetry reads best) |
| `ease-anticipate` | `cubic-bezier(0.68, -0.6, 0.32, 1.6)` | Pulls back before committing | Theatrical text entrances, "wind-up" before a reveal |
| `ease-linear` | `cubic-bezier(0, 0, 1, 1)` (i.e. `none` in GSAP) | No easing | **Always** for `scrub: true` tweens — the scrollbar itself provides the "easing," double-easing a scrubbed tween causes a laggy, rubber-banded feel |
| `ease-out-soft` | `cubic-bezier(0.33, 1, 0.68, 1)` | Gentle decel, no overshoot | Fade-ins, opacity transitions, glow pulses |
| `ease-in-quart` | `cubic-bezier(0.5, 0, 0.75, 0)` | Slow start, fast finish | Exit animations (elements leaving the scene) |

```css
:root {
  --ease-cinematic: cubic-bezier(0.16, 1, 0.3, 1);
  --ease-elastic-soft: cubic-bezier(0.68, -0.55, 0.27, 1.55);
  --ease-out-expo: cubic-bezier(0.19, 1, 0.22, 1);
  --ease-anticipate: cubic-bezier(0.68, -0.6, 0.32, 1.6);
  --ease-out-soft: cubic-bezier(0.33, 1, 0.68, 1);
  --ease-in-quart: cubic-bezier(0.5, 0, 0.75, 0);
}
```

**Rule: never use custom easing on a `scrub: true` GSAP tween.** Always `ease: 'none'` — the position IS the scroll position, so any easing curve just adds lag/rubber-banding between the actual scrollbar and the visual state.

## 3. Timing/Duration Conventions

| Interaction type | Duration | Notes |
|---|---|---|
| Micro-interaction (hover, button press) | 150–250ms | Anything longer feels sluggish for direct-manipulation feedback |
| Text entrance (single element) | 600–900ms | Long enough to read as intentional, short enough not to stall the story |
| Text entrance (staggered, per-word/char) | 40–80ms stagger between units | Total sequence duration = `unitCount × stagger + singleUnitDuration` — keep total under ~2s for a headline or it drags |
| Section-level entrance (hero image, card) | 800–1200ms | The "big" cinematic beat duration |
| Float-loop (ambient idle motion) | 6–14s, `yoyo: true`, `repeat: -1` | Never shorter than 6s — fast idle loops read as nervous/glitchy, not premium |
| Scrub-tied animation | N/A — duration is scroll distance, not time | Express as `end: '+=800'` (px) or `end: 'bottom top'`, never a time-based duration |
| Curtain/reveal wipes | 700–1000ms | Matches `ease-out-expo` for a decisive, fast-then-settling feel |

## 4. The 9 Scroll Architecture Patterns

### Pattern 1 — Pinned Sticky

Element/section stays fixed in the viewport while its internal content changes as the user continues scrolling.

```js
ScrollTrigger.create({
  trigger: '.pinned-scene',
  start: 'top top',
  end: '+=1500',      // scene stays pinned for 1500px of scroll
  pin: true,
  pinSpacing: true,   // reserves the scroll distance in document flow
});
```

### Pattern 2 — Scrub Timeline

Animation progress is tied 1:1 to scroll position rather than time.

```js
gsap.timeline({
  scrollTrigger: {
    trigger: '.scrub-scene',
    start: 'top top',
    end: 'bottom top',
    scrub: 1,          // 1 = slight smoothing lag (in seconds) catching up to scroll; `true` = zero lag
  }
})
.to('.depth-3', { scale: 1.4, ease: 'none' })
.to('.depth-4', { opacity: 0, y: -100, ease: 'none' }, '<'); // '<' = start at same time as previous tween
```

### Pattern 3 — 6-Layer Parallax

Covered fully in `references/depth-system.md`; the GSAP equivalent of the hand-rolled version:

```js
gsap.utils.toArray('[data-depth]').forEach((layer) => {
  const speed = Number(layer.dataset.depth) === 0 ? 0.10
              : Number(layer.dataset.depth) === 1 ? 0.25
              : Number(layer.dataset.depth) === 2 ? 0.50
              : Number(layer.dataset.depth) === 3 ? 0.80
              : Number(layer.dataset.depth) === 4 ? 1.00 : 1.20;
  gsap.to(layer, {
    y: () => (1 - speed) * -300, // px of parallax travel across the scene
    ease: 'none',
    scrollTrigger: { trigger: layer.closest('.scene'), start: 'top bottom', end: 'bottom top', scrub: true },
  });
});
```

### Pattern 4 — Perspective Zoom

Camera-push illusion: scale + slight translateZ as the user scrolls into a scene.

```js
gsap.fromTo('.perspective-target',
  { scale: 0.8, z: -200 },
  { scale: 1.15, z: 0, ease: 'none',
    scrollTrigger: { trigger: '.zoom-scene', start: 'top 80%', end: 'top 20%', scrub: true } }
);
```

### Pattern 5 — DJI Scale-In Pin

Named for the drone-brand style hero: an object scales up dramatically while pinned, filling the viewport before releasing to the next section.

```js
ScrollTrigger.create({
  trigger: '.dji-scene',
  start: 'top top',
  end: '+=1200',
  pin: true,
  onUpdate: (self) => {
    gsap.set('.dji-hero', { scale: 1 + self.progress * 3, ease: 'none' });
    gsap.set('.dji-hero', { filter: `brightness(${1 - self.progress * 0.3})` });
  },
});
```

### Pattern 6 — Horizontal Scroll (Vertical Scroll → Horizontal Motion)

```js
const track = document.querySelector('.h-track');
gsap.to(track, {
  x: () => -(track.scrollWidth - window.innerWidth),
  ease: 'none',
  scrollTrigger: {
    trigger: '.h-scroll-section',
    start: 'top top',
    end: () => `+=${track.scrollWidth - window.innerWidth}`,
    pin: true,
    scrub: true,
    invalidateOnRefresh: true,
  },
});
```

### Pattern 7 — Cascading Card Stack

```js
gsap.utils.toArray('.stack-card').forEach((card, i) => {
  ScrollTrigger.create({
    trigger: card,
    start: 'top 85%',
    onEnter: () => gsap.to(card, { y: 0, opacity: 1, rotate: 0, duration: 0.8, ease: 'var(--ease-cinematic)' }),
  });
  gsap.set(card, { y: 60 + i * 10, opacity: 0, rotate: i % 2 === 0 ? -3 : 3 });
});
```

### Pattern 8 — GSAP Flip Cross-Section Travel

An element visually "travels" from one section's layout position to another's using FLIP (First-Last-Invert-Play), covered fully with worked code in `references/inter-section-effects.md`.

### Pattern 9 — Stagger Grid

```js
gsap.from('.grid-item', {
  opacity: 0,
  y: 40,
  duration: 0.6,
  ease: 'var(--ease-out-soft)',
  stagger: { amount: 0.6, grid: 'auto', from: 'start' },
  scrollTrigger: { trigger: '.grid-section', start: 'top 75%' },
});
```

## 5. Choosing Scrub vs. Trigger (onEnter)

- **Use `scrub: true`** when the animation should feel like the user's scroll wheel *is* the play head — camera moves, parallax, reveals that should feel "attached" to scroll physically.
- **Use `onEnter`/`toggleActions` (trigger-based, plays once)** for discrete "this element has arrived" moments — card pop-ins, headline reveals — where a fixed-duration animation reads better than a scroll-locked one.

```js
gsap.from('.headline', {
  opacity: 0, y: 40,
  duration: 0.9, ease: 'var(--ease-cinematic)',
  scrollTrigger: { trigger: '.headline', start: 'top 80%', toggleActions: 'play none none reverse' },
});
```

`toggleActions: 'play none none reverse'` = play on enter scrolling down, do nothing on leave scrolling down, do nothing on enter scrolling up (re-entering from below), reverse on leave scrolling up (scrolling back past it upward) — the standard "animate in once per direction" pattern.
