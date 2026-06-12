# Cinematic Scroll Animation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the portfolio's one-shot IntersectionObserver reveals with a continuous GSAP ScrollTrigger-driven experience: scrubbed parallax (hero exit, ghost numerals, depth-staggered cards), a glowing SVG thread that draws down the whole page lighting nodes per section/job, and kinetic section headings.

**Architecture:** Everything stays in the single `index.html` (inline CSS + inline JS, current pattern). GSAP 3 + ScrollTrigger are vendored into `vendor/` (no CDN, GitHub Pages-safe). The thread is a body-spanning absolute SVG whose path is built programmatically from section positions and scrubbed via `stroke-dashoffset`. Three `gsap.matchMedia()` contexts: desktop full effects, mobile (≤900px) simplified, `prefers-reduced-motion` static.

**Tech Stack:** GSAP 3.13 + ScrollTrigger (vendored; GSAP Standard License — free for this use, not MIT), vanilla HTML/CSS/JS, no build step.

**Spec:** `docs/superpowers/specs/2026-06-12-scroll-animation-design.md`

**Worker model note:** Per user instruction, dispatch coding subagents on **Opus**. Tasks edit the same file — execute strictly sequentially, never in parallel.

**Verification environment:** Serve with `python3 -m http.server 8000` from repo root, view at `http://localhost:8000`. There is no test framework; each task ends with grep-based structural checks plus a browser sanity check where noted.

---

### Task 1: Vendor GSAP + load it

**Files:**
- Create: `vendor/gsap.min.js`, `vendor/ScrollTrigger.min.js`
- Modify: `index.html` (script tags before existing inline `<script>`)

- [ ] **Step 1: Download GSAP into vendor/**

```bash
mkdir -p vendor
curl -fsSL https://cdn.jsdelivr.net/npm/gsap@3.13.0/dist/gsap.min.js -o vendor/gsap.min.js
curl -fsSL https://cdn.jsdelivr.net/npm/gsap@3.13.0/dist/ScrollTrigger.min.js -o vendor/ScrollTrigger.min.js
```

- [ ] **Step 2: Verify downloads are real minified JS, not error pages**

```bash
head -c 200 vendor/gsap.min.js; echo; head -c 200 vendor/ScrollTrigger.min.js; echo; wc -c vendor/*.js
```
Expected: both start with a `/*!` GSAP license banner or minified JS; gsap.min.js ≈ 70–75KB, ScrollTrigger.min.js ≈ 40–45KB. If either is < 10KB or contains `<html`, the download failed — stop and retry.

- [ ] **Step 3: Add script tags to index.html**

In `index.html`, find the line `  <script>` (the opening tag of the existing inline script, just after the footer `</div>`). Insert immediately BEFORE it:

```html
  <script src="vendor/gsap.min.js"></script>
  <script src="vendor/ScrollTrigger.min.js"></script>
```

Then add as the FIRST line inside the existing inline `<script>` block (before `/* SCROLL PROGRESS */` comment block):

```js
    gsap.registerPlugin(ScrollTrigger);
```

- [ ] **Step 4: Verify page still works**

```bash
grep -n 'vendor/gsap.min.js\|vendor/ScrollTrigger.min.js\|registerPlugin' index.html
```
Expected: three matches in order (two script tags, then registerPlugin). Serve and load `http://localhost:8000` — page renders identically to before, no console errors (check: typewriter runs, sections reveal on scroll).

- [ ] **Step 5: Commit**

```bash
git add vendor/ index.html
git commit -m "Vendor GSAP 3.13 + ScrollTrigger and load on page"
```

---

### Task 2: Thread SVG, ghost numerals, structural HTML/CSS

**Files:**
- Modify: `index.html` (CSS block + HTML body)

- [ ] **Step 1: CSS — make body a positioning context**

In the `body { ... }` rule, add one declaration:

```css
      position: relative;
```

- [ ] **Step 2: CSS — rewrite the `.section` rule (remove one-shot fade, add stacking)**

Replace:

```css
    .section {
      padding: 72px 0;
      border-top: 1px solid var(--border);
      opacity: 0; transform: translateY(24px);
      transition: opacity 0.9s cubic-bezier(.16,1,.3,1), transform 0.9s cubic-bezier(.16,1,.3,1);
    }
    .section.visible { opacity: 1; transform: none; }
    .section-inner {
      max-width: 1100px; margin: 0 auto; padding: 0 48px;
    }
```

with:

```css
    .section {
      padding: 72px 0;
      border-top: 1px solid var(--border);
      position: relative; overflow: hidden;
    }
    .section-inner {
      max-width: 1100px; margin: 0 auto; padding: 0 48px;
      position: relative; z-index: 1;
    }
```

- [ ] **Step 3: CSS — add thread + ghost numeral styles**

Insert after the `/* ── Grain overlay ── */` block:

```css
    /* ── Thread ── */
    #thread-svg {
      position: absolute; top: 0; left: 0;
      z-index: 4; pointer-events: none; overflow: visible;
    }
    #thread-path {
      stroke: var(--accent); stroke-width: 2; fill: none;
      filter: drop-shadow(0 0 6px oklch(0.78 0.17 72 / 0.7));
    }
    .thread-node {
      fill: var(--bg); stroke: var(--border); stroke-width: 2;
      transition: stroke 0.4s, fill 0.4s, filter 0.4s;
    }
    .thread-node.lit {
      stroke: var(--accent); fill: oklch(0.3 0.08 72);
      filter: drop-shadow(0 0 6px var(--accent));
    }
    .thread-node-end.lit { filter: drop-shadow(0 0 16px var(--accent)); }

    /* ── Ghost numerals ── */
    .ghost-num {
      position: absolute; right: -1%; top: 8%;
      font-family: 'DM Serif Display', serif;
      font-size: clamp(8rem, 18vw, 15rem); line-height: 1;
      color: transparent; -webkit-text-stroke: 1px var(--border);
      opacity: 0.55; z-index: 0;
      pointer-events: none; user-select: none;
    }
    @media (max-width: 900px) { .ghost-num { display: none; } }
```

- [ ] **Step 4: HTML — add the thread SVG element**

Immediately after `<div id="progress-bar-track"><div id="progress-bar-fill"></div></div>`, insert:

```html
  <!-- Thread -->
  <svg id="thread-svg" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
    <path id="thread-path" d="" />
    <g id="thread-nodes"></g>
  </svg>
```

- [ ] **Step 5: HTML — add ghost numerals to the five content sections**

As the first child of each `<section>` (directly after the opening tag, before `<div class="section-inner">`):

- `#about`: `<div class="ghost-num" aria-hidden="true">01</div>`
- `#experience`: `<div class="ghost-num" aria-hidden="true">02</div>`
- `#projects`: `<div class="ghost-num" aria-hidden="true">03</div>`
- `#skills`: `<div class="ghost-num" aria-hidden="true">04</div>`
- `#contact`: `<div class="ghost-num" aria-hidden="true">05</div>`

- [ ] **Step 6: HTML — remove superseded elements**

1. Delete all four `<div class="exp-accent-line"></div>` lines (thread spine supersedes them).
2. Delete `<div class="scroll-cue-line"></div>` (thread is born here instead).
3. Change `<div class="reveal" id="contact-content">` to `<div id="contact-content">` (contact gets a custom timeline in Task 3, not the generic reveal).

- [ ] **Step 7: Verify**

```bash
grep -c 'ghost-num' index.html        # expect 6 (1 CSS + 5 HTML)
grep -c 'exp-accent-line' index.html  # expect 3 (CSS rules only — removed in Task 4)
grep -c 'thread-svg' index.html       # expect 2 (CSS + HTML)
grep -n 'class="reveal" id="contact-content"' index.html  # expect no match
```
Browser check: page loads, ghost numerals visible behind section content on desktop width, hidden below 900px; no horizontal scrollbar; existing reveal animations still function.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Add thread SVG, ghost numerals, and section stacking groundwork"
```

---

### Task 3: Replace animation system with GSAP ScrollTrigger

**Files:**
- Modify: `index.html` (CSS hidden-state rules + full inline `<script>` replacement)

- [ ] **Step 1: CSS — remove superseded hidden-state/transition rules**

GSAP now owns initial states via `fromTo`; leftover CSS transitions would fight scrubbed inline styles. Make these exact CSS edits:

1. `.hero-photo-wrap` — remove `opacity: 0; transform: translateY(24px);` and the whole `transition: ...;` line. Delete the rule `.hero-photo-wrap.loaded { opacity: 1; transform: translateY(0); }`. KEEP `.hero-photo-wrap.loaded .hero-photo-glow { animation: float 6s ease-in-out infinite; }`.
2. `.hero-photo-border` — remove `transform: scale(0.95);` and the `transition: ...;` line. Delete the rule `.hero-photo-wrap.loaded .hero-photo-border { transform: scale(1); }`.
3. Delete the `.scroll-cue-line { ... }` rule (element removed in Task 2).
4. Delete the entire `/* ── Reveal classes ── */` block (`.reveal`, `.reveal.from-left`, `.reveal.from-right`, `.reveal.visible`). The class names stay in HTML as JS selectors.
5. `.exp-item` — remove `opacity: 0; transform: translateX(-20px);` and the `transition: ...;` line (keep `display: grid`, columns, gap, padding, border-bottom, `position: relative`). Delete `.exp-item.visible { opacity: 1; transform: none; }`.
6. `.project-card-wrap` — remove `opacity: 0;` and the `transition: ...;` line (keep `height: 100%`). Delete the three rules: `.project-card-wrap:nth-child(odd) { transform: ... }`, `.project-card-wrap:nth-child(even) { transform: ... }`, `.project-card-wrap.visible { ... }`.
7. `.skill-category` — remove `opacity: 0; transition: opacity 0.5s;`. Delete the three `.skill-group[data-group-index="N"] .skill-category { transition-delay: ... }` rules and `.skill-group.visible .skill-category { opacity: 1; }`.
8. `.skill-tag` — remove `opacity: 0; transform: scale(0.9);` and remove `, opacity` concerns from its transition only if listed (`transition: all 0.2s ease;` can stay — it's for hover). Delete `.skill-group.visible .skill-tag { opacity: 1; transform: none; }`. (The inline `transition-delay` styles on tags in HTML become inert no-ops; leave them.)
9. Delete the unused `@keyframes slideUp` line. KEEP `blink`, `fadeInUp`, `scanline`, `float`, `slideWord`.
10. `.stats-grid-wrap` rules: KEEP unchanged (still class-toggled, one-shot).
11. `.section-label` rules: KEEP unchanged (still class-toggled, now reversible via ScrollTrigger).

- [ ] **Step 2: Replace the entire inline `<script>` body**

Replace everything between `<script>` and `</script>` (the inline block added to in Task 1 — NOT the two vendor `<script src=...>` tags) with:

```js
    gsap.registerPlugin(ScrollTrigger);

    const prefersReduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

    /* ═══ SCROLL PROGRESS ═══ */
    const fill = document.getElementById('progress-bar-fill');
    window.addEventListener('scroll', () => {
      const el = document.documentElement;
      const scrolled = el.scrollTop || document.body.scrollTop;
      const total = el.scrollHeight - el.clientHeight;
      fill.style.width = (total > 0 ? (scrolled / total) * 100 : 0) + '%';
    }, { passive: true });

    /* ═══ SECTION NAV ═══ */
    const sectionIds = ['hero', 'about', 'experience', 'projects', 'skills', 'contact'];
    const navLinks = document.querySelectorAll('.nav-dot-link');
    const sectionObs = new IntersectionObserver((entries) => {
      entries.forEach(e => {
        if (e.isIntersecting) {
          navLinks.forEach(l => l.classList.remove('active'));
          const active = document.querySelector(`.nav-dot-link[data-section="${e.target.id}"]`);
          if (active) active.classList.add('active');
        }
      });
    }, { threshold: 0.4 });
    sectionIds.forEach(id => {
      const el = document.getElementById(id);
      if (el) sectionObs.observe(el);
    });

    /* ═══ HERO LOAD ANIMATIONS ═══ */
    const typewriterEl = document.getElementById('typewriter-text');
    const cursorEl     = document.getElementById('typewriter-cursor');
    const typeText     = 'SOFTWARE ENGINEER · CS @ UF';

    if (prefersReduced) {
      typewriterEl.textContent = typeText;
      cursorEl.style.display = 'none';
      ['word-ignatius', 'word-martin'].forEach(id => {
        const el = document.getElementById(id);
        el.style.opacity = '1'; el.style.transform = 'none';
      });
      ['hero-bio', 'hero-ctas'].forEach(id => {
        const el = document.getElementById(id);
        el.style.opacity = '1'; el.style.transform = 'none';
      });
    } else {
      let typeIndex = 0;
      setTimeout(() => {
        const iv = setInterval(() => {
          typewriterEl.textContent = typeText.slice(0, typeIndex + 1);
          typeIndex++;
          if (typeIndex >= typeText.length) {
            clearInterval(iv);
            cursorEl.style.animation = 'none';
            cursorEl.style.opacity = '0';
          }
        }, 38);
      }, 300);

      const animateAfter = (el, delay) => setTimeout(() => el.classList.add('visible'), delay);
      animateAfter(document.getElementById('word-ignatius'), 600);
      animateAfter(document.getElementById('word-martin'), 820);
      animateAfter(document.getElementById('hero-bio'), 1100);
      animateAfter(document.getElementById('hero-ctas'), 1350);
    }

    /* ═══ HERO PHOTO ENTRANCE ═══ */
    const heroImg   = document.getElementById('hero-img');
    const photoWrap = document.getElementById('hero-photo-wrap');
    if (!prefersReduced) gsap.set(photoWrap, { autoAlpha: 0 });
    function onImgLoad() {
      photoWrap.classList.add('loaded'); // enables glow float
      if (prefersReduced) { gsap.set(photoWrap, { autoAlpha: 1 }); return; }
      gsap.fromTo(photoWrap,
        { autoAlpha: 0, y: 24 },
        { autoAlpha: 1, y: 0, duration: 1.2, delay: 0.9, ease: 'power3.out', clearProps: 'transform' });
      gsap.fromTo('.hero-photo-border',
        { scale: 0.95 },
        { scale: 1, duration: 1.2, delay: 0.9, ease: 'power3.out' });
    }
    if (heroImg.complete) onImgLoad();
    else heroImg.addEventListener('load', onImgLoad);

    /* ═══ COUNT-UP STATS ═══ */
    const statsWrap  = document.getElementById('stats-grid-wrap');
    const statValues = statsWrap.querySelectorAll('.stat-value[data-target]');
    function countUp(el, target, suffix, duration = 1200) {
      let start = null;
      function step(ts) {
        if (!start) start = ts;
        const p = Math.min((ts - start) / duration, 1);
        const ease = 1 - Math.pow(1 - p, 3);
        el.textContent = Math.round(ease * target) + suffix;
        if (p < 1) requestAnimationFrame(step);
      }
      requestAnimationFrame(step);
    }
    function runStats() {
      if (statsWrap.dataset.done) return;
      statsWrap.dataset.done = '1';
      statsWrap.classList.add('visible');
      statValues.forEach(el => countUp(el, parseInt(el.dataset.target), el.dataset.suffix || ''));
    }

    /* ═══ ONE-SHOT / CLASS-TOGGLED TRIGGERS (labels + stats) ═══ */
    function setupOneShots() {
      document.querySelectorAll('.section-label').forEach(el => {
        ScrollTrigger.create({
          trigger: el, start: 'top 90%',
          onEnter:     () => el.classList.add('visible'),
          onLeaveBack: () => el.classList.remove('visible')
        });
      });
      ScrollTrigger.create({ trigger: statsWrap, start: 'top 80%', once: true, onEnter: runStats });
    }

    /* ═══ THREAD ═══ */
    function buildThread() {
      const svg       = document.getElementById('thread-svg');
      const path      = document.getElementById('thread-path');
      const nodesWrap = document.getElementById('thread-nodes');
      const docH = document.documentElement.scrollHeight;
      const vw   = document.documentElement.clientWidth;
      svg.setAttribute('width', vw);
      svg.setAttribute('height', docH);
      svg.setAttribute('viewBox', `0 0 ${vw} ${docH}`);

      const innerRect = document.querySelector('#about .section-inner').getBoundingClientRect();
      const railX = innerRect.left + (vw > 900 ? 20 : 10);
      const pageY = el => el.getBoundingClientRect().top + window.scrollY;

      const cue    = document.querySelector('.scroll-cue');
      const startX = vw / 2;
      const startY = pageY(cue) + 10;
      const railTop = startY + 200;

      const btn  = document.querySelector('#contact .btn-primary');
      const endY = pageY(btn) + btn.offsetHeight / 2;

      let d = `M ${startX} ${startY}`;
      d += ` C ${startX} ${startY + 110}, ${railX} ${railTop - 110}, ${railX} ${railTop}`;
      d += ` L ${railX} ${endY}`;
      path.setAttribute('d', d);

      nodesWrap.innerHTML = '';
      const nodeYs = [];
      ['about', 'experience', 'projects', 'skills', 'contact'].forEach(id => {
        nodeYs.push({ y: pageY(document.getElementById(id)) + 70, big: false });
      });
      document.querySelectorAll('.exp-item').forEach(item => {
        nodeYs.push({ y: pageY(item) + 28, big: false });
      });
      nodeYs.push({ y: endY, big: true });

      const nodes = nodeYs.sort((a, b) => a.y - b.y).map(n => {
        const c = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
        c.setAttribute('cx', railX);
        c.setAttribute('cy', n.y);
        c.setAttribute('r', n.big ? 7 : 4.5);
        c.setAttribute('class', 'thread-node' + (n.big ? ' thread-node-end' : ''));
        nodesWrap.appendChild(c);
        return c;
      });

      const len = path.getTotalLength();
      path.style.strokeDasharray  = len;
      path.style.strokeDashoffset = len;
      return { path, len, nodes };
    }

    function enableThread() {
      const s = buildThread();
      const tween = gsap.to(s.path, {
        strokeDashoffset: 0, ease: 'none',
        scrollTrigger: {
          start: 0,
          end: () => document.documentElement.scrollHeight - window.innerHeight,
          scrub: 0.5,
          onUpdate(self) {
            const tipY = s.path.getPointAtLength(s.len * self.progress).y;
            s.nodes.forEach(n => n.classList.toggle('lit', +n.getAttribute('cy') <= tipY));
          }
        }
      });
      return () => { if (tween.scrollTrigger) tween.scrollTrigger.kill(); tween.kill(); };
    }

    function debounce(fn, ms) {
      let t; return () => { clearTimeout(t); t = setTimeout(fn, ms); };
    }

    /* ═══ RESPONSIVE CONTEXTS ═══ */
    const mm = gsap.matchMedia();
    const DESKTOP = '(min-width: 901px) and (prefers-reduced-motion: no-preference)';
    const MOBILE  = '(max-width: 900px) and (prefers-reduced-motion: no-preference)';
    const REDUCED = '(prefers-reduced-motion: reduce)';

    mm.add(DESKTOP, () => {
      setupOneShots();

      let teardownThread = enableThread();
      const rebuild = debounce(() => {
        teardownThread();
        teardownThread = enableThread();
        ScrollTrigger.refresh();
      }, 250);
      window.addEventListener('resize', rebuild);
      window.addEventListener('load', rebuild);

      // Hero cinematic exit
      gsap.timeline({
        scrollTrigger: { trigger: '#hero', start: 'top top', end: 'bottom top', scrub: true }
      })
        .to('.hero-inner',      { scale: 0.92, autoAlpha: 0.25, y: -60, ease: 'none' }, 0)
        .to('.hero-photo-wrap', { y: -40,       ease: 'none' }, 0)
        .to('.hero-grid-bg',    { yPercent: 18, ease: 'none' }, 0)
        .to('.hero-glow',       { yPercent: 30, ease: 'none' }, 0)
        .to('.scroll-cue',      { autoAlpha: 0, ease: 'none' }, 0);

      // Ghost numerals parallax
      gsap.utils.toArray('.ghost-num').forEach(el => {
        gsap.fromTo(el, { y: 120 }, {
          y: -120, ease: 'none',
          scrollTrigger: { trigger: el.closest('.section'), start: 'top bottom', end: 'bottom top', scrub: true }
        });
      });

      // Kinetic headings / generic reveals
      gsap.utils.toArray('.reveal').forEach(el => {
        const fromX = el.classList.contains('from-right') ? 60
                    : el.classList.contains('from-left')  ? -60 : 0;
        gsap.fromTo(el,
          { autoAlpha: 0, x: fromX, y: fromX ? 0 : 40 },
          { autoAlpha: 1, x: 0, y: 0, ease: 'none',
            scrollTrigger: { trigger: el, start: 'top 88%', end: 'top 55%', scrub: true } });
      });

      // Stats grid drifts slightly slower than text column
      gsap.fromTo(statsWrap, { y: 60 }, {
        y: 0, ease: 'none',
        scrollTrigger: { trigger: '#about', start: 'top 80%', end: 'center center', scrub: true }
      });
      statsWrap.classList.add('visible'); // GSAP owns motion; class keeps CSS opacity rule satisfied

      // Experience items — alternate sides
      gsap.utils.toArray('.exp-item').forEach((el, i) => {
        gsap.fromTo(el,
          { autoAlpha: 0, x: i % 2 ? 60 : -60 },
          { autoAlpha: 1, x: 0, ease: 'none',
            scrollTrigger: { trigger: el, start: 'top 92%', end: 'top 65%', scrub: true } });
      });

      // Project cards — column speed differential
      gsap.utils.toArray('.project-card-wrap').forEach((el, i) => {
        const rightCol = i % 2 === 1;
        gsap.fromTo(el,
          { autoAlpha: 0, x: rightCol ? 40 : -40, y: rightCol ? 80 : 0 },
          { autoAlpha: 1, x: 0, y: 0, ease: 'none',
            scrollTrigger: { trigger: el, start: 'top 95%', end: 'top 60%', scrub: true } });
      });

      // Skills — scrub-linked cascade
      gsap.utils.toArray('.skill-group').forEach(group => {
        const cat  = group.querySelector('.skill-category');
        const tags = group.querySelectorAll('.skill-tag');
        gsap.timeline({
          scrollTrigger: { trigger: group, start: 'top 90%', end: 'top 50%', scrub: true }
        })
          .fromTo(cat,  { autoAlpha: 0, x: -12 }, { autoAlpha: 1, x: 0 }, 0)
          .fromTo(tags, { autoAlpha: 0, scale: 0.85, y: 12 },
                        { autoAlpha: 1, scale: 1, y: 0, stagger: 0.06 }, 0);
      });

      // Contact finale
      gsap.timeline({
        scrollTrigger: { trigger: '#contact-content', start: 'top 90%', end: 'top 50%', scrub: true }
      })
        .fromTo('.contact-heading',      { autoAlpha: 0, y: 70 }, { autoAlpha: 1, y: 0 }, 0)
        .fromTo('.contact-body',         { autoAlpha: 0, y: 40 }, { autoAlpha: 1, y: 0 }, 0.1)
        .fromTo('#contact .btn-primary', { autoAlpha: 0, y: 24 }, { autoAlpha: 1, y: 0 }, 0.25)
        .fromTo('.contact-link',         { autoAlpha: 0, y: 16 },
                                         { autoAlpha: 1, y: 0, stagger: 0.08 }, 0.4);

      return () => {
        window.removeEventListener('resize', rebuild);
        window.removeEventListener('load', rebuild);
        teardownThread();
      };
    });

    mm.add(MOBILE, () => {
      setupOneShots();

      let teardownThread = enableThread();
      const rebuild = debounce(() => {
        teardownThread();
        teardownThread = enableThread();
        ScrollTrigger.refresh();
      }, 250);
      window.addEventListener('resize', rebuild);
      window.addEventListener('load', rebuild);

      // Simple one-shot fades — no scrub, no parallax
      gsap.utils.toArray('.reveal, .exp-item, .project-card-wrap, .skill-group, #contact-content')
        .forEach(el => {
          gsap.fromTo(el,
            { autoAlpha: 0, y: 24 },
            { autoAlpha: 1, y: 0, duration: 0.7, ease: 'power2.out',
              scrollTrigger: { trigger: el, start: 'top 90%' } });
        });
      statsWrap.classList.add('visible');

      return () => {
        window.removeEventListener('resize', rebuild);
        window.removeEventListener('load', rebuild);
        teardownThread();
      };
    });

    mm.add(REDUCED, () => {
      // Static page: thread fully drawn, all nodes lit, everything visible, final stat values.
      const s = buildThread();
      s.path.style.strokeDashoffset = 0;
      s.nodes.forEach(n => n.classList.add('lit'));
      document.querySelectorAll('.section-label').forEach(el => el.classList.add('visible'));
      statsWrap.classList.add('visible');
      statValues.forEach(el => { el.textContent = el.dataset.target + (el.dataset.suffix || ''); });
    });
```

- [ ] **Step 3: Structural verification**

```bash
grep -c 'matchMedia\|mm.add' index.html       # expect >= 4
grep -n 'makeObs\|revealObs\|sectionRevealObs' index.html  # expect no matches
grep -c 'buildThread\|enableThread' index.html # expect >= 4
grep -n '@keyframes slideUp' index.html        # expect no match
grep -n 'class="reveal' index.html             # expect 4 (about-text, exp-heading, projects-heading, skills-heading)
```

- [ ] **Step 4: Browser verification (desktop width)**

Serve and check at `http://localhost:8000`:
1. No console errors.
2. Thread line draws downward as you scroll; nodes light as the tip passes; scrolling up reverses both.
3. Hero shrinks/fades on scroll-away and restores on return; photo/grid/glow move at visibly different rates.
4. Ghost numerals drift slower than scroll behind each section.
5. Experience items slide in from alternating sides, scrubbed (reverse on scroll-up).
6. Right project card column starts lower and catches up.
7. Skill tags cascade in tied to scroll.
8. Contact: heading, body, button, links arrive in sequence; final big node glows at "Say hello".
9. Stats still count up once; typewriter and name entrance unchanged.

- [ ] **Step 5: Browser verification (mobile + reduced motion)**

1. DevTools responsive mode ≤900px wide, reload: ghost numerals hidden, simple fades only, thread still draws.
2. DevTools → Rendering → Emulate `prefers-reduced-motion: reduce`, reload: page fully visible immediately, thread fully drawn, all nodes lit, stats show final values, full typewriter text.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Replace IntersectionObserver reveals with GSAP ScrollTrigger system

Scrubbed hero exit, ghost numeral parallax, thread draw with section/job
nodes, alternating experience slides, column-staggered project cards,
skill cascade, contact finale. matchMedia contexts for mobile and
prefers-reduced-motion."
```

---

### Task 4: Dead CSS prune + overflow audit

**Files:**
- Modify: `index.html` (CSS block only)

- [ ] **Step 1: Remove dead CSS**

1. Delete the `.exp-accent-line { ... }` rule and the `.exp-item.visible .exp-accent-line { height: 100%; }` rule (elements removed in Task 2).
2. Confirm nothing else references removed classes:

```bash
grep -n 'exp-accent-line\|scroll-cue-line\|slideUp' index.html  # expect no matches
```

- [ ] **Step 2: Overflow + layout audit in browser**

1. Desktop: scroll full page — no horizontal scrollbar at any point (ghost numerals must clip at section edges).
2. Resize window across the 900px boundary in both directions — thread rebuilds to correct rail position, no duplicate paths/nodes, no console errors.
3. Click each nav dot — smooth anchor scrolling still works; progress bar fills correctly.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Prune CSS superseded by GSAP animation system"
```

---

### Task 5: Final verification pass

**Files:** none (verification only; fixes go in `index.html` if found)

- [ ] **Step 1: Serve and run the full checklist**

```bash
python3 -m http.server 8000
```

Full pass of Task 3 Steps 4–5 checklist plus Task 4 Step 2, end to end, after a hard reload (Cmd+Shift+R).

- [ ] **Step 2: GitHub Pages compatibility check**

```bash
grep -n 'src="vendor/' index.html   # relative paths only — expect 2 matches, no leading slash
grep -n 'http://' index.html         # expect no insecure/external script references
```

- [ ] **Step 3: Fix anything found, commit fixes**

Any defect: fix in `index.html`, re-verify the specific checklist item, then:

```bash
git add index.html && git commit -m "Fix scroll animation defects found in final verification"
```
