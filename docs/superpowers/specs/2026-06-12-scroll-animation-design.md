# Scroll Animation Design — Portfolio Website

**Date:** 2026-06-12
**Status:** Approved by user (visual storyboard reviewed in brainstorming companion)

## Goal

Transform the portfolio (`index.html`) from one-shot IntersectionObserver reveals into a continuous, scroll-driven cinematic experience spanning the entire page. Three combined motifs, chosen by the user from four prototyped directions:

- **A — Cinematic Scrub:** animations tied directly to scroll position, reversible when scrolling back up; parallax depth between layers.
- **C — Flowing Thread:** a glowing amber SVG line that draws itself down the full page, connecting sections; the signature element.
- **D — Kinetic Typography:** large ghost numerals (01–05) parallax-drifting behind sections; section headings slide in horizontally with scrub.

Rejected: B — Pinned Chapters (scroll-jacking page-turn sections).

## Engine

**GSAP 3 + ScrollTrigger, self-hosted.** Minified `gsap.min.js` and `ScrollTrigger.min.js` committed to the repo under `vendor/` and loaded via plain `<script>` tags before the closing `</body>`. No CDN dependency, no build step — fully compatible with GitHub Pages static hosting.

## Architecture

Site remains a single `index.html` (current pattern: inline CSS + inline JS). Changes:

1. **New SVG thread overlay** — a full-height `<svg>` positioned over the page content (left rail on desktop), `pointer-events: none`. Path drawn via `stroke-dasharray`/`stroke-dashoffset` scrubbed by a ScrollTrigger spanning the whole document. Node circles (one per section + one per experience item) light up as the thread's drawn tip passes them.
2. **Ghost numerals** — one absolutely-positioned outlined numeral (`01`–`05`) per content section (About, Experience, Projects, Skills, Contact), behind content (`z-index` below text), parallax-scrubbed at a slower rate than scroll.
3. **GSAP timeline blocks** replace the existing one-shot IntersectionObserver reveals for: section headings (slide from side, scrubbed), experience items (alternate sides), project cards (column-speed differential), skill tag cascade (scrub-linked stagger).
4. **Hero exit scrub** — a ScrollTrigger over the hero: name scales down (~0.9) and fades, photo translates up slower than scroll (depth), grid background slower still, glow drifts. Reverses on scroll-up.
5. **Contact finale** — thread terminates in a glowing node/burst beside the "Say hello" button; heading scrubs up; links fade last.

### Kept unchanged

- Typewriter hero tag, staggered name-word entrance (load-time, not scroll)
- Scroll progress bar, right-rail section nav dots
- All hover effects (project cards, skill tags, buttons)
- Count-up stats (fires once on enter, as today)
- Grain overlay, color system, fonts, layout, content

### Removed/replaced

- The generic `.reveal` / `.section.visible` IntersectionObserver one-shot system is replaced by ScrollTrigger equivalents (scrubbed or `toggleActions` where one-shot still appropriate). Existing CSS transition classes pruned where superseded.

## Section-by-section spec

| Section | Effects |
|---|---|
| Hero | Exit scrub: name scale 1→0.9 + fade, photo parallax (slower), grid bg (slowest), glow drift. Thread born at scroll cue position. |
| About | Ghost `01`; heading slides from left (scrub); stats grid drifts subtly slower than text column; thread node at entry. |
| Experience | Ghost `02`; thread acts as timeline spine through the four jobs, node per job lights as thread tip reaches it; items slide in from alternating sides. The existing per-item `.exp-accent-line` is superseded by the thread spine. |
| Projects | Ghost `03`; heading scrub-slide; left card column scrolls slightly faster than right, drifting into alignment mid-viewport. |
| Skills | Ghost `04`; heading slides from left; skill tags cascade as scrub-linked wave instead of one-shot stagger. |
| Contact | Ghost `05`; heading scrubs up; thread finale burst node at "Say hello"; links fade in last. |

## Mobile & accessibility

- **Mobile/touch (≤900px, matching existing breakpoint):** simplified treatment — thread line + simple fade reveals remain; parallax, scrub-reversal, and column-speed effects disabled (`ScrollTrigger.matchMedia`). Ghost numerals hidden or static.
- **`prefers-reduced-motion: reduce`:** all scroll animation disabled; content appears instantly (no motion), thread rendered fully drawn.
- Performance: transforms/opacity only (no layout-triggering properties); `will-change` used sparingly; single document-spanning ScrollTrigger for the thread rather than per-pixel listeners.

## Testing / verification

- Manual: serve locally, verify each section's animation forward and reverse, at multiple viewport sizes (desktop, 900px boundary, mobile).
- Verify reduced-motion via OS setting or DevTools emulation.
- Verify no horizontal overflow introduced (ghost numerals clipped via `overflow: hidden` on sections).
- Verify GitHub Pages compatibility: no external requests beyond Google Fonts (existing); GSAP served from repo.

## Implementation notes

- Planning (this doc, implementation plan): Fable. Coding tasks: dispatched to Opus subagents per user instruction.
- Single-file edit surface (`index.html`) means coding tasks must be sequenced or carefully partitioned (CSS block / HTML additions / JS block) to avoid edit conflicts.
