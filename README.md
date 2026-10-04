# Matrix-Rain-Generative-Art

A seed-based generative art catalogue for **Matrix Rain**, a computational digital-rain system. Built as a single-file HTML page with p5.js, styled as a gallery-ready portfolio and licensing surface.

**Live:** `https://reyrove.github.io/Matrix-Rain-Generative-Art/`

---

## Overview

**Matrix Rain** renders tiled digital rain from numeric seeds. Each composition picks one of six stream styles — vertical, diagonal, wave, concentric, binary, particle — and one of six terminal palettes — classic green, blue, cyan, purple, red alert, white/gray. Every lit cell draws its own stream column with a brightness gradient, and optional glitch effects (scanlines, flicker, horizontal bands, pixel noise) sit on top.

The page functions as a **catalogue and licensing surface** for fashion houses, textile studios, and surface designers. Every composition is reproducible by seed, meaning the same number always produces the identical stream.

---

## Features

### Generative Engine (p5.js)

- **100 × 100 tile grid** — 10,000 cells per composition, each lit or dark per the stream rule
- **Six stream styles**:
  - **Vertical** — classic `sin(y · 20 + i · 0.5)` falling columns
  - **Diagonal** — `sin((x + y) · 15 + i · 0.3)` diagonal drift
  - **Wave** — `sin(y · 25 + sin(x · 8) · 3)` horizontal modulation
  - **Concentric** — radial field with an angular term
  - **Binary** — hard thresholding to binary rain
  - **Particle** — interference of two sine products
- **Six terminal palettes** — Classic Green, Blue, Cyan, Purple, Red Alert, White/Gray
- **Head and tail effects** — brighter character head at `sin(y · 30 + i · 2) > 0.9`, softened tail via `cos(y · 15 − i)`
- **Intensity** — 0.3 – 0.8 density, drawn per seed
- **Glitch effects** — enabled at ~60% probability per seed, with horizontal bands, vertical bands, and pixel glitches
- **Scanlines** — subtle horizontal lines at 2% alpha
- **Digital overlay** — 50 katakana/Latin characters at 10% alpha, plus a cursor blinker
- **Flicker** — bright yellow flash at ~3% per cell

### Same Seed → Many Surfaces

A single seed drives every instance on the page:

| Instance | Purpose | Tiling |
|----------|---------|--------|
| **Cover** | Hero composition | 1 × 1 |
| **Plate** | Main work (master) | 1 × 1 |
| **Frame** | Framed archival print | 1 × 1 |
| **Print** | Archival print surface | 1 × 1 |
| **Scarf** | Silk scarf (3:1) | 3 × 1 |
| **Textile** | Fabric yardage (4:3) | 2 × 2 |
| **Wall** | Wallpaper (2:3) | 1 × 2 |
| **Archive** | 8 unique seeds | 1 × 1 each |

### Controls

- **New Seed** button / `R` key — regenerate with a fresh seed
- **Download** button / `S` key — export the plate canvas as PNG
- **Click any archive plate** — load that seed into the main view

### Performance

- Debounced resize handling (250ms)
- `IntersectionObserver` pauses off-screen sketches
- `visibilitychange` pauses rendering when the tab is hidden
- `pixelDensity(1)` for a lighter canvas footprint
- Two-frame render (`maxFrames: 2`) so tiles draw once and stop

---

## File Structure

```
Matrix-Rain-Generative-Art/
├── index.html          # Single-file page (structure + styles + generative engine)
├── README.md
└── images/
    ├── fav.svg         # Favicon
    ├── tote.png        # Mockup: tote bag
    ├── tee.png         # Mockup: t-shirt
    └── cushion.png     # Mockup: cushion
```

No build step. No dependencies to install. p5.js is loaded from a CDN.

---

## Getting Started

1. Clone the repo:
   ```bash
   git clone https://github.com/reyrove/Matrix-Rain-Generative-Art.git
   cd Matrix-Rain-Generative-Art
   ```
2. Add the three mockup images to `images/` (optional — they appear in the Commission section).
3. Open `index.html` in a browser.

That's it. The page self-initialises on `DOMContentLoaded`.

---

## How the Generative System Works

### Seeded Random

A linear congruential generator replaces p5's native randomness inside each sketch:

```js
class SeededRandom {
  constructor(s) { this.state = (Math.abs(s) % 2147483647) || 1; }
  rand() { this.state = (this.state * 9301 + 49297) % 233280; return this.state / 233280; }
  range(min, max) { return this.rand() * (max - min) + min; }
  int(min, max) { return Math.floor(this.range(min, max + 1)); }
  pick(arr) { return arr[Math.floor(this.rand() * arr.length)]; }
}
```

Every aesthetic decision — stream type, palette, intensity, glitch flag, per-cell brightness, per-cell character, glitch position — is drawn from this generator in a fixed order. Same seed, same sequence, same composition.

### Parameter Space (per seed)

| Parameter | Range |
|-----------|-------|
| Grid size | 100 × 100 |
| Stream style | 6 options (vertical → particle) |
| Palette | 6 options (green → gray) |
| Intensity | 0.3 – 0.8 |
| Glitch flag | ~60% probability |
| Flicker probability | 3% per cell |
| Character overlay | 50 characters at 10% alpha |

### The Six Stream Rules

Each style uses a different mathematical rule to compute the fall intensity at every cell. The threshold is `0.5 + (1 − intensity) · 0.5`, so higher intensity means lower threshold and more lit cells:

| Style | Rule |
|-------|------|
| **Vertical** | `sin(y · 20 + i · 0.5)` |
| **Diagonal** | `sin((x + y) · 15 + i · 0.3)` |
| **Wave** | `sin(y · 25 + sin(x · 8) · 3)` |
| **Concentric** | `sin(distance · 20 + angle · 5)` |
| **Binary** | `sin(y · 30 + x · 100) > 0.8` |
| **Particle** | `sin(x · 40) · cos(y · 25) + sin(x · 15) · cos(y · 20)` |

After the base rule, two modulation terms are applied:

```js
const headEffect = sin(y * 30 + i * 2) > 0.9 ? 1.5 : 1;   // bright head
const tailEffect = cos(y * 15 - i) * 0.3 + 0.7;            // softened tail
rainValue *= headEffect * tailEffect;
```

The head effect makes every ~30 rows a brighter "new character" that reads as the leading edge of the stream. The tail effect dims older cells slightly, so each column fades from head to tail.

### Glitch and Overlay

When `hasGlitch` is true, three kinds of glitch are drawn:

- **Horizontal bands** — 5 bands, each at 30–40% alpha, some bright yellow, some random hue
- **Vertical bands** — 3 bands, each at 30% alpha, in the background hue
- **Pixel glitches** — 20 single cells at 60% alpha, bright yellow or black

Then the digital overlay adds:

- **50 katakana + Latin characters** at 10% alpha, scattered at random
- **Grid diagonal ticks** at 5% alpha where `rand > 0.9`
- **A cursor blinker** at 80% alpha — one small yellow rectangle

### Tiling Logic

Rectangular surfaces (scarf, textile, wall) tile the square composition N×M. Each tile is drawn into its own translated coordinate space, centred within its cell:

```js
const tileW = canvasW / tileCols;
const tileH = canvasH / tileRows;
const tileSize = Math.min(tileW, tileH);
// draw square of side tileSize at (ox, oy) for each cell
```

This preserves the square composition's proportions regardless of surface aspect ratio.

---

## Customisation

### Change the default seed

Edit the init function:

```js
function init() {
  seed = 123456789;  // fixed seed instead of random
  // ...
}
```

### Adjust the grid resolution

Edit the constant near the top of the script:

```js
const GRID_SIZE = 100;
```

Higher values (150, 200) produce finer, denser rain with smaller cells. Lower values (60, 80) produce chunkier, more legible streams.

### Add a new stream style

1. Add the name to `STREAM_NAMES`:
   ```js
   const STREAM_NAMES = ['Vertical', ..., 'YourNewStyle'];
   ```
2. Add a `case` in the stream-generation switch inside `drawComposition`:
   ```js
   case 6: // YourNewStyle
     rainValue = /* your math */;
     break;
   ```
3. Add a render branch in the render loop if the style needs a different visual (e.g., if it draws a shape rather than a rectangle).
4. Update `sr.int(0, 5)` to `sr.int(0, 6)`.

### Add a new palette

Extend the `PALETTES` array with a new `{ bg: [h,s,b], fg: [[h,s,b], ...] }` entry, and extend `PALETTE_NAMES` with a matching name. The seed will automatically pick it up.

### Adjust the intensity range

```js
const intensity = sr.range(0.3, 0.8);
```

Lower values (e.g., `0.1, 0.4`) produce sparse rain. Higher values (e.g., `0.7, 0.95`) produce dense, almost full-canvas rain.

### Adjust the head effect frequency

```js
const headEffect = Math.sin(y * 30 + i * 2) > 0.9 ? 1.5 : 1;
```

Change the `30` multiplier to alter how often a bright head appears in the vertical rhythm. Change `0.9` to alter how many cells qualify as heads.

### Disable glitch entirely

```js
const hasGlitch = sr.rand() > 0.4;
```

Change to `const hasGlitch = false;` to disable glitches for all seeds.

### Change the digital overlay characters

```js
const DATA_CHARS = "01アイウエオカキクケコサシスセソタチツテトナニヌネノハヒフヘホマミムメモヤユヨラリルレロワヲン";
```

Replace with any string. Adding more Latin characters (or numbers) shifts the balance toward ASCII; removing katakana shifts it away from the classic Matrix aesthetic.

### Change surface tiling

Edit `surfaceDefs`:

```js
const surfaceDefs = [
  { id: 'surfacePrint',   wrapId: 'wrap-surfacePrint',   tile: { cols: 1, rows: 1 } },
  { id: 'surfaceScarf',   wrapId: 'wrap-surfaceScarf',   tile: { cols: 3, rows: 1 } },
  // ...
];
```

The `tile` object accepts any `{ cols, rows }` pair.

### Replace mockup images

Swap the files in `images/` or update the `src` attributes in the Commission section:

```html
<img src="images/your-mockup.png" alt="..." loading="lazy" />
```

---

## Design System

| Token | Value | Use |
|-------|-------|-----|
| `--bg` | `#0a0a0a` | Page background |
| `--paper` | `#f4f1ea` | Light surface (Surfaces section) |
| `--gold` | `#b8935a` | Primary accent |
| `--gold-soft` | `#d4b483` | Italic emphasis |
| `--serif` | Cormorant Garamond | Headings, titles |
| `--mono` | DM Mono | Labels, metadata, UI |

Type scale, spacing, and section rhythm match the Skyline, Portrait, Girih, Gothic Grid, Order to Chaos, Wallpaper Groups, Wandering Tiles, Food Grid, Framed Line, Deconstructed Stage, Layered Diagram, and Japanese Patterns templates. Responsive breakpoints at 900px, 720px, 560px, and 400px.

---

## Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `R` | Regenerate with a new seed |
| `S` | Download plate as PNG |
| `Enter` / `Space` | Activate focused archive item |
| `Esc` | Close legal modal |

---

## Browser Support

Modern evergreen browsers with:

- Canvas 2D (universal)
- `IntersectionObserver` (universal in modern browsers)
- `backdrop-filter` (Safari 9+, Chrome 76+, Firefox 103+)
- CSS custom properties (universal)

Fallback: if `IntersectionObserver` is unavailable, sketches render continuously.

---

## Accessibility

- Archive items are keyboard-focusable with `role="button"` and `aria-label`
- Legal modal traps focus, closes on `Esc` and overlay click
- `prefers-reduced-motion` disables transitions and smooth scroll
- All interactive elements have visible focus states (via browser default)

---

## Section Map

| Section | ID | Purpose |
|---------|-----|---------|
| Cover | — | Hero title + live rain |
| Statement | `#statement` | Artist statement |
| Plate | `#work` | Main work + metadata + controls |
| Surfaces | `#application` | Framed print + 4 tiled surfaces |
| Process | `#process` | Three process cards |
| Archive | `#archive` | 8 selectable seeds |
| Commission | `#commission` | Services + mockups + CTA |
| Colophon | — | Footer, links, legal |
| Legal modal | `#legalModal` | Licensing / Terms / Credits |

---

## Deployment (GitHub Pages)

1. Push the repo to GitHub.
2. Go to **Settings → Pages**.
3. Under **Source**, select `Deploy from a branch`.
4. Choose branch `main` and folder `/ (root)`.
5. Save — the site goes live at `https://reyrove.github.io/Matrix-Rain-Generative-Art/`.

---

## Notes on the Source Sketch

The generative engine is adapted almost verbatim from the original p5 sketch. The adaptations that turn it into a catalogue page:

1. **Seeded random replaces p5's `random()`** — a local `SeededRandom` class drives every choice. The original sketch calls `setSeeds()` for fxhash compatibility; that's replaced with the seeded generator directly.
2. **Sketch factory** — `makeSketch({ slotId, seedValue, ... })` returns a p5 sketch that can be instantiated into any DOM slot, enabling cover / plate / frame / surfaces / archive from one seed.
3. **DOM-driven canvas sizing** — `cs = min(windowHeight, windowWidth)` becomes slot-rect dimensions.
4. **Tiling support** — the `tile: { cols, rows }` parameter repeats the square composition across rectangular surfaces.
5. **Drawing logic takes `p` explicitly** — the original used p5's global mode; here `p` is passed so the factory closure works.
6. **`map(rainValue, 0.5, 1.5, 0.2, 1)` inlined** — replaced with `((rainValue - 0.5) / 1.0) * 0.8 + 0.2`, mathematically identical.
7. **`floor(random() * ...)` replaced with `sr.int(...)` or `Math.floor(sr.rand() * ...)`** — fully deterministic per seed.
8. **`windowResized` replaced by a debounced `handleResize`** — all sketches are destroyed and recreated on resize (250ms debounce), since their canvases are locked to slot dimensions.
9. **`triggerPreview` removed** — the original calls it for fxhash thumbnail generation; not needed in the plain-browser catalogue.

---

## Related Repositories

| Repo | Theme |
|------|-------|
| [`Skyline-Series`](https://github.com/reyrove/Skyline-Series) | Themed cityscapes |
| [`Portrait-Series-Generative-Art`](https://github.com/reyrove/Portrait-Series-Generative-Art) | Generative faces |
| [`Girih-2-Generative-Art`](https://github.com/reyrove/Girih-2-Generative-Art) | Islamic geometric ornament |
| [`Gothic-Grid-Generative-Art`](https://github.com/reyrove/Gothic-Grid-Generative-Art) | Mosaic grid composition |
| [`Order-to-Chaos-Generative-Art`](https://github.com/reyrove/Order-to-Chaos-Generative-Art) | Gradient grid composition |
| [`Wallpaper-Groups-Generative-Art`](https://github.com/reyrove/Wallpaper-Groups-Generative-Art) | Crystallographic symmetry |
| [`Wandering-Tiles-Generative-Art`](https://github.com/reyrove/Wandering-Tiles-Generative-Art) | Line-drawing composition |
| [`Food-Grid-Generative-Art`](https://github.com/reyrove/Food-Grid-Generative-Art) | Illustrated food grid |
| [`Framed-Line-Generative-Art`](https://github.com/reyrove/Framed-Line-Generative-Art) | Gesture line composition |
| [`Deconstructed-Stage-Generative-Art`](https://github.com/reyrove/Deconstructed-Stage-Generative-Art) | Suprematist composition |
| [`Layered-Diagram-Generative-Art`](https://github.com/reyrove/Layered-Diagram-Generative-Art) | Layered linework composition |
| [`Japanese-Patterns-Generative-Art`](https://github.com/reyrove/Japanese-Patterns-Generative-Art) | Traditional motif composition |
| **`Matrix-Rain-Generative-Art`** | Digital rain composition *(this repo)* |

All thirteen share the same catalogue template, design tokens, and seed-based reproducibility model.

---

## Credits

- **Design & Generative System** — Reyhaneh Daneshdoost
- **Typefaces** — Cormorant Garamond, DM Mono
- **Engine** — p5.js 1.9.0
- **Reference** — Classic digital rain terminal aesthetics
- **Edition** — Matrix Rain, Autumn 2026

---

## License

All artwork, code, and generative systems are the intellectual property of the artist. Compositions may not be reproduced, redistributed, resold, or adapted without a written license.

For licensing, commissions, or private systems: **reyhanehdaneshdoost@gmail.com**