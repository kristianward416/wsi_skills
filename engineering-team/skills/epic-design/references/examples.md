# Examples

Four worked, end-to-end builds combining techniques from the other reference files. Each notes exactly which techniques are used and why, per the skill's "note at top listing techniques applied" output requirement.

---

## Example 1 — Product Launch Hero (Single Section)

**Techniques applied:** 6-layer parallax (`depth-system.md`) · Split converge (`text-animations.md`) · Float loop · DJI scale-in pin (`motion-system.md`) · prefers-reduced-motion fallback (`accessibility.md`)

```html
<section class="scene" data-scene="hero" style="--scene-height: 250vh">
  <div class="layer depth-0" data-depth="0" aria-hidden="true">
    <picture>
      <source srcset="studio-bg-2560.avif" type="image/avif">
      <img src="studio-bg-2560.jpg" alt="" class="bg-fill" loading="eager" fetchpriority="high">
    </picture>
  </div>

  <div class="layer depth-1" data-depth="1" aria-hidden="true">
    <div class="glow-blob"></div>
  </div>

  <div class="layer depth-2" data-depth="2" aria-hidden="true">
    <img src="leaf-accent.png" class="companion" style="right: calc(50% - 260px - 4vw);" alt="">
  </div>

  <div class="layer depth-3" data-depth="3">
    <img class="product-hero float-loop" src="bottle-1200.png"
         srcset="bottle-800.png 800w, bottle-1200.png 1200w, bottle-1600.png 1600w"
         sizes="60vw" fetchpriority="high"
         alt="Matte black 500ml insulated bottle, angled three-quarter view">
  </div>

  <div class="layer depth-4" data-depth="4">
    <h1 class="split-converge" aria-label="Cold for 24 hours. Hot for 12.">Cold for 24 hours. Hot for 12.</h1>
    <p class="light-word">Engineered vacuum insulation in a shape that disappears in your hand.</p>
    <button class="cta">Shop now</button>
  </div>

  <div class="layer depth-5" data-depth="5" aria-hidden="true">
    <div class="sparkle-field"></div>
  </div>
</section>
```

```css
.scene { position: relative; isolation: isolate; overflow: clip; height: var(--scene-height); }
.layer { position: absolute; inset: 0; will-change: auto; }
.depth-0 { z-index: 0; filter: blur(8px); transform: scale(0.70); }
.depth-1 { z-index: 1; filter: blur(4px); transform: scale(0.85); }
.depth-2 { z-index: 2; }
.depth-3 { z-index: 3; display: flex; align-items: center; justify-content: center; }
.depth-4 { z-index: 4; display: flex; flex-direction: column; align-items: center; justify-content: center; text-align: center; }
.depth-5 { z-index: 5; }

.product-hero { width: min(55vw, 620px); }
.float-loop { animation: float 9s ease-in-out infinite alternate; }
@keyframes float { 0% { transform: translateY(0); } 100% { transform: translateY(-16px); } }

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

```js
gsap.registerPlugin(ScrollTrigger);
const reduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

// depth parallax
const speeds = { 0: 0.10, 1: 0.25, 2: 0.50, 3: 0.80, 4: 1.00, 5: 1.20 };
if (!reduced) {
  document.querySelectorAll('[data-scene="hero"] [data-depth]').forEach((layer) => {
    gsap.to(layer, {
      y: () => (1 - speeds[Number(layer.dataset.depth)]) * -300,
      ease: 'none',
      scrollTrigger: { trigger: '.scene', start: 'top top', end: 'bottom top', scrub: true },
    });
  });

  // DJI scale-in on the hero image while pinned for the first 1200px
  ScrollTrigger.create({
    trigger: '.scene', start: 'top top', end: '+=1200', pin: true,
    onUpdate: (self) => gsap.set('.product-hero', { scale: 1 + self.progress * 0.6 }),
  });

  // split converge headline
  const chars = document.querySelector('h1').textContent.split('');
  const h1 = document.querySelector('h1');
  h1.textContent = '';
  chars.forEach((c) => {
    const span = document.createElement('span');
    span.textContent = c; span.setAttribute('aria-hidden', 'true'); span.style.display = 'inline-block';
    gsap.set(span, { x: gsap.utils.random(-150, 150), y: gsap.utils.random(-100, 100), opacity: 0 });
    h1.appendChild(span);
  });
  gsap.to(h1.children, { x: 0, y: 0, opacity: 1, duration: 1, stagger: 0.02, ease: 'cubic-bezier(0.16,1,0.3,1)',
    scrollTrigger: { trigger: '.scene', start: 'top 90%' } });
} else {
  document.querySelectorAll('[data-depth]').forEach((l) => (l.style.transform = 'none'));
}
```

---

## Example 2 — Product Showcase (Multi-Section, Inter-Section Floating Product)

**Techniques applied:** Inter-section floating product · Clip-path birth · Sticky column journey · Word-by-word scroll lighting · Cascading card stack (features section)

Structure: three sections — `intro` → `spec-journey` (sticky column) → `features` (card stack) — with one product image that visually persists across the seam between `intro` and `spec-journey`.

```html
<section class="intro">…</section>

<div class="floating-product-wrapper" aria-hidden="true">
  <img class="floating-product" src="device.png" alt="">
</div>

<section class="spec-journey">
  <div class="sticky-journey">
    <div class="sticky-journey-media">
      <img id="spec-media" src="spec-1.jpg" alt="Device detail: aluminum unibody">
    </div>
    <div class="sticky-journey-copy">
      <p class="copy-block light-word">Machined from a single block of aluminum...</p>
      <p class="copy-block light-word">Every port precision-cut to 0.02mm tolerance...</p>
      <p class="copy-block light-word">Weighs 38% less than the previous generation...</p>
    </div>
  </div>
</section>

<section class="features">
  <div class="stack-container">
    <div class="stack-card">Battery: 22 hours</div>
    <div class="stack-card">Display: 120Hz ProMotion</div>
    <div class="stack-card">Weight: 1.2kg</div>
  </div>
</section>
```

```css
.floating-product-wrapper { position: sticky; top: 8vh; height: 0; z-index: 10; display: flex; justify-content: center; pointer-events: none; }
.floating-product { width: min(35vw, 420px); clip-path: inset(100% 0 0 0); }

.sticky-journey { display: grid; grid-template-columns: 1fr 1fr; gap: 4rem; padding: 10vh 6vw; }
.sticky-journey-media { position: sticky; top: 15vh; height: 70vh; align-self: start; }
.copy-block { min-height: 60vh; display: flex; align-items: center; font-size: 1.5rem; }

.stack-container { position: relative; padding: 20vh 6vw; }
.stack-card {
  position: sticky; top: 10vh; height: 50vh; margin-bottom: -20vh;
  border-radius: 24px; box-shadow: 0 20px 60px rgba(0,0,0,0.3);
  display: flex; align-items: center; justify-content: center; font-size: 2rem; background: #1a1a1a; color: #fff;
}
```

```js
// Floating product birth + travel
gsap.timeline({
  scrollTrigger: { trigger: '.floating-product-wrapper', start: 'top 90%', end: '+=100%', scrub: true },
})
.to('.floating-product', { clipPath: 'inset(0% 0 0 0)', ease: 'none' }, 0)
.to('.floating-product', { y: -100, scale: 1.05, ease: 'none' }, 0.4)
.to('.floating-product', { opacity: 0, y: -180, ease: 'none' }, 0.8);

// Sticky column swaps media per copy block
const mediaSources = ['spec-1.jpg', 'spec-2.jpg', 'spec-3.jpg'];
document.querySelectorAll('.copy-block').forEach((block, i) => {
  ScrollTrigger.create({
    trigger: block, start: 'top center', end: 'bottom center',
    onEnter: () => (document.getElementById('spec-media').src = mediaSources[i]),
    onEnterBack: () => (document.getElementById('spec-media').src = mediaSources[i]),
  });
});

// Word-lighting on each copy block
document.querySelectorAll('.light-word').forEach((p) => {
  const words = p.textContent.split(/(\s+)/).map((w) => `<span class="lw">${w}</span>`).join('');
  p.innerHTML = words;
  ScrollTrigger.create({
    trigger: p, start: 'top 70%', end: 'bottom 40%',
    onUpdate: (self) => {
      const spans = p.querySelectorAll('.lw');
      const lit = Math.floor(self.progress * spans.length);
      spans.forEach((s, i) => s.classList.toggle('is-lit', i <= lit));
    },
  });
});

// Card stack darken-on-cover
gsap.utils.toArray('.stack-card').forEach((card, i) => {
  if (i === 0) return;
  ScrollTrigger.create({
    trigger: card, start: 'top top', end: 'bottom top', scrub: true,
    onUpdate: (self) => gsap.set(card.previousElementSibling, { scale: 1 - self.progress * 0.05, filter: `brightness(${1 - self.progress * 0.2})` }),
  });
});
```

---

## Example 3 — Full Story-Scroll Page (Editorial / Brand Narrative)

**Techniques applied:** Curtain panel roll-up (intro) · Masked line reveal · Theatrical enter+exit · Perspective zoom · Horizontal scroll gallery · Circle iris expand (closing CTA)

Section sequence: `curtain-intro` → `manifesto` (theatrical text) → `zoom-statement` (perspective zoom on a single image+word pairing) → `gallery` (horizontal scroll) → `cta` (iris expand reveal).

```js
gsap.registerPlugin(ScrollTrigger);

// 1. Curtain intro
gsap.to('.curtain-panel', {
  yPercent: -100, duration: 1.1, ease: 'cubic-bezier(0.19,1,0.22,1)', stagger: 0.1,
  scrollTrigger: { trigger: '.curtain-intro', start: 'top 60%' },
});

// 2. Manifesto — masked line reveal + theatrical exit
gsap.utils.toArray('.masked-line').forEach((line, i) => {
  gsap.to(line, {
    clipPath: 'inset(0 0% 0 0)', duration: 0.9, delay: i * 0.15,
    ease: 'cubic-bezier(0.19,1,0.22,1)',
    scrollTrigger: { trigger: '.manifesto', start: 'top 75%' },
  });
});
gsap.to('.manifesto', {
  opacity: 0, y: -60, ease: 'none',
  scrollTrigger: { trigger: '.manifesto', start: 'bottom 60%', end: 'bottom top', scrub: true },
});

// 3. Perspective zoom statement
gsap.fromTo('.zoom-image', { scale: 0.8 }, {
  scale: 1.2, ease: 'none',
  scrollTrigger: { trigger: '.zoom-statement', start: 'top bottom', end: 'bottom top', scrub: true },
});

// 4. Horizontal scroll gallery
const track = document.querySelector('.gallery-track');
gsap.to(track, {
  x: () => -(track.scrollWidth - window.innerWidth),
  ease: 'none',
  scrollTrigger: {
    trigger: '.gallery', start: 'top top',
    end: () => `+=${track.scrollWidth - window.innerWidth}`,
    pin: true, scrub: true, invalidateOnRefresh: true,
  },
});

// 5. Closing CTA — circle iris expand
gsap.to('.cta-iris', {
  clipPath: 'circle(75% at 50% 50%)', duration: 1.2, ease: 'cubic-bezier(0.19,1,0.22,1)',
  scrollTrigger: { trigger: '.cta', start: 'top 65%' },
});
```

```css
.cta-iris { clip-path: circle(0% at 50% 50%); background: #0a0a0a; color: #fff; }
.gallery { overflow: clip; }
.gallery-track { display: flex; gap: 4vw; width: max-content; }
.zoom-statement { overflow: clip; height: 150vh; display: flex; align-items: center; justify-content: center; }
```

Note the pattern of chaining a `pin: true` horizontal-scroll section directly after a scrubbed zoom statement — per `references/depth-system.md` §5, the gallery section needs an explicit higher `z-index` and opaque background so it visually overtakes the zoom section on pin release.

---

## Example 4 — SaaS Landing Page (Feature Grid + Window Pane Iris)

**Techniques applied:** Stagger grid · Window pane iris (hero-to-feature transition) · Variable font wave (headline) · Skew + elastic bounce (feature cards) · Section peel (pricing transition)

```html
<section class="saas-hero">
  <h1 class="wave-text">Ship faster. Sleep better.</h1>
  <div class="pane-grid" aria-hidden="true"><!-- 16 .pane divs generated via JS --></div>
</section>

<section class="feature-grid">
  <div class="grid-item skew-bounce">Real-time sync</div>
  <div class="grid-item skew-bounce">Zero-config deploys</div>
  <div class="grid-item skew-bounce">Built-in observability</div>
  <div class="grid-item skew-bounce">Team permissions</div>
</section>

<section class="pricing peel-section">…</section>
```

```js
// Window pane iris — hero closes to reveal feature grid beneath
document.querySelector('.pane-grid').innerHTML = Array.from({ length: 16 }, () => '<div class="pane"></div>').join('');
gsap.to('.pane', {
  scale: 0, duration: 0.8, ease: 'cubic-bezier(0.19,1,0.22,1)',
  stagger: { amount: 0.5, grid: [4, 4], from: 'center' },
  scrollTrigger: { trigger: '.saas-hero', start: 'top top', end: 'bottom top', scrub: true },
});

// Variable font wave headline
const waveChars = document.querySelector('.wave-text').textContent.split('').map((c) => {
  const span = document.createElement('span'); span.textContent = c; span.setAttribute('aria-hidden', 'true'); return span;
});
const h1 = document.querySelector('.wave-text');
h1.setAttribute('aria-label', h1.textContent);
h1.innerHTML = '';
waveChars.forEach((s) => h1.appendChild(s));
ScrollTrigger.create({
  trigger: '.wave-text', start: 'top 70%', end: 'bottom 30%',
  onUpdate: (self) => {
    const peak = Math.floor(self.progress * waveChars.length);
    waveChars.forEach((c, i) => c.style.fontVariationSettings = `'wght' ${Math.abs(i - peak) <= 1 ? 800 : 400}`);
  },
});

// Stagger grid + skew bounce feature cards
gsap.from('.grid-item', {
  y: -60, skewY: 8, opacity: 0, duration: 0.9,
  ease: 'cubic-bezier(0.68,-0.55,0.27,1.55)',
  stagger: { amount: 0.6, grid: 'auto', from: 'start' },
  scrollTrigger: { trigger: '.feature-grid', start: 'top 75%' },
});

// Pricing section peel-in
gsap.fromTo('.peel-section', { rotationX: -80, y: '20%', opacity: 0.4 }, {
  rotationX: 0, y: '0%', opacity: 1, ease: 'none',
  scrollTrigger: { trigger: '.peel-section', start: 'top bottom', end: 'top center', scrub: true },
});
```

```css
.saas-hero { position: relative; overflow: clip; }
.pane-grid { position: absolute; inset: 0; display: grid; grid-template-columns: repeat(4,1fr); grid-template-rows: repeat(4,1fr); z-index: 5; }
.pane { background: #0a0a0a; }
.wave-text span { display: inline-block; transition: font-variation-settings 0.3s; }
.feature-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 2rem; padding: 10vh 6vw; }
.grid-item { padding: 2rem; border-radius: 16px; background: #f5f5f5; }
.peel-section { transform-origin: bottom center; transform-style: preserve-3d; }
```

All four examples include the `prefers-reduced-motion` kill-switch from `references/accessibility.md` in production; omitted above for brevity where a single example already demonstrates the full pattern (Example 1).
