# Accessibility

Every non-negotiable rule here maps directly to WCAG 2.1 AA. None of these are optional polish — the skill treats them as mandatory in every output.

## 1. `prefers-reduced-motion` — Baseline Implementation

This block is required in every build, no exceptions:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

This is a minimum floor, not a complete solution — it stops CSS `animation`/`transition` runaway, but it does **not** stop JS-driven GSAP tweens, ScrollTrigger scrubbing, or parallax `transform` writes on scroll. Those need explicit handling:

```js
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

if (prefersReducedMotion) {
  // Kill all ScrollTrigger-driven scrub/pin behavior
  ScrollTrigger.getAll().forEach((trigger) => trigger.kill());
  gsap.globalTimeline.clear();

  // Reveal all scroll-triggered content immediately in its final state,
  // rather than leaving it invisible with no way to trigger the reveal
  document.querySelectorAll('[data-animate], [data-reveal]').forEach((el) => {
    el.style.opacity = '1';
    el.style.transform = 'none';
    el.style.clipPath = 'none';
    el.style.filter = 'none';
  });

  // Disable parallax offset writes entirely
  document.documentElement.classList.add('motion-reduced');
}
```

```css
.motion-reduced [data-depth] { transform: none !important; }
.motion-reduced .float-loop, .motion-reduced .drift { animation: none !important; }
```

Also listen for the setting changing mid-session (users can toggle OS settings without reloading):

```js
window.matchMedia('(prefers-reduced-motion: reduce)').addEventListener('change', (e) => {
  document.documentElement.classList.toggle('motion-reduced', e.matches);
});
```

## 2. Scroll-Hijacking: Keyboard and Screen-Reader Implications

"Pinned"/scrub-driven sections (`position: sticky` or GSAP `pin: true`) visually decouple scroll distance from viewport progress — the user scrolls but the page doesn't visibly move for a while. This breaks several assumptions keyboard and screen-reader users rely on:

- **Keyboard users pressing Page Down / Space** expect each press to move roughly one viewport. A pinned section that consumes 400vh of scroll distance for one visual "scene" can make keyboard navigation feel broken — many page-downs pass with no visible change.
- **Screen reader users** navigating by heading/landmark (not by scroll at all) are entirely unaffected by scroll-hijacking *visually*, but if content is hidden via `opacity: 0` + `clip-path` until a scroll-triggered class is added, and that JS never fires for a reason unrelated to visual scroll (e.g. `IntersectionObserver` firing differently, or the observer never attaching because JS errored), the content can become **permanently invisible and undiscoverable** even though it's in the DOM and would normally be read.
- **Voice control / switch access users** who navigate by landmark or "scroll down" voice commands face the same page-down mismatch as keyboard users.

### Required mitigations

1. **Never gate final content visibility on JS success alone.** Use a no-JS/no-animation fallback state in CSS so content defaults to visible:

   ```css
   /* Default: visible. JS ADDS the animated hidden-state class; it never removes a default-hidden state. */
   [data-animate] { opacity: 1; transform: none; }
   [data-animate].js-ready { opacity: 0; transform: translateY(40px); } /* hidden only once JS confirms it can also reveal it */
   [data-animate].js-ready.is-visible { opacity: 1; transform: none; transition: opacity 0.6s, transform 0.6s; }
   ```

   ```js
   // Only opt content INTO the hidden starting state once we know the reveal system is live
   document.querySelectorAll('[data-animate]').forEach((el) => el.classList.add('js-ready'));
   ```

   This way, if JS fails to load/execute at all, content is visible by default rather than stuck invisible.

2. **Never trap keyboard focus inside a pinned section.** Pinning must be purely visual (`position`/`transform`); it must never intercept `Tab`, arrow keys, or `Enter`/`Space` on focusable elements inside the pinned content. Test by tabbing through a pinned section — focus should move to each interactive element in DOM order exactly as it would on a static page.

3. **Respect `scroll-behavior: auto` under reduced motion** (already in the baseline CSS block above) — smooth-scroll snapping/easing can otherwise fight against a keyboard user's Page Down expectations.

4. **Provide a skip mechanism for long scroll-jacked sequences.** If a single scene consumes several viewport-heights of scroll distance purely for a visual sequence with no unique content, consider a visually-hidden skip link:

   ```html
   <a href="#after-cinematic-intro" class="skip-link">Skip intro animation</a>
   ```

   ```css
   .skip-link {
     position: absolute; left: -9999px;
   }
   .skip-link:focus {
     left: 1rem; top: 1rem; z-index: 100;
     background: #fff; color: #000; padding: 0.5rem 1rem;
   }
   ```

## 3. ARIA for Decorative vs. Meaningful Layers

Every purely decorative layer (background textures, glow blobs, particles, floating shapes with no independent meaning) gets `aria-hidden="true"` — this is Rule #8 of the skill and is non-negotiable:

```html
<div class="layer depth-0" data-depth="0" aria-hidden="true"><!-- background texture --></div>
<div class="layer depth-1" data-depth="1" aria-hidden="true"><!-- glow --></div>
<div class="layer depth-5" data-depth="5" aria-hidden="true"><!-- sparkles --></div>
```

Layers carrying actual content (headings, body copy, the hero product image with meaning) never get `aria-hidden` — they get proper semantics instead:

```html
<div class="layer depth-3">
  <img src="bottle.png" alt="Matte black 500ml insulated bottle, angled three-quarter view" />
</div>
<div class="layer depth-4">
  <h1 class="split-text" data-animate="converge">Cold for 24 hours. Hot for 12.</h1>
</div>
```

### Alt text rules for animated images

- Product/hero images: describe the object and its meaningful visual state, not the animation ("Matte black bottle" not "Bottle floating in with elastic bounce")
- Decorative accents: `alt=""` (empty, not missing) if not `aria-hidden` for some structural reason — empty alt tells screen readers to skip it without erroring
- Never put marketing copy that also exists as real text into an image's `alt` if it duplicates an adjacent heading — this causes double-announcement

### Split-text / letter-by-letter animation and screen readers

Splitting a heading into individual `<span>` characters for a "kinetic typography" effect (see `references/text-animations.md`) breaks how screen readers announce the phrase — many will spell out or pause oddly between spans. Always keep the real text in an `aria-label` on the parent and hide the split spans from the accessibility tree:

```html
<h1 aria-label="Cold for 24 hours. Hot for 12.">
  <span aria-hidden="true" class="split-char">C</span>
  <span aria-hidden="true" class="split-char">o</span>
  <span aria-hidden="true" class="split-char">l</span>
  <!-- … -->
</h1>
```

This is the single most commonly missed accessibility requirement in kinetic-typography implementations — flag it every time split-text is used.

## 4. Contrast Under Motion

Text laid over parallax/video/image layers must maintain WCAG AA contrast (4.5:1 for body text, 3:1 for large text) at *every point in its scroll journey*, not just its resting position — a headline that starts on a dark background but scrolls over a light image mid-animation needs either a scrim, a text-shadow, or a background blend that holds contrast throughout:

```css
.hero-heading {
  color: #fff;
  text-shadow: 0 2px 24px rgba(0,0,0,0.6); /* holds legibility over variable imagery */
}
/* Or, a gradient scrim behind text that travels with it */
.text-scrim {
  background: linear-gradient(to bottom, rgba(0,0,0,0) 0%, rgba(0,0,0,0.55) 100%);
}
```

## 5. Focus Order and Off-Canvas Elements

Elements positioned with large negative `transform` offsets (e.g. "scattered" companions post-scroll, or elements staged off-screen before a directional reveal) must still make sense in **DOM order** for keyboard/screen-reader navigation, since visual position and DOM position diverge under transform-based motion. Never reorder DOM purely to achieve a visual staging position — use `transform`/`order` (flex/grid `order` is visual-only and also doesn't affect DOM tab order, which is what's wanted) rather than physically relocating markup.

## 6. Quick Accessibility Checklist

- [ ] `prefers-reduced-motion` CSS block present
- [ ] JS also checks `prefers-reduced-motion` and kills ScrollTrigger/GSAP tweens, not just CSS transitions
- [ ] All content visible by default; JS *adds* hidden states, never the only path to visible states
- [ ] No focus trapping inside pinned/sticky sections
- [ ] All decorative layers `aria-hidden="true"`
- [ ] All meaningful images have descriptive, non-redundant `alt`
- [ ] Split-text effects preserve a real `aria-label` on the parent, spans are `aria-hidden`
- [ ] Text-over-imagery maintains 4.5:1 (body) / 3:1 (large) contrast throughout its scroll journey
- [ ] Tab order matches logical reading order regardless of visual transform positioning
