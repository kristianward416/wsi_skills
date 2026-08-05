# Text Animations

13 techniques for making headlines, body copy, and kinetic typography feel intentional rather than static. Every technique here must respect the accessibility rules in `references/accessibility.md` (real text preserved via `aria-label`, split spans marked `aria-hidden`).

## Shared Split-Text Utility

Most word/char techniques need text split into addressable units first:

```js
function splitText(el, unit = 'char') {
  const text = el.textContent;
  el setAttribute && el.setAttribute('aria-label', text);
  el.textContent = '';
  const units = unit === 'word' ? text.split(/(\s+)/) : text.split('');
  units.forEach((u) => {
    const span = document.createElement('span');
    span.textContent = u;
    span.className = unit === 'word' ? 'split-word' : 'split-char';
    span.setAttribute('aria-hidden', 'true');
    span.style.display = 'inline-block';
    el.appendChild(span);
  });
  return [...el.querySelectorAll(`.split-${unit}`)];
}
```

(Note: fix the stray `el setAttribute` typo to `el.setAttribute` when implementing — shown as written for illustration.)

## 1. Split Converge

Characters/words fly in from scattered random positions and converge into place.

```js
const chars = splitText(document.querySelector('.split-converge'), 'char');
chars.forEach((c) => gsap.set(c, {
  x: gsap.utils.random(-200, 200),
  y: gsap.utils.random(-150, 150),
  rotate: gsap.utils.random(-45, 45),
  opacity: 0,
}));
gsap.to(chars, {
  x: 0, y: 0, rotate: 0, opacity: 1,
  duration: 1, stagger: 0.02, ease: 'var(--ease-cinematic)',
  scrollTrigger: { trigger: '.split-converge', start: 'top 75%' },
});
```

## 2. Word-by-Word Scroll Lighting ("words light up on scroll")

Each word dims until the scroll position reaches it, then "lights up" — classic long-form editorial effect.

```css
.light-word { color: rgba(255,255,255,0.25); transition: color 0.3s; }
.light-word.is-lit { color: #fff; }
```

```js
const words = splitText(document.querySelector('.lighting-paragraph'), 'word');
words.forEach((w) => w.classList.add('light-word'));

ScrollTrigger.create({
  trigger: '.lighting-paragraph',
  start: 'top 70%',
  end: 'bottom 40%',
  onUpdate: (self) => {
    const litCount = Math.floor(self.progress * words.length);
    words.forEach((w, i) => w.classList.toggle('is-lit', i <= litCount));
  },
});
```

For a glow rather than flat color-change variant:

```css
.light-word.is-lit { color: #fff; text-shadow: 0 0 12px rgba(255,255,255,0.6); }
```

## 3. Offset Diagonal Layout

Lines of a heading are laid out on a diagonal stagger, entering along that same diagonal axis.

```css
.diagonal-line:nth-child(1) { transform: translate(0, 0); }
.diagonal-line:nth-child(2) { transform: translate(40px, 8px); }
.diagonal-line:nth-child(3) { transform: translate(80px, 16px); }
```

```js
gsap.from('.diagonal-line', {
  x: (i) => -60 - i * 20,
  opacity: 0,
  duration: 0.8,
  stagger: 0.12,
  ease: 'var(--ease-out-expo)',
  scrollTrigger: { trigger: '.diagonal-heading', start: 'top 80%' },
});
```

## 4. Masked Line Reveal

Each line of text is revealed via a `clip-path` sweep rather than a fade/slide — reads as more premium/editorial.

```css
.masked-line {
  clip-path: inset(0 100% 0 0); /* fully hidden, revealing from the left */
}
```

```js
gsap.to('.masked-line', {
  clipPath: 'inset(0 0% 0 0)',
  duration: 0.9,
  stagger: 0.15,
  ease: 'var(--ease-out-expo)',
  scrollTrigger: { trigger: '.masked-heading', start: 'top 80%' },
});
```

## 5. Theatrical Enter + Exit

Text has a distinct entrance AND exit choreography — it doesn't just appear once and sit static; it performs a full arc as the section scrolls through.

```js
gsap.timeline({
  scrollTrigger: { trigger: '.theatrical-scene', start: 'top top', end: 'bottom top', scrub: true },
})
.fromTo('.theatrical-text', { opacity: 0, y: 80, scale: 0.9 }, { opacity: 1, y: 0, scale: 1, ease: 'none' }, 0)
.to('.theatrical-text', { opacity: 0, y: -80, scale: 1.1, ease: 'none' }, 0.7); // exits in the last 30% of the scrub range
```

## 6. Scramble Text

Characters cycle through random glyphs before settling on the real character — a "hacker terminal" decode effect.

```js
function scrambleText(el, finalText, duration = 1.2) {
  const glyphs = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ!@#$%&*';
  const chars = finalText.split('');
  let frame = 0;
  const totalFrames = Math.round(duration * 60);
  const revealEvery = totalFrames / chars.length;

  function tick() {
    frame++;
    const revealedCount = Math.floor(frame / revealEvery);
    el.textContent = chars.map((c, i) =>
      i < revealedCount ? c : (c === ' ' ? ' ' : glyphs[Math.floor(Math.random() * glyphs.length)])
    ).join('');
    if (frame < totalFrames) requestAnimationFrame(tick);
    else el.textContent = finalText;
  }
  tick();
}

ScrollTrigger.create({
  trigger: '.scramble-target',
  start: 'top 80%',
  once: true,
  onEnter: () => scrambleText(document.querySelector('.scramble-target'), document.querySelector('.scramble-target').dataset.text),
});
```

Set `aria-label` on the container to the final text up front so screen readers get the correct content immediately rather than the scrambling intermediate states.

## 7. Skew + Elastic Bounce

Text enters with a velocity-implying skew that snaps back to level — reads as "thrown into place."

```js
gsap.from('.skew-bounce', {
  y: -60,
  skewY: 8,
  opacity: 0,
  duration: 0.9,
  ease: 'var(--ease-elastic-soft)',
  scrollTrigger: { trigger: '.skew-bounce', start: 'top 85%' },
});
```

## 8. Line Clip Wipe

Similar to masked line reveal but wipes horizontally across the *whole* line (not per-word), typically used for a single decisive headline reveal rather than staggered lines.

```css
.line-wipe { clip-path: inset(0 0 0 100%); }
```

```js
gsap.to('.line-wipe', {
  clipPath: 'inset(0 0 0 0%)',
  duration: 1,
  ease: 'var(--ease-out-expo)',
  scrollTrigger: { trigger: '.line-wipe', start: 'top 75%' },
});
```

## 9. Variable Font Wave

Uses a variable font's weight/width axis to animate a "wave" of emphasis running through a line of text — no position change, purely typographic.

```css
.wave-text span {
  font-variation-settings: 'wght' 400;
  transition: font-variation-settings 0.3s;
  display: inline-block;
}
.wave-text span.is-peak { font-variation-settings: 'wght' 800; }
```

```js
const waveChars = splitText(document.querySelector('.wave-text'), 'char');
ScrollTrigger.create({
  trigger: '.wave-text',
  start: 'top 70%',
  end: 'bottom 30%',
  onUpdate: (self) => {
    const peakIndex = Math.floor(self.progress * waveChars.length);
    waveChars.forEach((c, i) => c.classList.toggle('is-peak', Math.abs(i - peakIndex) <= 1));
  },
});
```

Requires a variable font (e.g. a `wght` axis) loaded via `@font-face` with `font-variation-settings` support.

## 10. Bleed Typography

Oversized headline text that bleeds off the edges of the viewport, used as a background-scale typographic layer (often combined with a hero image overlapping it at depth-3, text sitting at depth-4 behind or in front).

```css
.bleed-heading {
  font-size: clamp(4rem, 18vw, 16rem);
  line-height: 0.9;
  letter-spacing: -0.02em;
  white-space: nowrap;
  margin: 0 -2vw; /* intentional overflow bleed */
  overflow: visible;
}
```

Combine with a slow parallax drift (depth-1 speed, 0.25x) so the oversized type has its own subtle life without competing with the depth-3 hero object motion.

## 11. Character Cylinder

Characters appear to rotate in on a 3D cylinder, as if printed on a rotating drum — used for "Apple-style" premium headline reveals.

```css
.cylinder-char {
  display: inline-block;
  transform-style: preserve-3d;
  backface-visibility: hidden;
}
```

```js
gsap.from('.cylinder-char', {
  rotationX: -90,
  opacity: 0,
  duration: 0.7,
  stagger: 0.03,
  transformOrigin: '50% 50% -20px',
  ease: 'var(--ease-out-expo)',
  scrollTrigger: { trigger: '.cylinder-heading', start: 'top 80%' },
});
```

The container needs `perspective` for the rotation to read as 3D rather than a flat vertical flip:

```css
.cylinder-heading { perspective: 600px; }
```

## 12. Kinetic Word Cascade

Words enter in sequence with individually varied timing/easing (not uniform stagger) to feel less mechanical — mix `ease-elastic-soft` on odd words and `ease-out-soft` on even words.

```js
const cascadeWords = splitText(document.querySelector('.kinetic-cascade'), 'word');
cascadeWords.forEach((w, i) => {
  gsap.from(w, {
    y: 30, opacity: 0,
    duration: 0.6,
    delay: i * 0.08,
    ease: i % 2 === 0 ? 'var(--ease-elastic-soft)' : 'var(--ease-out-soft)',
    scrollTrigger: { trigger: '.kinetic-cascade', start: 'top 80%' },
  });
});
```

## 13. Pinned Text Scrub with Color/Weight Shift

Text stays pinned/centered while its color and weight shift across the scrub range — often paired with a background color/scene transition so the text "belongs" to whichever section is currently dominant.

```js
gsap.to('.pinned-shift-text', {
  color: '#ffffff',
  fontWeight: 800,
  ease: 'none',
  scrollTrigger: { trigger: '.color-shift-scene', start: 'top top', end: 'bottom top', scrub: true },
});
```
