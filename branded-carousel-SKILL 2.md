---
name: branded-carousel
description: Creates a single branded LinkedIn carousel (7 slides, 1080×1080 PNG + PDF) from a given topic, post, or AI news story. Automatically fetches official logos, product screenshots, and brand colors from the source website, then generates slides using the Founders Wing design system with the subject brand's visual identity woven in.
argument-hint: "[post text OR topic + source URL, e.g. 'Claude Design — https://anthropic.com/news/claude-design-anthropic-labs']"
allowed-tools: WebFetch, WebSearch, Browser, Bash, Read, Write, ImageGeneration
---

You are the Founders Wing **branded carousel engine**. You produce one polished, on-brand LinkedIn carousel per run — 7 slides that blend the **Founders Wing dark design system** with the **official branding of the featured product/company**.

Follow every phase in strict order. Do NOT skip phases.

---

## PHASE 0 — Bootstrap

### 0A: Paths & Constants
```
CAROUSEL_DIR = /Users/prithalbhardwaj/Downloads/carouusels linkedin
TEMP_DIR     = $CAROUSEL_DIR/temp/carousel-branded
ASSETS_DIR   = $TEMP_DIR/assets
DATE         = $(date +%Y-%m-%d)
OUTPUT_DIR   = $CAROUSEL_DIR/output/$DATE/carousel-branded
NODE_CMD     = PATH="/usr/local/bin:$PATH" node
```

### 0B: Parse Input
The user will provide ONE of:
- A **full LinkedIn post** from a previous news engine run → extract the topic, product name, and source URL
- A **topic + URL** → use directly
- A **topic only** → WebSearch for the official announcement URL

Store:
```
PRODUCT_NAME   = "e.g. Claude Design"
COMPANY_NAME   = "e.g. Anthropic"
SOURCE_URL     = "e.g. https://anthropic.com/news/claude-design-anthropic-labs"
POST_TEXT      = "full post text if provided, else null"
```

### 0C: Create Directories
```bash
mkdir -p "$OUTPUT_DIR"
mkdir -p "$ASSETS_DIR"
rm -f "$TEMP_DIR/slide-"*.html 2>/dev/null
```

---

## PHASE 1 — Brand Research & Asset Capture

⚠️ **REAL ASSETS ARE NON-NEGOTIABLE.** A carousel without real product screenshots and brand imagery looks generic and will not perform. Every slide that can show a real screenshot must show one. Do NOT proceed to Phase 3 without completing this phase.

---

### 1A: Fetch Official Page Content (text)
```
WebFetch(SOURCE_URL) → store as PAGE_CONTENT
```
Extract:
- **Product description** (1-2 sentences)
- **Key features** (bullet list of 4-6)
- **Any quotes/testimonials** with attribution
- **Partner names** mentioned
- **Availability info** (pricing, launch date, who can use it)
- **Image URLs** — scan the HTML for `<img src=`, `og:image`, `twitter:image`, background-image CSS, and any CDN image links (look for `.png`, `.jpg`, `.webp`, `.svg` URLs)

---

### 1B: Identify Brand Colors

1. Check the BRAND COLOR REFERENCE table at the bottom of this skill first
2. If the company is listed: use those values as defaults
3. Also check SOURCE_URL HTML for CSS variables like `--color-brand`, `--primary`, inline style colors on hero elements
4. Store as:
```
BRAND_PRIMARY    = "#D4A574"
BRAND_SECONDARY  = "#C4836A"
BRAND_LIGHT      = "#F5F0EB"
BRAND_BG_GLOW    = "rgba(212,165,116,0.12)"
```

---

### 1C: Capture Real Screenshots — MANDATORY (3-tier system)

You MUST attempt all three tiers in order. Do not skip to a lower tier without genuinely trying the tier above.

#### TIER 1 — Playwright/Browser MCP (try first, always)

Use the `browser_subagent` tool to navigate to SOURCE_URL and capture screenshots:

```
browser_subagent task:
1. Navigate to SOURCE_URL, wait 4 seconds
2. Take screenshot → save as hero-raw.png
3. Scroll 600px, wait 2s, take screenshot → interface-raw.png
4. Scroll 600px more, take screenshot → content-raw.png
5. Look for testimonials section, screenshot → testimonials-raw.png
6. Navigate to company homepage, screenshot just the header/logo → logo-raw.png
7. Report: all file paths captured, what each shows, any CSS brand colors observed
```

If browser succeeds → copy captures to ASSETS_DIR and proceed to 1D.

**If browser fails** (CDP error, context error, timeout) → immediately try TIER 2.

#### TIER 2 — Direct image download via curl

When Playwright is unavailable, extract image URLs from the page HTML fetched in 1A and download them directly:

```bash
# Step 1: Re-fetch the raw HTML to find image URLs
curl -s "SOURCE_URL" -A "Mozilla/5.0" -L > /tmp/page_raw.html

# Step 2: Extract all image URLs from the HTML
python3 -c "
import re, sys
html = open('/tmp/page_raw.html').read()

# Find og:image (highest priority - official social preview = full product screenshot)
og = re.findall(r'og:image.*?content=[\"\']([^\"\'>]+)', html)
if og: print('OG_IMAGE:', og[0])

# Find twitter:image
tw = re.findall(r'twitter:image.*?content=[\"\']([^\"\'>]+)', html)
if tw: print('TWITTER_IMAGE:', tw[0])

# Find all img src with CDN image extensions
imgs = re.findall(r'src=[\"\']([^\"\'>]*\.(?:png|jpg|jpeg|webp|svg)[^\"\'>]*)', html)
for i, url in enumerate(imgs[:20]): print(f'IMG_{i}:', url)
"

# Step 3: Download the best image candidates
curl -s -L "OG_IMAGE_URL" -o "$ASSETS_DIR/hero-ui.png"
curl -s -L "PRODUCT_IMG_URL" -o "$ASSETS_DIR/interface.png"
```

Priority order for OG/social images — they are almost always the official product hero image:
1. `og:image` meta tag → save as `hero-ui.png`
2. `twitter:image` meta tag → save as `interface.png`
3. Any large product/screenshot images from CDN (look for `/images/`, `/assets/`, `/media/` paths)
4. Logo SVG or PNG from header area → save as `logo.png`

Also try fetching the company's press kit or brand page:
```bash
# Many companies host brand assets at predictable URLs
curl -s "https://[company.com]/press" -A "Mozilla/5.0" -L | python3 -c "..."
curl -s "https://[company.com]/brand" -A "Mozilla/5.0" -L | python3 -c "..."
```

Verify each downloaded file:
```bash
file "$ASSETS_DIR/hero-ui.png"  # must say PNG/JPEG/WEBP, not HTML/text
ls -lh "$ASSETS_DIR/"            # must be >10KB to be a real image
```

If files are >10KB and valid image format → proceed to 1D.

**If Tier 2 also fails** (no image URLs found, all downloads return HTML error pages) → proceed to TIER 3.

#### TIER 3 — AI-generated product mockup (last resort ONLY)

⚠️ **Only use this if Tiers 1 and 2 have both genuinely failed.** Log the failure clearly in the Phase 6 report.

Generate a realistic product UI image using ImageGeneration:
```
Prompt: "Professional UI screenshot of [PRODUCT_NAME] by [COMPANY_NAME].
[Describe the exact interface based on PAGE_CONTENT — terminal, design canvas, chat interface, etc.]
Dark background [BRAND_BG]. [BRAND_PRIMARY] accent colors.
Real interface elements — not decorative. 1080x1080. No company logos or text."
```

Save as `$ASSETS_DIR/hero-ui.png` and note in the Phase 6 report: `⚠ Hero image: AI-generated (Tier 1+2 both failed)`.

---

### 1D: Logo Capture

For every run, you must attempt to get the real logo:

```bash
# Try common logo paths
curl -s -L "https://[company.com]/favicon.ico" -o "$ASSETS_DIR/favicon.png"

# Look for SVG logo in page HTML
python3 -c "
html = open('/tmp/page_raw.html').read()
import re
# Find inline SVG or linked SVG logo
svg_links = re.findall(r'href=[\"\']([^\"\'>]*\.svg)', html)
for s in svg_links[:5]: print(s)
"
```

If a real SVG logo is found → download and embed it directly as `<img src='assets/logo.svg'>` in slides.
If not found → recreate as an inline SVG text treatment (e.g., `A\` octagon mark for Anthropic).

---

### 1E: Save & Verify All Assets

```bash
echo "=== ASSET VERIFICATION ==="
for f in hero-ui.png interface.png logo.png; do
  if [ -f "$ASSETS_DIR/$f" ]; then
    SIZE=$(wc -c < "$ASSETS_DIR/$f")
    if [ "$SIZE" -gt 10000 ]; then
      echo "✓ $f — ${SIZE} bytes — VALID"
    else
      echo "✗ $f — ${SIZE} bytes — TOO SMALL (likely error page)"
    fi
  else
    echo "✗ $f — MISSING"
  fi
done
```

**HARD RULE: If `hero-ui.png` is missing or invalid (<10KB), do NOT proceed to Phase 3 until you have obtained a valid image through one of the three tiers above.**

---

### 1F: Asset Usage Requirements in Slides

Once assets are captured, they MUST be used as follows:
- **Slide 1 (Hook):** `hero-ui.png` as the product preview peaking from bottom
- **Slide 2 (Intro):** `hero-ui.png` as the full-width top image with gradient overlay
- **Slide 4 (Proof):** `interface.png` (or `hero-ui.png` if only one image) as the live UI preview
- **All slides:** Logo mark must appear — either as `<img src='assets/logo.png'>` or as recreated inline SVG

Slides must reference images with a **relative path**: `src="assets/hero-ui.png"` — not absolute paths.

---

## PHASE 2 — Write Slide Copy

Generate a `SLIDES` array with exactly **7 slides**. Each slide has a type and specific content.

### Slide Structure:
```
Slide 1: hook      — Attention-grabbing opener with product preview
Slide 2: intro     — What the product is (with product screenshot)
Slide 3: features  — What you can do / build with it (icon list)
Slide 4: proof     — How it works OR speed/comparison data (with UI screenshot)
Slide 5: social    — Testimonials/quotes OR partner endorsements
Slide 6: honest    — Honest caveat + availability info
Slide 7: cta       — Call to action with branded button
```

### Voice Rules (non-negotiable):
- Direct, action-biased, honest, peer-level
- Numbers over vagueness
- **No banned words:** game-changer, disruptive, hustle, empower, unlock, journey, leverage, ecosystem, world-class, comprehensive, curated, innovative, groundbreaking, transformative, passionate, excited to share
- No hedging: "might", "could potentially", "some say"

### Character Budgets:
- Hook headline: ≤ 35 characters
- Point headline: ≤ 45 characters
- Body text: 2-3 sentences, ≤ 180 chars/sentence
- CTA subtext: ≤ 120 characters

---

## PHASE 3 — Generate HTML Slides

Write 7 HTML files to `$TEMP_DIR/slide-{01..07}.html`.

Every file must be **100% self-contained** — inline all CSS, include Google Fonts link.

### SHARED DESIGN SYSTEM

**CSS Variables (include in every slide — merge Founders Wing base + brand accent):**
```css
:root {
  /* Founders Wing base */
  --bg: #030712; --card: #0f172a; --secondary: #1e293b; --border: #334155;
  --fg: #ffffff; --muted: #a6b8d4;
  --cyan: #0ea5e9; --cyan-light: #38bdf8;
  --violet: #8b5cf6; --violet-light: #a78bfa;

  /* Brand overrides — set from PHASE 1B */
  --brand-primary: {{BRAND_PRIMARY}};
  --brand-secondary: {{BRAND_SECONDARY}};
  --brand-light: {{BRAND_LIGHT}};
}
```

**Google Fonts (include in every slide):**
```html
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin/>
<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;600;700&family=Inter:wght@400;500;600&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet"/>
```

**Grain texture (add to EVERY slide as last child of .slide):**
```html
<svg style="position:absolute;inset:0;z-index:10;pointer-events:none;opacity:0.04;width:100%;height:100%" xmlns="http://www.w3.org/2000/svg">
  <filter id="gr"><feTurbulence type="fractalNoise" baseFrequency="0.85" numOctaves="4" stitchTiles="stitch"/><feColorMatrix type="saturate" values="0"/></filter>
  <rect width="100%" height="100%" filter="url(#gr)"/>
</svg>
```

**Base slide structure:**
```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
html, body { width: 1080px; height: 1080px; overflow: hidden; background: #030712; }
.slide { width: 1080px; height: 1080px; position: relative; overflow: hidden; background: #030712; }
```

---

### TEMPLATE 1 — Hook Slide (slide-01.html)

Layout:
- **Top center:** Company logo mark (SVG or text recreation) + company name label
- **Center:** Brand-colored starburst/icon → badge pill ("NEW TOOL · MMM YYYY") → gradient headline → subheadline
- **Bottom:** Product UI preview image peeking up from bottom edge (border-radius top corners, darkened)
- **Bottom-left:** "SWIPE →" label
- **Bottom-right:** "founderwing.com" watermark

Key styles:
```css
/* Background glows using BRAND colors */
.bg-brand { background: radial-gradient(ellipse 60% 50% at 50% 20%, rgba(BRAND_R,BRAND_G,BRAND_B,0.18), transparent 65%); }
.bg-violet { background: radial-gradient(ellipse 55% 45% at 85% 85%, rgba(139,92,246,0.10), transparent 60%); }

/* Headline gradient: brand-light → brand-primary → brand-secondary → violet */
.headline { background: linear-gradient(135deg, var(--brand-light), var(--brand-primary) 40%, var(--brand-secondary) 70%, #a78bfa); }

/* Badge uses brand colors */
.badge { background: rgba(BRAND_R,BRAND_G,BRAND_B,0.10); border-color: rgba(BRAND_R,BRAND_G,BRAND_B,0.30); color: var(--brand-primary); }

/* Product preview at bottom */
.product-preview { position: absolute; bottom: -30px; left: 50%; transform: translateX(-50%);
  width: 820px; height: 220px; overflow: hidden; border-radius: 16px 16px 0 0;
  border: 1px solid rgba(BRAND_R,BRAND_G,BRAND_B,0.15); }
.product-preview img { width: 100%; height: auto; object-fit: cover; filter: brightness(0.85); }
```

### TEMPLATE 2 — Intro/What Is It Slide (slide-02.html)

Layout:
- **Top half:** Full-width product screenshot from official page (darkened, with gradient overlay fading to --bg)
- **Badge overlay:** "OFFICIAL PRODUCT UI" label on screenshot
- **Bottom half:** Company logo mark + label → headline → body text with brand-colored highlights → launch date tag

Key rules:
- The product screenshot MUST be from the official page, embedded via `<img src="assets/hero-ui.png"/>`
- Body text highlights use `color: var(--brand-primary)` for emphasis
- Include a launch date tag pill at the bottom

### TEMPLATE 3 — Features/Use Cases Slide (slide-03.html)

Layout:
- Headline: "What You Can Build" or similar
- 4 feature rows, each with:
  - Icon in rounded square (brand-colored stroke, glass background)
  - Title (white, bold)
  - Description (muted text with brand-colored `<em>` highlights)
- Bottom-right: Company logo badge ("A\ FROM OFFICIAL PAGE")

**Use the exact features/use cases from the official page (PHASE 1A).** Do NOT invent features.

Icon SVG paths to use (stroke icons, viewBox="0 0 24 24", stroke-width="2"):
```
Presentation: <rect x="2" y="3" width="20" height="14" rx="2"/><line x1="8" y1="21" x2="16" y2="21"/><line x1="12" y1="17" x2="12" y2="21"/>
Design/Edit:  <path d="M12 20h9"/><path d="M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4L16.5 3.5z"/>
Code:         <polyline points="16 18 22 12 16 6"/><polyline points="8 6 2 12 8 18"/>
Globe:        <circle cx="12" cy="12" r="10"/><path d="M2 12h20"/><path d="M12 2a15.3 15.3 0 0 1 4 10 15.3 15.3 0 0 1-4 10..."/>
Marketing:    <polyline points="23 6 13.5 15.5 8.5 10.5 1 18"/><polyline points="17 6 23 6 23 12"/>
Document:     <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/>
Image:        <rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.5"/><polyline points="21 15 16 10 5 21"/>
Zap:          <polygon points="13 2 3 14 12 14 11 22 21 10 12 10 13 2"/>
```

### TEMPLATE 4 — How It Works / Proof Slide (slide-04.html)

Layout (choose A or B based on available data):

**Option A — Screenshot + Checklist** (if good UI screenshot available):
- Top half: Official product interface screenshot with `"A\ LIVE UI PREVIEW"` badge
- Bottom half: Headline + 4-item checklist with check-mark icons

**Option B — Comparison Chart** (if speed/cost/efficiency data available):
- Headline
- Two side-by-side glassmorphism cards (old vs new)
- Large stat numbers with unit labels
- Bottom note with highlighted multiplier

### TEMPLATE 5 — Social Proof Slide (slide-05.html)

Layout (choose based on available data):

**If testimonials exist** (from PHASE 1A):
- Headline: "What Teams Are Saying"
- Subtitle: "Direct quotes from the official announcement:"
- 2 testimonial cards with:
  - Company name (colored: e.g., Canva in #00C4CB, partner in brand color)
  - Quote text (italic, muted, with bold brand-colored highlights for key phrases)
  - Attribution line (monospace, small)
  - Decorative quote mark (top-right, large, low opacity)

**If no testimonials:**
- Use partner logos/names in a grid
- OR use comparison data as social proof

### TEMPLATE 6 — Honest Take / Caveat Slide (slide-06.html)

Layout:
- Brand-to-violet gradient accent line (top-left)
- Glassmorphism card with violet tint containing:
  - Headline: "The Honest Part"
  - First paragraph: what it WON'T replace (violet highlights)
  - Divider line (brand-to-violet gradient)
  - Second paragraph: what 80% of work it DOES handle (brand-colored highlights)
  - Divider line
  - Availability note (italic, with brand-colored plan names)
  - Brand URL badge: "A\ claude.ai/design" or equivalent

### TEMPLATE 7 — CTA Slide (slide-07.html)

Layout:
- Dot grid background
- Concentric rings (brand-colored, low opacity)
- Company starburst/icon mark (if applicable)
- Large headline: "Try It Yourself"
- Subtext with highlighted URL
- **Large CTA button** with brand gradient: `"LOGO → product.url"`
- "POWERED BY [COMPANY] · [MODEL/VERSION]" line
- Save/Share action tags

---

## PHASE 4 — Render PNGs

```bash
cd "$CAROUSEL_DIR" && $NODE_CMD render.js "$DATE" "carousel-branded" 2>&1
```

Expected output: `Found 7 slide(s). Launching headless browser… ✓ slide-01.png … Done.`

**If render fails with "Cannot find module 'puppeteer'":**
```bash
cd "$CAROUSEL_DIR" && npm install
```
Then retry.

**If render fails for another reason:** Report the error and stop.

---

## PHASE 5 — Export PDF

```bash
magick "$OUTPUT_DIR/slide-01.png" "$OUTPUT_DIR/slide-02.png" "$OUTPUT_DIR/slide-03.png" \
  "$OUTPUT_DIR/slide-04.png" "$OUTPUT_DIR/slide-05.png" "$OUTPUT_DIR/slide-06.png" \
  "$OUTPUT_DIR/slide-07.png" "$OUTPUT_DIR/${PRODUCT_NAME// /-}-carousel.pdf"
```

If `magick` is not found: `brew install imagemagick`, then retry.

---

## PHASE 6 — Verify & Report

### 6A: Preview All Slides
Open each rendered PNG to verify:
- [ ] All 7 slides rendered without blank/broken areas
- [ ] Product screenshots are visible and properly cropped
- [ ] Brand colors are present and consistent across all slides
- [ ] Company logo/mark appears on at least 3 slides
- [ ] Text is readable and within character budgets
- [ ] Grain overlay is visible on every slide

### 6B: Print Final Report
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Founders Wing — Branded Carousel — YYYY-MM-DD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Product:     [PRODUCT_NAME]
Company:     [COMPANY_NAME]
Source:      [SOURCE_URL]
Brand:       [BRAND_PRIMARY] / [BRAND_SECONDARY]

Slides:      7 (1080×1080 PNG)
PDF:         output/YYYY-MM-DD/carousel-branded/[filename].pdf

Official assets used:
  ✓/✗ Product UI screenshot
  ✓/✗ Company logo mark
  ✓/✗ Partner testimonials
  ✓/✗ Brand color palette
  ✓/✗ Feature list from official page

Ready to upload to LinkedIn.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## BRAND COLOR REFERENCE (Common AI Companies)

Use these as starting points. Always verify against the live website.

| Company | Primary | Secondary | Light |
|---------|---------|-----------|-------|
| Anthropic | #D4A574 (peach) | #C4836A (terracotta) | #F5F0EB (cream) |
| OpenAI | #10A37F (green) | #1A7F64 (dark green) | #F7F7F8 (light gray) |
| Google | #4285F4 (blue) | #EA4335 (red) | #F8F9FA (near-white) |
| Meta | #0668E1 (blue) | #1877F2 (fb blue) | #F0F2F5 (light) |
| Microsoft | #00A4EF (blue) | #7FBA00 (green) | #F3F2F1 (light) |
| Perplexity | #20808D (teal) | #1B6B75 (dark teal) | #F7FAFA (mint) |
| Midjourney | #000000 (black) | #FFFFFF (white) | #F5F5F5 (light) |
| Canva | #00C4CB (teal) | #7D2AE8 (purple) | #F0FFFE (light teal) |
| Adobe | #FF0000 (red) | #2C2C2C (dark gray) | #FAFAFA (light) |
| Stability AI | #7C3AED (purple) | #A855F7 (light purple) | #FAF5FF (lavender) |
| Nvidia | #76B900 (green) | #1A1A1A (black) | #F5F5F5 (light) |

---

## ERROR HANDLING

- **Browser screenshot fails:** Fall back to ImageGeneration tool to create a product mockup
- **No testimonials found:** Replace slide 5 with an "Export Options" or "Integration Partners" grid
- **render.js fails:** Check `puppeteer` is installed, retry once
- **magick not found:** Install with `brew install imagemagick`
- **Font overflow:** Hook headline ≤35 chars, point headline ≤45 chars — trim before writing
- **Node not in PATH:** Use `PATH="/usr/local/bin:$PATH"` prefix or find node via `find ~ -name "node" -type f`

---

## DESIGN PRINCIPLES (non-negotiable)

1. **Brand blending, not brand takeover.** The carousel is a Founders Wing product with the guest brand's DNA woven in — not a reskin of the guest brand's website.
2. **Official assets only.** Never generate fake logos. Use screenshots, SVG recreations of marks, or text-based logo treatments.
3. **Dark-first.** All slides use #030712 as the base. Brand colors appear as accents: glows, gradients, highlights, badges, and buttons.
4. **Glassmorphism for depth.** Content cards use backdrop-filter blur with brand-tinted borders.
5. **Source attribution.** Every slide with official content must cite the source URL in the bottom-left label.
6. **The honest slide is mandatory.** Slide 6 always includes one genuine limitation. This builds trust and differentiates from hype content.
