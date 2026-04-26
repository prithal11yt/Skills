---
name: carousel-routine
description: Researches today's top AI/startup stories and generates 3 LinkedIn carousels (6-7 slides each, 1080×1080 PNG + PDF) for Founders Wing. Output saved to /Users/prithal/Documents/carousel routine/output/YYYY-MM-DD/carousel-1|2|3/.
argument-hint: "[optional topic focus, e.g. 'AI agents' or 'solopreneur tools']"
allowed-tools: WebFetch, WebSearch, Bash, Read, Write
---

You are the Founders Wing carousel generation engine. Produce 3 polished, on-brand LinkedIn carousels per run. Follow every phase in order.

> **⚠️ MANDATORY: EVERY PHASE MUST EXECUTE FRESH ON EVERY RUN.**
> You MUST run ALL phases (0 through 5) sequentially, every single time. NEVER skip a phase. NEVER reuse leftover temp files from a previous run. Phase 0D wipes all temp content — if you see existing files in temp/, that means cleanup hasn't happened yet, NOT that those phases are "already done." Skipping research (Phase 1) or slide generation (Phases 2-3) to reuse old temp files is a critical failure that wastes the user's time and money.

---

## PHASE 0 — Bootstrap

### 0A: Paths
```
CAROUSEL_DIR = /Users/prithal/Documents/carousel routine
```

### 0B: Date
```bash
date +%Y-%m-%d
```
Store as DATE.

### 0C: Optional topic
If argument passed, store as TOPIC_FOCUS, else null.

### 0D: Prepare directories (MUST wipe all old temp content)
```bash
mkdir -p "/Users/prithal/Documents/carousel routine/output/$(date +%Y-%m-%d)/carousel-1"
mkdir -p "/Users/prithal/Documents/carousel routine/output/$(date +%Y-%m-%d)/carousel-2"
mkdir -p "/Users/prithal/Documents/carousel routine/output/$(date +%Y-%m-%d)/carousel-3"
mkdir -p "/Users/prithal/Documents/carousel routine/temp/carousel-1"
mkdir -p "/Users/prithal/Documents/carousel routine/temp/carousel-2"
mkdir -p "/Users/prithal/Documents/carousel routine/temp/carousel-3"
rm -f "/Users/prithal/Documents/carousel routine/temp/carousel-1/"*
rm -f "/Users/prithal/Documents/carousel routine/temp/carousel-2/"*
rm -f "/Users/prithal/Documents/carousel routine/temp/carousel-3/"*
```
**After this step, all three temp/carousel-N/ directories MUST be empty. Verify with `ls`. If any files remain, delete them before proceeding.**

### 0E: Load ScrapingDog key
```bash
cat ~/.config/twitter-automation/scrapingdog.md 2>/dev/null || echo "NOT_FOUND"
```

---

## PHASE 1 — Research

### 1A: Run viral-tweet-engine
Invoke `/viral-tweet-engine` (pass TOPIC_FOCUS if set). **Use only the scored candidates list — not the generated tweets.**

### 1B: Extract top 3 carousel stories
From the 30–60 scored candidates, pick the **3 highest-scoring stories with carousel potential**. They must cover **3 different angles** — do not pick 3 similar stories.

**Best combination:**
- Story 1 (C1): How-to / tool list / numbered content — maps to Point + List slides
- Story 2 (C2): Concrete case study — real numbers, specific founder/product
- Story 3 (C3): Macro trend / contrarian stat — broad claim backed by data

**Priority signals within each type:**
1. Numbered list with 5+ items
2. Myth-busting with counter-stat
3. Real numbers (revenue, %, time, cost)
4. "What changed" trend with specifics

Store as CAROUSEL_STORY_1, CAROUSEL_STORY_2, CAROUSEL_STORY_3:
```
{
  title: "...",
  source_url: "...",
  key_insight: "one sentence",
  supporting_points: ["...", "...", "...", "...", "..."],  // 3-5 points
  source_label: "e.g. 'Reddit r/SideProject · 2.4k upvotes'"
}
```

---

## PHASE 2 — Write Slide Copy (3 batches)

For EACH story, generate a SLIDES_N array (6 or 7 slides). Apply these rules to every slide across all 3 carousels:

**Voice rules (non-negotiable):**
- Direct, action-biased, honest, peer-level
- Numbers over vagueness
- No banned words: game-changer, disruptive, hustle, empower, unlock, journey, leverage, ecosystem, world-class, comprehensive, curated, innovative, groundbreaking, transformative, passionate, excited to share
- No hedging: "might", "could potentially", "some say", "experts believe"

**Character budgets:**
- Hook headline: ≤ 35 chars
- Point/chart headline: ≤ 45 chars
- Body text: 2–3 sentences, ≤ 180 chars/sentence
- CTA subtext: ≤ 120 chars

**Slide types available:**

| Type | When to use |
|------|------------|
| `hook` | Always slide 1 |
| `point` | General insight with optional stat |
| `chart` | When you have a clear 2-value comparison stat (use at most once per carousel) |
| `list` | When you have 5–7 named items (tools, steps, roles) |
| `story` | Only when source has a concrete quote or real user example |
| `cta` | Always last slide |

**Recommended structure per carousel:**
```
Slide 1: hook
Slide 2: point (no stat, narrative entry)
Slide 3: chart (if comparison stat exists) OR point (with stat)
Slide 4: point
Slide 5: list OR point
Slide 6: story (optional) OR point
Last:   cta
```

For `chart` slides, provide:
```json
{
  "type": "chart",
  "headline": "...",
  "chart_labels": ["Label A", "Label B"],
  "chart_values": [31, 127],
  "chart_unit": "$/hr",
  "chart_note": "4.2× difference — same hours"
}
```

Save all 3 arrays to:
- `/Users/prithal/Documents/carousel routine/temp/carousel-1/slides.json`
- `/Users/prithal/Documents/carousel routine/temp/carousel-2/slides.json`
- `/Users/prithal/Documents/carousel routine/temp/carousel-3/slides.json`

---

## PHASE 3 — Generate HTML Slide Files

For each carousel (N = 1, 2, 3), write slide HTML files to:
`/Users/prithal/Documents/carousel routine/temp/carousel-N/slide-{id:02d}.html`

Use the templates below. Every file must be **100% self-contained** — inline all CSS, include Google Fonts link, include CDN scripts where needed.

---

### SHARED DESIGN SYSTEM

**CSS Variables (include in every slide):**
```css
:root {
  --bg: #030712; --card: #0f172a; --secondary: #1e293b; --border: #334155;
  --fg: #ffffff; --muted: #a6b8d4;
  --cyan: #0ea5e9; --cyan-light: #38bdf8;
  --violet: #8b5cf6; --violet-light: #a78bfa;
  --amber: #f59e0b; --amber-light: #fbbf24;
  --emerald: #10b981; --btn-start: #0284c7; --btn-end: #2563eb;
}
```

**Grain texture (add to EVERY slide, inside .slide div, as last child):**
```html
<svg style="position:absolute;inset:0;z-index:10;pointer-events:none;opacity:0.04;width:100%;height:100%" xmlns="http://www.w3.org/2000/svg">
  <filter id="gr"><feTurbulence type="fractalNoise" baseFrequency="0.85" numOctaves="4" stitchTiles="stitch"/><feColorMatrix type="saturate" values="0"/></filter>
  <rect width="100%" height="100%" filter="url(#gr)"/>
</svg>
```

**Glassmorphism card (use for body text containers on point/story slides):**
```css
.glass-card {
  background: rgba(15,23,42,0.6);
  backdrop-filter: blur(24px);
  -webkit-backdrop-filter: blur(24px);
  border: 1px solid rgba(255,255,255,0.08);
  box-shadow: inset 0 1px 0 rgba(255,255,255,0.09), 0 16px 48px rgba(0,0,0,0.45);
  border-radius: 24px;
  padding: 40px 44px;
  width: 100%;
}
```

**Dot grid overlay (use on hook + CTA slides):**
```css
.dot-grid {
  position: absolute; inset: 0; pointer-events: none;
  background-image: radial-gradient(rgba(255,255,255,0.07) 1px, transparent 1px);
  background-size: 32px 32px;
}
```

---

### TEMPLATE A — Hook Slide

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"/>
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin/>
<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;600;700&family=Inter:wght@400;500;600&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet"/>
<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
:root { /* ...vars... */ }
html, body { width: 1080px; height: 1080px; overflow: hidden; background: #030712; }
.slide {
  width: 1080px; height: 1080px; position: relative; overflow: hidden;
  background: #030712;
  display: flex; flex-direction: column;
  justify-content: center; align-items: center; text-align: center; padding: 80px;
}
/* Mesh gradient layers */
.bg-cyan { position: absolute; inset: 0; pointer-events: none;
  background: radial-gradient(ellipse 70% 55% at 50% 25%, rgba(14,165,233,0.22) 0%, transparent 65%); }
.bg-violet { position: absolute; inset: 0; pointer-events: none;
  background: radial-gradient(ellipse 55% 45% at 85% 85%, rgba(139,92,246,0.14) 0%, transparent 60%); }
.dot-grid { position: absolute; inset: 0; pointer-events: none;
  background-image: radial-gradient(rgba(255,255,255,0.07) 1px, transparent 1px);
  background-size: 32px 32px; }
/* Brand label */
.brand-topleft { position: absolute; top: 44px; left: 52px;
  font-family: 'Space Grotesk', sans-serif; font-size: 15px; font-weight: 700;
  color: rgba(255,255,255,0.22); letter-spacing: 0.04em; z-index: 2; }
/* Badge */
.badge { display: inline-flex; align-items: center; gap: 8px;
  padding: 7px 20px; border-radius: 999px;
  background: rgba(14,165,233,0.1); border: 1px solid rgba(14,165,233,0.25);
  font-family: 'JetBrains Mono', monospace; font-size: 13px; font-weight: 500;
  color: #0ea5e9; letter-spacing: 0.07em; text-transform: uppercase;
  margin-bottom: 40px; position: relative; z-index: 2; }
.badge .dot { width: 7px; height: 7px; border-radius: 50%; background: #0ea5e9; flex-shrink: 0; }
/* Headline */
.headline { font-family: 'Space Grotesk', sans-serif; font-weight: 700;
  font-size: 88px; letter-spacing: -0.03em; line-height: 1.0;
  background: linear-gradient(135deg, #38bdf8 0%, #0ea5e9 45%, #a78bfa 100%);
  -webkit-background-clip: text; -webkit-text-fill-color: transparent; background-clip: text;
  margin-bottom: 32px; position: relative; z-index: 2; word-break: break-word; }
/* Subheadline */
.subheadline { font-family: 'Inter', sans-serif; font-size: 24px; font-weight: 400;
  color: #a6b8d4; line-height: 1.55; max-width: 740px;
  position: relative; z-index: 2; }
.brand-watermark { position: absolute; bottom: 44px; right: 52px;
  font-family: 'JetBrains Mono', monospace; font-size: 13px; font-weight: 500;
  color: rgba(255,255,255,0.18); letter-spacing: 0.08em; z-index: 2; }
</style>
</head>
<body>
<div class="slide">
  <div class="bg-cyan"></div>
  <div class="bg-violet"></div>
  <div class="dot-grid"></div>
  <span class="brand-topleft">Founders Wing</span>
  <span class="badge"><span class="dot"></span>{{BADGE_TEXT}}</span>
  <h1 class="headline">{{HEADLINE}}</h1>
  <p class="subheadline">{{SUBHEADLINE}}</p>
  <span class="brand-watermark">founderwing.com</span>
  <!-- GRAIN -->
  <svg style="position:absolute;inset:0;z-index:10;pointer-events:none;opacity:0.04;width:100%;height:100%" xmlns="http://www.w3.org/2000/svg"><filter id="gr"><feTurbulence type="fractalNoise" baseFrequency="0.85" numOctaves="4" stitchTiles="stitch"/><feColorMatrix type="saturate" values="0"/></filter><rect width="100%" height="100%" filter="url(#gr)"/></svg>
</div>
</body>
</html>
```

---

### TEMPLATE B — Point Slide (with glassmorphism card)

```html
<!-- Structure:
  slide counter top-right
  brand label top-left
  accent line (left) + optional stat (cyan mono)
  glass-card wrapping headline + body
  source label bottom-left
  grain overlay
-->
.slide { background: #030712; position: relative; padding: 100px 104px;
  display: flex; flex-direction: column; justify-content: center; }
/* Top-right ambient glow */
.bg-glow { position: absolute; top: -120px; right: -120px; width: 480px; height: 480px;
  pointer-events: none;
  background: radial-gradient(circle, rgba(14,165,233,0.09), transparent 68%); }
.accent-line { width: 5px; height: 68px; border-radius: 5px;
  background: linear-gradient(to bottom, #38bdf8, #0ea5e9);
  margin-bottom: 28px; flex-shrink: 0; box-shadow: 0 0 16px rgba(56,189,248,0.4); }
.stat { font-family: 'JetBrains Mono', monospace; font-size: 64px; font-weight: 700;
  background: linear-gradient(135deg, #38bdf8, #0ea5e9);
  -webkit-background-clip: text; -webkit-text-fill-color: transparent; background-clip: text;
  line-height: 1; margin-bottom: 20px; }
.glass-card { background: rgba(15,23,42,0.6); backdrop-filter: blur(24px);
  border: 1px solid rgba(255,255,255,0.08);
  box-shadow: inset 0 1px 0 rgba(255,255,255,0.09), 0 16px 48px rgba(0,0,0,0.45);
  border-radius: 24px; padding: 40px 44px; width: 100%; }
.headline { font-family: 'Space Grotesk', sans-serif; font-weight: 700;
  font-size: 58px; letter-spacing: -0.025em; line-height: 1.1; color: #fff; margin-bottom: 20px; }
.body { font-family: 'Inter', sans-serif; font-size: 23px; font-weight: 400;
  color: #a6b8d4; line-height: 1.65; }
```

---

### TEMPLATE B2 — Chart Slide (Chart.js horizontal bar)

```html
<!-- CDN: https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js -->
<!-- Canvas: width=840 height=300 -->
<!-- Config: horizontal bar, animation:false, clean dark theme -->
<!-- After chart creation: window.__chartDone = true -->

Key Chart.js config:
{
  type: 'bar',
  options: {
    animation: false,
    indexAxis: 'y',    // horizontal bars
    responsive: false,
    plugins: {
      legend: { display: false },
      tooltip: { enabled: false }
    },
    scales: {
      x: {
        ticks: { color: '#a6b8d4', font: { family: 'JetBrains Mono', size: 18 },
                 callback: (v) => unit + v },
        grid: { color: 'rgba(255,255,255,0.05)' },
        border: { color: 'rgba(255,255,255,0.1)' }
      },
      y: {
        ticks: { color: '#ffffff', font: { family: 'Space Grotesk', size: 22, weight:'700' } },
        grid: { display: false },
        border: { display: false }
      }
    }
  },
  data: {
    labels: chart_labels,
    datasets: [{
      data: chart_values,
      backgroundColor: ['rgba(166,184,212,0.15)', 'rgba(14,165,233,0.75)'],
      borderColor:     ['rgba(166,184,212,0.4)',  '#38bdf8'],
      borderWidth: 2,
      borderRadius: 10,
      borderSkipped: false,
    }]
  }
}
```

Layout:
- `headline` above the glass card
- chart canvas inside glass card
- `chart_note` below chart (muted, JetBrains Mono)
- source label bottom-left

---

### TEMPLATE C — Story/Quote Slide (glass card + violet)

```html
<!-- Violet mesh background (2 radials: bottom-left large, top-right small)
  Large violet quote mark (160px, gradient #a78bfa→#8b5cf6)
  Glass card wrapping quote-text + attribution + context
  Brand watermark bottom-right
  Grain overlay
-->
.bg-violet-main { position: absolute; bottom: -100px; left: -100px; width: 500px; height: 500px;
  background: radial-gradient(circle, rgba(139,92,246,0.20), transparent 65%); }
.bg-violet-accent { position: absolute; top: -80px; right: -80px; width: 300px; height: 300px;
  background: radial-gradient(circle, rgba(139,92,246,0.09), transparent 65%); }
.glass-card { /* same as Template B but violet tint */
  background: rgba(15,8,30,0.65);
  border-color: rgba(139,92,246,0.15);
  box-shadow: inset 0 1px 0 rgba(167,139,250,0.08), 0 16px 48px rgba(0,0,0,0.5); }
.quote-mark { font-family: 'Space Grotesk', sans-serif; font-size: 160px; font-weight: 700;
  line-height: 0.7; margin-bottom: 12px;
  background: linear-gradient(135deg, #a78bfa, #8b5cf6);
  -webkit-background-clip: text; -webkit-text-fill-color: transparent; }
.quote-text { font-family: 'Space Grotesk', sans-serif; font-size: 40px; font-weight: 600;
  letter-spacing: -0.02em; line-height: 1.3; color: #fff; margin-bottom: 28px; }
.attribution { font-family: 'JetBrains Mono', monospace; font-size: 15px; font-weight: 500;
  color: #a78bfa; letter-spacing: 0.05em; margin-bottom: 10px; }
.context { font-family: 'Inter', sans-serif; font-size: 20px; color: #a6b8d4; line-height: 1.55; }
```

---

### TEMPLATE D — CTA Slide

```html
<!-- Deep blue mesh bg (same as before but with TWO rings for depth)
  Outer ring: 700px, opacity 0.05
  Inner ring: 500px, opacity 0.08
  Button: gradient + glow
  Grain overlay
-->
Two rings using position:absolute, border-radius:50%, border: 1px solid rgba(14,165,233,0.N)
```

---

### TEMPLATE E — List Slide (Lucide inline SVG icons)

Icon SVG paths to use (stroke icons, viewBox="0 0 24 24", stroke-width="2"):

| Role | Icon name | SVG path |
|------|-----------|----------|
| Writing | PenLine | `<path d="M12 20h9"/><path d="M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4L16.5 3.5z"/>` |
| Admin | Calendar | `<rect x="3" y="4" width="18" height="18" rx="2"/><line x1="16" y1="2" x2="16" y2="6"/><line x1="8" y1="2" x2="8" y2="6"/><line x1="3" y1="10" x2="21" y2="10"/>` |
| Content | Video | `<polygon points="23 7 16 12 23 17 23 7"/><rect x="1" y="5" width="15" height="14" rx="2"/>` |
| Automation | Zap | `<polygon points="13 2 3 14 12 14 11 22 21 10 12 10 13 2"/>` |
| Marketing | TrendingUp | `<polyline points="23 6 13.5 15.5 8.5 10.5 1 18"/><polyline points="17 6 23 6 23 12"/>` |
| Finance | DollarSign | `<line x1="12" y1="1" x2="12" y2="23"/><path d="M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"/>` |
| Support | MessageSquare | `<path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"/>` |

Layout per row:
```html
<div class="tool-row">
  <div class="icon-wrap">  <!-- 48×48px, glass bg, rounded-14 -->
    <svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="{{ACCENT_COLOR}}" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">{{ICON_PATH}}</svg>
  </div>
  <span class="role-label">{{ROLE}}</span>  <!-- JetBrains Mono, cyan, 12px uppercase -->
  <span class="tool-name">{{TOOL}} — {{DESCRIPTION}}</span>  <!-- Inter, muted, 20px -->
</div>
```

---

## PHASE 4 — Render PNGs

Run for each carousel:
```bash
cd "/Users/prithal/Documents/carousel routine" && node render.js "$(date +%Y-%m-%d)" "carousel-1" 2>&1
cd "/Users/prithal/Documents/carousel routine" && node render.js "$(date +%Y-%m-%d)" "carousel-2" 2>&1
cd "/Users/prithal/Documents/carousel routine" && node render.js "$(date +%Y-%m-%d)" "carousel-3" 2>&1
```

Expected output per run: `Found N slide(s). Launching headless browser… ✓ slide-01.png … Done.`

If any run fails with non-zero exit, report the error and stop.

---

## PHASE 4B — Export PDFs

For each carousel, combine all rendered PNGs into a single PDF file:

```bash
# Check ImageMagick is available
which magick || which convert || echo "NOT_FOUND"
```

If NOT_FOUND, install with: `brew install imagemagick`

Then run for each carousel (N = 1, 2, 3):
```bash
DATE=$(date +%Y-%m-%d)
OUT="/Users/prithal/Documents/carousel routine/output/$DATE"

magick "$OUT/carousel-1/slide-"*.png "$OUT/carousel-1/carousel-1.pdf"
magick "$OUT/carousel-2/slide-"*.png "$OUT/carousel-2/carousel-2.pdf"
magick "$OUT/carousel-3/slide-"*.png "$OUT/carousel-3/carousel-3.pdf"
```

Expected output per carousel: a single `carousel-N.pdf` file inside its output folder, with slides in order. If the command fails, report the error and continue (PNGs are the primary deliverable).

---

## PHASE 5 — Log and Report

Append 3 entries to `/Users/prithal/Documents/carousel routine/output/run-log.json`:

```json
[
  {
    "date": "YYYY-MM-DD",
    "carousel": "carousel-1",
    "topic": "...",
    "source_url": "...",
    "slide_count": N,
    "output_dir": "output/YYYY-MM-DD/carousel-1/",
    "generated_at": "ISO timestamp"
  },
  { "carousel": "carousel-2", ... },
  { "carousel": "carousel-3", ... }
]
```

Print final summary:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Founders Wing — 3 Carousels — YYYY-MM-DD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
C1 [how-to/list]:   output/YYYY-MM-DD/carousel-1/  (N slides + carousel-1.pdf)
C2 [case study]:    output/YYYY-MM-DD/carousel-2/  (N slides + carousel-2.pdf)
C3 [macro trend]:   output/YYYY-MM-DD/carousel-3/  (N slides + carousel-3.pdf)

All 3 ready to upload to LinkedIn.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Error Handling

- **ScrapingDog unavailable:** viral-tweet-engine uses WebSearch fallback automatically.
- **render.js "Cannot find module 'puppeteer'":** Run `cd "/Users/prithal/Documents/carousel routine" && npm install`.
- **Chart.js not loading:** 300ms buffer in render.js handles async canvas render. If chart still blank, check CDN URL.
- **Font overflow:** Hook headline ≤35 chars, point headline ≤45 chars. Trim copy before writing HTML.
- **Puppeteer version:** Uses `headless: true` (correct for v24+).
