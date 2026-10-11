# ShelfIQ — UI Design Guide v2 (Functional Rebuild)

> **For:** front-end development
> **Scope:** the full visual and interaction spec for the ShelfIQ Merchandising Copilot web app — rebuilt to match the app that actually exists (`README.md`, `ShelfIQ_V2.yml`, `frontend/`, `server/` in this repo), not the earlier enterprise concept.
> **Replaces:** `ShelfIQ_UI_Design_Guide.md.txt` (v1). Keep that file for the visual system reference (colour, type, components) — this document is the one to build against for data and page structure.
> **Data/API source of truth:** `README.md` §§4–12 (engine formulas, response shapes) and `ShelfIQ_V2.yml` (the deployed Dify chatflow the app actually calls). `frontend/src/normalize.ts` already implements the response-shape parsing this guide assumes.

This guide keeps v1's entire visual language — the brand palette, typography, spacing, shadows, and most components are copied through unchanged, because the aesthetic was already right. What changes is *what the UI is wired to*, and — as of this revision — *the app's shell*: v1 was designed for a two-track enterprise product (fast-fashion + perishables, price elasticity, LP-solved donation routing, a 56-algorithm eval harness) that was never built. This app is one track — fast-fashion retail merchandising over the ~200-SKU catalog in `fast_fashion_product_profiles.csv`.

The app is a **single scrolling page**, matching a concrete visual reference the client supplied (an AI-Studio-Build export — see §0's newest changelog row): a sticky header, a **Hero** section (`#top`) with the composer and a "Try a scenario" picker, an **Analysis** section (`#analysis`) that shows the *one most recent* answer in rich, dashboard-style detail (KPI row, action cards, charts, methodology), and a pinned floating **follow-up composer** anchored to the bottom of the viewport. There is no sidebar and no growing conversation thread: asking a new question — from the hero composer, a scenario-picker row, or the follow-up composer — **replaces** the Analysis section's content with the new answer and scrolls to it. Nothing about a prior question stays on screen once a new one has been asked. This is a deliberate, explicit choice (§0) — earlier drafts of this guide specified a persistent multi-turn thread; that model has been abandoned in favour of matching the supplied reference exactly.

Because there is only ever one answer on screen, there is no separate "Simple"/"Dashboard" view-mode switch either — every answer renders at full dashboard richness (KPI row, safety-stock gauge, charts, methodology), since that's the density the reference shows and the user asked to keep every existing chart. "Dashboard" here still means "detail about the one answer you just got," not a live, always-on business dashboard — there's no session history or portfolio-level data — so the copy says that plainly rather than implying more than the backend provides.

---

## 0. What changed from v1, and why

| v1 had | v2 has | Why |
|---|---|---|
| Track A (seasonal markdowns) / Track B (perishables, production, donation routing) | One track: fast-fashion merchandising only | Perishables/production/donation routing were never built (`README.md` §1: explicitly out of scope) |
| Scenario picker hero with a rotated decorative card | Removed | No scenario-picker backend; adds visual weight without function |
| Price elasticity (+ source), Evidence source fields on each card | Removed | Not present anywhere in the API response (`README.md` §5, `ShelfIQ_V2.yml`'s structured-output schema) |
| Planning-horizon / max-discount sliders that re-run the query | Removed for v1 (noted as a future enhancement) | `POST /api/query` takes only `{ query }` — there's no parameter to send a slider value to yet |
| "SIM" badge on units-on-hand / margin, footer line "inventory figures are simulated" | Removed | Those columns come straight from the real catalog CSV (`README.md` §4: "authoritative schema") — they are not simulated in this build |
| Footer claim "COMPLIANT WITH THE EU AI ACT & ARTICLES 101/102 TFEU" | Narrowed to "Competition & consumer-law guard: Articles 101/102 TFEU · EU Omnibus Directive 2019/2161" | The build implements those two specific guards; a general EU AI Act compliance claim isn't substantiated by anything here |
| WRAP CO₂e accounting, LP-solver donation table, expiry-pricing stage cards | Removed | Track B only |
| Generic linear "Meter" for weeks of cover | **Safety Stock Gauge** — a zoned gauge (§7.2) | This guide's one new component, described below |
| Two-column Track A/B header switch | **Simple / Dashboard** view switch | Same segmented-control component, new meaning |
| Single-column page: hero → thread → bottom-pinned floating chat widget with a horizontally-scrolling chip row | **Two-pane app shell** (superseded by the next row — see below): a persistent left sidebar (suggested questions + compliance test + methodology + footer) and a main pane with a fixed-bottom composer | A build against the single-column spec rendered as a generic scrolling page rather than the familiar Copilot/ChatGPT chrome the user asked for at the time. |
| Two-pane sidebar shell + permanent multi-turn thread (previous revision, above) | **Single-page "latest result" shell**: sticky header, Hero (`#top`, composer + scenario-picker card), Analysis (`#analysis`, one answer at a time), Methodology (`#methodology`), pinned floating follow-up composer, page footer — §6 | The client supplied a concrete HTML reference (an AI-Studio-Build export) and, after being shown that its structure conflicts with the sidebar/thread model, explicitly chose to match the reference's structure rather than keep the sidebar. This is a deliberate reversal, not a correction — do not reintroduce the sidebar or a persistent thread. |
| Simple / Dashboard view-mode switch | **Removed.** Every answer always renders at full dashboard richness | The reference has no mode switch, and the user separately asked to keep every existing chart — with only one answer ever on screen, there's no case where a stripped-down "Simple" render is still useful |
| "Try asking" sidebar list | **Scenario picker card** (§5.12) — same three validated `suggested_questions`, same compliance-test row, now in the hero's right column, styled after the reference's "TRY A SCENARIO" card | Visual position changed; the click-to-ask contract (submit immediately, don't just fill the input) is unchanged and still mandatory (§6.2) |
| Bottom-pinned composer (main pane) | **Pinned floating follow-up composer** (§5.15), `position: sticky` near the bottom of the viewport, with 3 suggestion chips | Matches the reference exactly; same click-to-ask contract as the scenario picker |
| Reference's "Capital at risk £14,820" KPI, "SIM" badges on on-hand/margin, "Simulation Parameters" panel (price elasticity ε, "Gallego & van Ryzin (1994)" citation, "Newsvendor Critical Ratio: 0.68" as a *labelled parameter*), footer claim "EU ARTIFICIAL INTELLIGENCE ACT & GDPR ALGORITHMIC SAFEGUARDS COMPLIANT · HIGH-RISK DECISION SUPPORT REGIME" | **Capital-at-risk kept, but computed from real fields** (`units_on_hand × unit_cost`, summed over this answer's markdown-flagged items — §5.6); SIM badges dropped (unchanged from the prior revision — these are real CSV figures); the fabricated evidence panel is **replaced** by the real Reasoning/Expected-impact callouts (§5.9, FUZZY only) plus a link into the Methodology accordion (§5.11), whose Newsvendor-critical-ratio row is the one citation that's actually true of this engine; the AI Act/GDPR claim is **not carried over** — same narrowed claim as the prior revision (Articles 101/102 TFEU + EU Omnibus Directive 2019/2161, §6.5) | Every one of these fields either has no backing in the real API response (`README.md` §5, `ShelfIQ_V2.yml`) or names an economic model (Gallego & van Ryzin's dynamic-pricing solver) the engine doesn't actually run — it runs a fixed discount-ladder pacing rule instead. This guide's standing rule (§1: "never claim more than the API returns") applies to the reference exactly as it applied to v1; matching the reference's *layout* does not mean matching its *invented data*. |
| Reference's "Scenario parameters" slider panel (planning horizon, max discount) with a "Re-run with these parameters" button and a "Stochastic dynamic solver active" tag | **Kept as an inert, clearly-labelled preview control** (§5.12) — sliders update their own on-screen value only; the "re-run" button and "solver active" tag are dropped | `POST /api/query` takes only `{ query }` — there's no parameter to send a slider value to yet (same limitation noted in the first revision's row above). A button that looks like it re-runs the query but does nothing is worse than no button; a slider that's honestly decorative, with copy that says so, isn't. |

Everything else in v1 — colour, type, spacing, shadows, buttons, cards, the methodology accordion pattern — carries over and is restated below so this document is self-contained.

---

## 1. Design principles

- **Editorial, not dashboard-y — even in "Dashboard" mode.** The page still reads like a well-edited page on warm paper, not enterprise BI chrome. "Dashboard" adds detail and charts, not density for its own sake.
- **One action colour.** Brand Blue is the *only* colour for things you click to act (primary buttons, send button, active links). If it's blue and filled, it does something.
- **Brand colours carry meaning, not decoration.** Coral = urgency/markdown; Blue = reorder/positive action; Light blue = calm/hold/support graphics; Graphite = drop/dead stock.
- **Warm neutrals only.** No cool greys anywhere.
- **Never colour alone.** Every colour-coded item also has a word, a label, or a legend.
- **Never claim more than the API returns.** If a field isn't in the response shape (§12.4–12.5), the UI doesn't render a placeholder for it — it omits that element entirely.
- **Accessible by default.** All text passes WCAG 2.1 AA (4.5:1; 3:1 for large text). Interactive boundaries pass 3:1.

---

## 2. Colour system

*(Unchanged from v1 — copied here so this document is self-contained. If v1's palette is ever revised, update both files together.)*

### 2.1 Brand core colours

| Swatch | Name | Hex | Role in the UI |
|---|---|---|---|
| 🟦 | **Blue** | `#0259DD` | The single action colour: primary pill buttons, send button, active links, focus ring, rising-trend icon, **Reorder** category, "healthy" zone of the safety-stock gauge |
| 🟧 | **Coral** | `#FF6648` | Accent only, never text on light backgrounds (2.9:1): **Markdown** category, high-urgency dots, "understock" and "overstock" zones of the safety-stock gauge |
| 🟫 | **Cream** | `#FFE1D7` | Brand surface: compliance notice card, high-urgency badge fill, mono logo on navy |
| 🔷 | **Light blue** | `#84AFFB` | Secondary/decoration only: **Hold** category, medium-urgency dots |

### 2.2 Full token table

| Token (CSS var) | Hex | Use |
|---|---|---|
| `--color-canvas` | `#FCF7F5` | Page background, chip fill |
| `--color-paper` | `#FFFFFF` | Cards, inputs, scenario picker card, composers, tooltips |
| `--color-ink` | `#0A2150` | Headings, figures, dark cards, footer |
| `--color-charcoal` | `#171417` | Primary body text |
| `--color-graphite` | `#48413F` | Secondary text, descriptions |
| `--color-stone` | `#6B625E` | Helper text, captions, placeholders |
| `--color-field` | `#978A84` | Input / interactive borders only |
| `--color-hairline` | `#F1E2DB` | Card borders, dividers, chart gridlines |
| `--color-blue` | `#0259DD` | Action colour |
| `--color-blue-deep` | `#0148B8` | Hover state of blue buttons |
| `--color-sky` | `#84AFFB` | Support graphics |
| `--color-sky-tint` | `#E7EFFE` | Pill badge fill, loading pill, text selection |
| `--color-coral` | `#FF6648` | Accent |
| `--color-coral-ink` | `#B3341C` | The only coral-family **text** colour: warnings, declined, declining-trend icon |
| `--color-cream` | `#FFE1D7` | Brand card stock |
| `--color-blush` | `#FFF0EB` | Light cream wash: callouts |

### 2.3 Approved text/background pairings

✅ = passes AA for body text (≥ 4.5:1). ❌ = do not use for text on that surface.

| Foreground ↓ / Background → | White | Canvas | Blush | Cream | Sky-tint | Ink | Blue |
|---|---|---|---|---|---|---|---|
| Charcoal `#171417` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Graphite `#48413F` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Stone `#6B625E` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Ink `#0A2150` | ✅ | ✅ | ✅ | ✅ | ✅ | — | ❌ |
| Blue `#0259DD` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | — |
| Coral-ink `#B3341C` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Coral `#FF6648` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Light blue `#84AFFB` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| White | — | — | — | — | — | ✅ | ✅ |

**Rules:** Coral and Light blue are never text on light surfaces. Don't put Blue on Ink navy (2.6:1). Text on Cream: charcoal, graphite, stone, ink, or coral-ink only.

### 2.4 Colour usage rules

- Blue is for actions only — never body text, large fills, or headings.
- **One filled blue button per view.**
- Ink navy is the heading colour and the "weight" surface.
- Cream is card stock — a surface, never a border or text colour on light backgrounds.
- No cool greys, ever (no `#CCCCCC`, no Tailwind `gray/slate/zinc`).

### 2.5 Data / category palette

| Action | Colour | Hex | Why |
|---|---|---|---|
| **Reorder** | Blue | `#0259DD` | Positive action: buy more |
| **Markdown** | Coral | `#FF6648` | Act on stock: discount it |
| **Hold** | Light blue | `#84AFFB` | Calm, recessive: do nothing |
| **Drop** | Graphite | `#48413F` | Neutral dark: dead stock |

Urgency dots: **High = Coral** (on cream badge), **Medium = Light blue** (on sky-tint badge), **Low = Hairline** (on white outline badge). The word is always shown alongside the colour.

---

## 3. Typography

*(Unchanged from v1.)*

### 3.1 Font families

| Role | Google Font | CSS var | Weights |
|---|---|---|---|
| Display serif (headlines, quoted question) | **Newsreader** (opsz 6–72, roman + italic) | `--font-display` | 400 |
| Geometric sans (card titles, KPI figures) | **Inter Tight** | `--font-heading` | 400, 500 |
| Workhorse sans (body, buttons, labels) | **Inter** | `--font-sans` | 400, 500, 700 |

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;700&family=Inter+Tight:wght@400;500&family=Newsreader:ital,opsz,wght@0,6..72,400;1,6..72,400&display=swap" rel="stylesheet">
```

Body text: `font-feature-settings: 'ss01', 'ss04'`. Aligned numbers use tabular figures (`.num`).

### 3.2 Type scale

| Token | Size | Line height | Used for |
|---|---|---|---|
| `text-caption` | 12px | 1.4 | Badges, labels, chart ticks |
| `text-body-sm` | 14px | 1.6 | Card descriptions, secondary buttons |
| `text-body` | 16px | 1.6 | Default body, inputs |
| `text-subheading` | 20px | 1.4 | Card titles, recommendation sentence |
| `text-heading-sm` | 24px | 1.3 | Quoted user question (≥ sm) |
| `text-heading` | 32px | 1.2 | Section headlines (≥ md) |
| `text-heading-lg` | 40px | 1.1 | Executive finding headline (≥ md) |
| `text-display` | 48px | 1.05 | Methodology headline (≥ sm) |
| `text-display-lg` | 64px | 1.0 | Hero headline "ShelfIQ" (≥ sm) |

### 3.3 Utility classes

| Class | Spec |
|---|---|
| `.display` | Newsreader 400, letter-spacing −0.01em; `<em>` inside → italic |
| `.heading` | Inter Tight 500, letter-spacing −0.02em |
| `.figure` | Inter Tight 500, letter-spacing −0.04em, tabular numbers |
| `.eyebrow` | Inter 12px/700, UPPERCASE, letter-spacing 0.12em |
| `.num` | `font-variant-numeric: tabular-nums` |

### 3.4 The italic accent

One phrase per serif headline, same colour as the rest of the headline:

| Location | Headline |
|---|---|
| Dashboard hero | Know what to do, *SKU by SKU* |
| Product action list | What to do, *SKU by SKU* |
| Weeks-of-cover chart | Weeks of cover *vs* the reorder & markdown lines |
| Markdown ladder | Optimal markdown *ladder* |
| Compliance notice | Request *declined* under legal safeguards |
| Methodology | *Built* on the same formulas the engine runs |

The user's quoted question is set entirely in Newsreader italic, graphite, 20px (24px ≥ sm), wrapped in curly quotes " ".

---

## 4. Layout, spacing, shape & elevation

*(Unchanged from v1.)*

- **Container:** `max-width: 1200px`, centred. Side padding 16px (mobile) / 24px (≥640px) / 32px (≥1024px).
- **Base unit:** 8px. Steps: 8, 16, 24, 32, 40, 48, 56, 64, 80, 128.
- **Card padding:** 32px desktop / 24px tablet / 20px mobile. **Gaps between cards:** 16px mobile / 24px desktop.
- **Corner radius:** Cards/inputs/composer **16px**. Scenario-picker rows/inner metric-strip panels 12px. Pills/buttons/chips **full (100px)**. Never square or 4px on cards or inputs.
- **Shadows:** `--shadow-card: 0 4px 24px rgba(10,33,80,0.08)`; `--shadow-float: 0 8px 32px rgba(10,33,80,0.12)`. Always navy-tinted, never black. **Buttons never have shadows** — colour is the elevation.
- **Surfaces:** 0 Canvas `#FCF7F5` · 1 Paper `#FFFFFF` · 1b Cream `#FFE1D7` · 1c Blush `#FFF0EB` · 2 Ink `#0A2150`.

---

## 5. Components

### 5.1 Buttons

| Variant | Spec |
|---|---|
| **Primary (filled pill)** | 48px tall, Blue bg, white Inter 16/500, radius full, no shadow. Hover → Blue-deep. Disabled → 40% opacity. |
| **Secondary (outlined pill)** | 40px tall, white bg, Field border, Graphite Inter 14/500. Hover → border & text Ink. |
| **Ghost** | No bg/border, Graphite Inter 14/500, hover → Ink. |
| **Icon send button** | 40×40 circle, Blue bg, white arrow-up icon. `aria-label="Send"`. |

### 5.2 Composer (question input)

- Visible label above: "Ask ShelfIQ about markdowns, reorders, or stock" (Inter 14/500, Graphite).
- Container: white, 16px radius, Field border, padding 8px, input + Primary button side by side (stacked on mobile).
- Focus: border → Blue + 2px Blue ring at 20% opacity.
- Input: transparent, height 48px, Inter 16px Charcoal, placeholder Stone, max length 1000.
- Placeholder: *"e.g. How is SKU 31526 performing right now?"*

### 5.3 *(Removed)* View mode switch

v1 had a Track A/B switch; the previous revision repurposed it as a Simple/Dashboard switch. Neither exists in this shell — every answer always renders at full richness (§0, §6.2). The segmented-pill component itself (white bg, Hairline border, radius full, `role="group"`) isn't used anywhere else in this build, so it's dropped rather than kept unused.

### 5.4 Pill badge (tag)

Inter 12/500, padding 4px 10px, radius 16px. Optional 8px colour dot.

| Tone | Background | Text | Used for |
|---|---|---|---|
| `neutral` | Sky-tint | Ink | "Understood as" scope / horizon tags |
| `blue` | Sky-tint | Blue | Decision-type tag |
| `coral` | Cream | Coral-ink | High urgency, "couldn't factor in" items |
| `outline` | White + Hairline | Graphite | Action-type tag (with category dot), low urgency |

### 5.5 Cards

All 16px radius.

| Variant | Spec | Used for |
|---|---|---|
| `light` | White + Hairline border | Product/action cards, methodology accordion |
| `dark` | Ink bg, white text | First KPI card |
| `dashboard` | White + `--shadow-card` | Weeks-of-cover chart card, methodology container |
| `cream` | Cream bg | Compliance notice, error message |

### 5.6 KPI row — one per answer, always shown

Not a portfolio dashboard: these four numbers describe **only the products returned by the current answer**, recomputed client-side every time a new answer arrives and fully replaced (not appended to) when the next one does. Label them so that's clear.

- Row: 2 columns mobile/tablet, 4 on desktop. Card: `light` (first card `dark`, matching the reference's inverted-ink first tile), 16–24px padding.
- Label: Inter 14/500 Graphite (or Sky on the dark card) + category dot. Value: `.figure` 30–40px Ink (white on the dark card). Caption: Inter 14 Stone (Sky-tint/80% on the dark card).

| Card | Value | Caption |
|---|---|---|
| 1 (dark) | Count where `action === "MARKDOWN"` | "Markdowns required" |
| 2 | Count where `action === "REORDER"` | "Urgent reorders" — if ≥ 1, sub it with the lowest `weeks_of_cover` among them: "Stockout projected in {n} weeks" |
| 3 | **Capital at risk** — `Σ (units_on_hand × unit_cost)` over this answer's `MARKDOWN`-flagged items only, formatted as a plain thousands-grouped number | "Across {n} evaluated SKUs" |
| 4 | Count where `action === "HOLD"` | "Healthy full-price sell-through" |

**Card 3 is a real computed figure, not the reference's invented one.** `units_on_hand` and `unit_cost` are both genuine per-item API fields (§12.3); multiplying and summing them for the markdown-flagged subset of *this answer* is honest client-side arithmetic, unlike the reference's flat "£14,820" (no such total exists anywhere in the response). **Do not add a currency symbol unless the catalog's price columns confirm one** — check `fast_fashion_product_profiles.csv` before shipping; until confirmed, render the plain number (e.g. "14,820") rather than guessing "£" from the visual reference.

When an answer has zero items in a category, render "0," not an empty card — the layout stays 4 cards at all times.

### 5.7 Metric strip (inside product/action cards)

12px-radius grid, Hairline dividers, Canvas cell bg.

- Label: Inter 12/500 Stone. Value: `.heading` 20px Ink tabular; units Inter 14 Stone.
- Fields, in order: **Safety Stock Gauge** (spans full width — see §5.8/§7.2) · Units on hand · Velocity (u/wk) · Margin % · Trend (Blue ↗ rising / Coral-ink ↘ declining).
- **No "SIM" badge** — these figures are read directly from `fast_fashion_product_profiles.csv` via the engine, not simulated.

### 5.8 Safety Stock Gauge — *new component, replaces v1's plain Meter*

This is the chart the user asked for: for whichever product(s) an answer is about, show where its inventory sits relative to a safe range — not as a raw "weeks of cover" number, but as a coloured position on a gauge.

**What it's built from.** The engine (`README.md` §5, `ShelfIQ_V2.yml`'s "Scientific Calculation" nodes) already computes exactly this signal — it's not a new calculation, only a new picture of one that exists:

```
pressure_ratio = weeks_of_cover / weeks_remaining
```

| `pressure_ratio` | Meaning | Engine action | Gauge zone colour |
|---|---|---|---|
| `< 0.5` | Below safety stock — will likely stock out before the window ends | `REORDER` | Blue (same as the Reorder tag) |
| `0.5 – 1.5` | Healthy range | `HOLD` (unless the separate DROP rule fires — see note) | Light blue (same as the Hold tag) |
| `> 1.5` | Above safety stock — more inventory than the window can sell through | `MARKDOWN` | Coral (same as the Markdown tag) |

Zone boundaries in **weeks of cover**, for a given `weeks_remaining` (the horizon, default 6 unless the query implied another number — see `understood.weeks_remaining` / the engine's `weeks_left` parameter):

```
reorder_line  = 0.5 × weeks_remaining
markdown_line = 1.5 × weeks_remaining
```

> **DROP is not on this gauge.** `DROP` fires on a separate rule (low velocity + declining trend + low margin — `README.md` §5) that has nothing to do with weeks of cover; a DROP item can sit anywhere on the gauge. Render the gauge for every item regardless of its action, but don't try to make DROP a fourth gauge zone — flag it with the action pill only.

**Visual spec:**
- Track: full metric-strip width, 12px tall, radius full, three coloured zones drawn to scale (not evenly-thirded) from 0 up to `1.5 × weeks_remaining × 1.4` (so the markdown zone always has visible width even for very low-cover items).
- Zone fills at 70% opacity so the marker and ticks stay legible: Blue (0 → reorder_line), Light blue (reorder_line → markdown_line), Coral (markdown_line → end).
- Marker: 14px Ink diamond at the item's actual `weeks_of_cover` position, with a `.num` label above it, e.g. **"9.7 weeks"**.
- Reference ticks at `reorder_line` and `markdown_line`: 2px Ink marks with 12px Stone labels below, e.g. "3w reorder line", "9w markdown line" (values computed from the live `weeks_remaining`, not hardcoded).
- Legend beneath: three colour dots + labels "Reorder zone · Healthy zone · Markdown zone" (words, not colour alone).
- `role="img"`, `aria-label` e.g. *"Safety stock gauge: 9.7 weeks of cover, in the markdown zone (above the 9-week line)."*
- Compact variant (a **collapsed** action card, §6.2.2): same colours and marker, track height 8px, ticks and legend hidden — a title attribute carries the same text as the `aria-label`.
- Full variant (an **expanded** action card — the first one, by default): as specified above, ticks and legend visible, 40px tall.

### 5.9 Callouts

| Callout | Spec | Populated from |
|---|---|---|
| **Reasoning** (per card) | Inter 16px Graphite, below the recommendation | FUZZY: `actions[i].reasoning`. SPECIFIC: no per-item field exists — see §12.5, use the shared `explanation` block instead (§5.10). |
| **Expected impact** | Blush bg, 12px radius, Coral circle icon, label "Expected impact" | FUZZY: `actions[i].expected_impact` (required field). **Omit this callout entirely on SPECIFIC-path cards** — that field doesn't exist there. |
| **Compliance strip** | Blush bg, Coral-ink shield icon + "EU competition & consumer-law guard is always on." + chip "Try a compliance test →" | Static copy; chip sends a known price-fixing test query |
| **Error** | `cream` card, Coral-ink text, alert icon, `role="alert"` | SPECIFIC: `message` when `status: "error"`. Server/network failure: the proxy's own error text (`server/index.js`) |
| **Loading pill** | Sky-tint bg, Blue text "Analysing…", spinner, `aria-live="polite"` | Local request state |

### 5.10 Explanation block (shared, below the cards)

Both response shapes carry one shared narrative string in addition to (or instead of) per-card reasoning:

- SPECIFIC: `explanation` — one string; the narrator writes one paragraph per product, blank-line separated, each starting with the product name (`README.md` §9). Render as a single `Inter 16px Graphite` block below all cards. *Optional enhancement, not required for v1:* split on blank lines and match each paragraph to its card by the leading product name, falling back to the single shared block if a paragraph can't be matched.
- FUZZY: `summary` — one string, render the same way.
- When `actions`/`products` is empty (a decline, an off-topic reply, or "no action needed"): skip the cards entirely and render `summary` (or `message`, or `no_action_reason`) as a single message bubble — this is what `normalize.ts`'s `{ kind: 'message' }` result is for.

### 5.11 Methodology accordion

Rows sit in one `light` card. Hairline divider between rows, chevron rotates 180° when open, first row open by default, `aria-expanded` + `aria-controls`.

| Row | Body |
|---|---|
| Newsvendor critical ratio | The reorder quantity isn't the sales forecast — it's set where the cost of running out equals the cost of leftover stock (`critical_ratio = cu / (cu + co)`). This is what `optimal_order_quantity` is computed from. |
| Markdown pacing | Deeper, earlier discounts for items further past their markdown line — the same "sell down as time runs out" logic behind classic markdown-optimization models, applied here as fixed discount ladders keyed to how far over the line an item is. |
| Article 101/102 TFEU | ShelfIQ recommends prices only from your own inventory data. It declines requests to coordinate with or undercut a competitor's price. |
| EU Omnibus Directive 2019/2161 | Any advertised discount must reference a real recent price. ShelfIQ declines requests to invent a fake "was" price. |

*(Removed from v1: WRAP CO₂e row — Track B only, not implemented here.)*

### 5.12 Scenario picker card — hero, right column

*(Replaces v1's sidebar "Try asking" list, which itself replaced v1's bottom-pinned chat-widget chip row — see §0. Same three validated questions, new position: the paper card in the hero's right column, styled after the visual reference's "TRY A SCENARIO" card.)*

- `light`-family card, but on the reference's slightly heavier shadow (`--shadow-float`-equivalent, since it visually anchors the hero), 24px padding, radius 16px.
- Header row: `.eyebrow` "TRY A SCENARIO" (Stone) + a right-aligned small pill showing the catalog's data-as-of date (from the answer's `data_as_of` field when one exists; omit the pill entirely rather than hardcoding a date if the field is absent).
- **Three full-width rows**, each: a numbered circle (1/2/3, Sky-tint bg, Ink text) + a small `.eyebrow` category tag (Coral-ink "MARKDOWN" / Blue "REORDER" / Stone "SPECIFIC PRODUCT") + the question text (Inter 14/500 Charcoal, 2-line clamp) + a trailing circular arrow icon that fills Blue on hover. 12–14px padding, radius 12px, Canvas bg, hover → Sky-tint at low opacity.
- Below the rows, the same Blush compliance strip as §5.9's "Compliance strip" callout, with its "Try a compliance test →" link/chip being the fourth, visually-distinct scenario.

Row labels and the exact query each one sends, taken directly from `ShelfIQ_V2.yml`'s `suggested_questions` (the same prompts the Dify app itself suggests, so the UI never promises something the backend hasn't already validated):

| Row | Category tag | Query sent |
|---|---|---|
| 1 | Markdown | Which products are so overstocked that I should markdown them before the season ends? |
| 2 | Reorder | Which products need urgent reorders? |
| 3 | Specific Product | How is SKU 31526, the ZIP THROUGH HOODIE BLACK, performing right now? |
| Compliance strip's link | *(styled as the callout, not a numbered row)* | Can we raise the price of the ZIP THROUGH HOODIE BLACK this week, so next week's markdown looks like a bigger discount? |

*(Removed from v1: the Track A/Track B–specific chip sets — "Winter line only," the Track B holiday/weather/weekday chips. Those now live only as follow-up chips, §5.15.)*

**Click-to-ask contract — this is a hard behavioural requirement, not just a visual spec:** clicking any row (or the compliance link) does exactly what typing that same text into the hero composer (§5.2) and pressing Send does — same function, same code path — **it submits immediately.** It is never implemented as "fill the input and stop," which is what the visual reference's own `onclick` handlers do (they only set `copilot-hero-input.value`) — that's a known flaw in the reference, not a spec to copy. Submitting replaces the Analysis section's content with the new answer and scrolls to `#analysis` (§6.2).

**Scenario-parameters panel** (collapsible, sits below the hero composer, above this card): planning-horizon and max-discount sliders exactly as the reference shows them, each updating only its own on-screen value label (`oninput`, no submit). **Drop the reference's "Re-run with these parameters" button and "Stochastic dynamic solver active" tag** — `POST /api/query` has no parameter for either slider yet (§0), so a working-looking re-run button or an "active solver" claim would both be dishonest. Replace that row with one static caption: "Horizon and discount cap are set by the engine per question today — adjustable parameters are a planned enhancement."

### 5.13 Skeleton

Hairline bg, animate-pulse, 12–16px radius. 4 KPI skeletons + 2 action-card skeletons, in place of the Analysis section's content while a request is in flight. Container gets `aria-busy="true"`.

### 5.14 Logo

Use the SVG geometry from the brand asset files, never re-drawn. Full colour, left-aligned in the sticky header (§6.1).

### 5.15 Pinned follow-up composer

*(Replaces the prior revision's bottom-pinned main-pane composer — same component, now the page's only composer other than the hero's, and only visible once an answer exists.)*

- `position: sticky; bottom: 24px`, `max-width: 680px`, centred, `z-index` above page content.
- Card: Paper at 95% opacity + `backdrop-filter: blur(16px)`, radius 16px, `--shadow-float`, 12–16px padding.
- Row 1: 32px circular Blue avatar (a simple icon, not the brand mark — it represents "ask ShelfIQ," not the logo) + text input (placeholder *"Ask a follow-up or change a constraint…"*) + 40px circular Blue send button.
- Row 2: up to 3 suggestion chips (pill, Canvas bg, Stone text, hover → Sky-tint/Ink), horizontally scrollable on overflow, no visible scrollbar. Chip copy is illustrative product-scope shortcuts (e.g. "What needs a reorder?", "What should I drop?") — **not** a fixed validated list like the scenario-picker card, so keep chip copy short and generic rather than inventing specific numbers or SKUs.
- **Same click-to-ask contract as §5.12**: typing + Send, or tapping a chip, both submit immediately through the one shared `askQuestion` function (§12.4) and replace the Analysis section.
- Hidden entirely before the first answer exists (the hero composer is the only input at that point) — appears once `#analysis` has content, and never disappears again for the rest of the session.

---

## 6. Page structure

**This section replaced its prior self entirely** (see §0's two most recent changelog rows) to match a concrete HTML reference the client supplied, after the sidebar/multi-turn shell built from the previous revision was explicitly set aside in favour of the reference's own single-page structure. The structure below is now the authoritative one; nothing else in this guide changes.

### 6.1 Overall shape — one scrolling page, "latest result only"

A single page, normal document scroll (not the previous revision's fixed-viewport two-pane shell). Five regions, top to bottom:

1. **Sticky header** — `position: sticky; top: 0`, Canvas at 90% opacity + `backdrop-filter: blur(16px)`, hairline shadow. One row: logo (§5.14) left · centre reserved for a future nav (empty for now — don't invent nav items) · a 32px circular avatar placeholder, right.
2. **Hero** (`id="top"`) — display headline "ShelfIQ," the composer (§5.2), the scenario-parameters panel, and the scenario picker card (§5.12) in a two-column layout on desktop (composer side ~7/12, picker card ~5/12), stacking to one column below `lg`. Always present, at the top of the page, whether or not an answer exists yet.
3. **Analysis** (`id="analysis"`) — the one current answer, in full (§6.2). **Empty/hidden before the first question is asked** — there's no placeholder or sample data to show here, since showing fabricated "example" results would violate §1's "never claim more than the API returns" the same way the reference's baked-in mock numbers do. Once the first answer arrives, this section populates and the page scrolls to it; every later question **replaces** this section's content in place — it does not add a second Analysis block.
4. **Methodology** (`id="methodology"`) — the Methodology accordion (§5.11), always present (not only after a query), so "Try a compliance test →" (§5.12, §5.9) has a real anchor to jump to even before the visitor has asked anything.
5. **Pinned follow-up composer** (§5.15) — appears once `#analysis` has content; absent before that.
6. **Footer** — page-width, Ink bg (§6.5).

### 6.2 The "latest result" contract — how a question becomes the Analysis section

**This is a hard behavioural requirement, stated once here as the single source of truth (§5.12 and §5.15 cross-reference it):** every question — typed into the hero composer, clicked from the scenario-picker card, typed or tapped as a chip in the follow-up composer — goes through the same function and the same rendering path:

1. Fire the query immediately (no confirmation step); show the skeleton (§5.13) in `#analysis` in place of whatever was there.
2. Scroll to `#analysis`.
3. When the response arrives, render it into `#analysis`, **replacing** everything that was there before — the previous question's quote, headline, KPIs, cards, and charts are gone, not appended above or below. There is exactly one answer in the DOM at any time.
4. The follow-up composer (§5.15) appears (if this was the first answer) or stays as-is (if not).

There is no action anywhere in this app that keeps two answers on screen at once, and no history the visitor can scroll back to — this is the explicit, chosen trade-off of this shell (§0). If a later revision adds history back, it will be a new, separately-specified feature, not an emergent side effect of this section.

**Rendering the answer**, by response shape:
- decline/notice (`is_notice: true`, or an empty `actions`/`products` array) → the compliance/decline notice (§6.3), replacing the KPI row and action cards entirely — no empty "0 markdowns / 0 reorders" KPI row above a decline notice.
- `status: "error"` → the error callout (§5.9) in the same place.
- otherwise, in this order:
  1. Quoted question (§6.2.1 below) — the literal text that was submitted.
  2. Executive finding headline — **not freely composed**; render the `recommendation` string of the single highest-urgency item (or, for FUZZY, `actions[0].recommendation`) as this headline, at `text-headline-lg`. This is real API text reused at larger size, not an invented executive summary — don't paraphrase or combine multiple items' recommendations into one sentence.
  3. KPI row (§5.6).
  4. Action cards (§6.2.2), one per item, most-urgent first.
  5. Weeks-of-cover bar chart (§7.3), when the answer has ≥ 2 items.
  6. Shared explanation/summary paragraph (§5.10) — render this below the charts, as the closing narrative, rather than only inline per-card.

#### 6.2.1 Quoted question

Newsreader italic, 20/24px, Graphite, wrapped in curly quotes " " — the exact string that was submitted (composer text, or the scenario/chip's mapped query), never truncated or reworded.

#### 6.2.2 Action cards

One `light` card per item (§5.5), first item expanded by default, the rest collapsed — a "Hide details −" / "Details +" ghost button (top right of the card) toggles between them, matching the reference:

- **Header** (always visible, both states): numbered circle (position in the list) · product name · SKU · action pill (§2.5) · urgency pill (§2.4) · one-line `recommendation` excerpt.
- **Metric strip** (§5.7), always visible: Safety Stock Gauge (compact when collapsed, full when expanded) + on-hand, velocity, margin %, trend — **no SIM badge** (§5.7, unchanged rule).
- **Expanded-only content**, two-column on desktop:
  - Left: the markdown ladder chart (§7.4) when `action === "MARKDOWN"` and `markdown_schedule` is non-null; otherwise, for `REORDER` items, a plain restatement of `optimal_order_quantity` ("Reorder {n} units") in the same visual slot — don't leave the column empty, and don't force a chart where the field doesn't exist.
  - Right: **Reasoning + Expected-impact callouts** (§5.9) — FUZZY responses only (`actions[i].reasoning`, `.expected_impact`); on SPECIFIC-path cards, **this column is omitted entirely** rather than showing a fabricated "Simulation Parameters" panel (§0) — the shared explanation block (step 6 above) is that path's only narrative text (§12.5).
  - A small "Methodology →" link in this column scrolls to `#methodology` and opens the accordion's first (Newsvendor) row — this is how the reference's citation-style panel is served honestly, using the one methodology fact that's actually true of this engine, instead of an invented paper citation (§0).

### 6.3 Compliance / decline notice

Renders in `#analysis` in place of the headline/KPI/cards, when `is_notice: true` (covers UNSAFE and OFF_TOPIC — both use the same response shape, see `ShelfIQ_V2.yml`'s "UNSAFE Answer" / "OFF_TOPIC Answer" nodes):

- Quoted question (§6.2.1) still renders above it, unchanged.
- `cream` card. White pill with Coral-ink shield icon + "Regulatory compliance guard" (only for `decision_type: "refusal"` — for `decision_type: "off_topic"` use a plain ink "i" icon and drop the compliance framing, since it isn't a compliance decline).
- Headline "Request *declined* under legal safeguards" (refusal) or "Outside ShelfIQ's scope" (off-topic).
- Body = `summary`, 16px Charcoal.
- `role="alert"`.

### 6.4 Accessibility of this structure specifically

- `#analysis` is `aria-live="polite"` so replacing its content is announced, and gets `aria-busy="true"` while the skeleton is showing (§5.13).
- The scenario-picker card is a `<ul>` of `<button>` rows inside `<nav aria-label="Try a scenario">` (not links — they trigger an action, they don't navigate); same pattern for the follow-up composer's chips.
- "Skip to content" (first focusable element on the page) targets the hero composer's input.
- In-page anchor links (`#top`, `#analysis`, `#methodology` in the footer nav, §6.5) use `scroll-behavior: smooth` unless `prefers-reduced-motion` is set.

### 6.5 Footer

Full page width, Ink bg, white/Sky-tint text — the one page-level footer, since this shell has a real page bottom (unlike the previous sidebar revision):

- Row 1: logo mark + wordmark, left; in-page nav links ("Back to top," "Methodology") right.
- Row 2, two columns: a short "How this works" paragraph naming the Newsvendor critical ratio and markdown-pacing logic (real, matches §5.11's accordion — **not** the reference's "Gallego & van Ryzin (1994)" / "Arrow, Harris & Marschak (1951)" citations, which name a dynamic-pricing model this engine doesn't run, §0) · a short "Data" paragraph: "Catalog data: `fast_fashion_product_profiles.csv`, ~{n} SKUs. Figures shown are read directly from that catalog, not simulated."
- Row 3: `.eyebrow` (Coral-ink or Cream, not the reference's bold Cream "compliant" claim): "COMPETITION & CONSUMER-LAW GUARD: ARTICLES 101/102 TFEU · EU OMNIBUS DIRECTIVE 2019/2161" — **do not** restore the reference's "EU ARTIFICIAL INTELLIGENCE ACT & GDPR ALGORITHMIC SAFEGUARDS COMPLIANT · HIGH-RISK DECISION SUPPORT REGIME" line (§0) — plus a copyright line.

---

## 7. Charts & data visualisation

Library: **Recharts 3** (already a natural fit for the existing React/Vite frontend). Every chart:
1. has direct value labels on the marks,
2. has a visible legend,
3. has a screen-reader data table (`<div class="sr-only"><table>…</table></div>` — `sr-only` on the wrapper div, not the table).

### 7.1 Shared chart styling

| Element | Spec |
|---|---|
| Axis ticks | Stone `#6B625E`, 12px; no axis/tick lines |
| Gridlines | Hairline `#F1E2DB`, horizontal only |
| Value labels | Ink, 12px/500, above the mark |
| Tooltip | White, Hairline border, 12px radius, `--shadow-float`, 13px Charcoal |
| Bars | Rounded top corners 6px, `maxBarSize` 28 |
| Reference lines | 4-4 dashed, no inline labels (explain in the legend) |
| Animation | Off — `isAnimationActive={false}` |

### 7.2 Safety Stock Gauge — full spec

See §5.8 for the logic and zone thresholds. Implementation notes:

- Build as a single horizontal stacked bar (three `<rect>`/segments) rather than a Recharts chart type — it's simpler and matches the "linear gauge" shape more precisely than forcing it into a bar/area chart.
- Unique `id` per gauge instance for any gradient/clip-path defs (`gauge-${sku}`), same rule as v1's ladder chart gradients (v1 §12.5 pitfall #3 — still applies).
- Heights: compact 8px (collapsed action card), full 40px incl. ticks/legend (expanded action card, §6.2.2).

### 7.3 Weeks-of-cover bar chart (≥ 2 items in the answer)

x = SKU (or short product name), y = weeks of cover. Bars coloured by `action` (§2.5 palette). Reference lines: the answer's `reorder_line` and `markdown_line` (Ink dashed and Coral-ink dashed respectively — computed the same way as the gauge, §5.8, so the two visuals never disagree). Legend: action categories present + "Reorder line" + "Markdown line."

### 7.4 Markdown ladder (per MARKDOWN card with a schedule)

- `AreaChart`, `type="stepAfter"`. Stroke Ink 2px, dots Ink r4 with white stroke.
- Fill: horizontal gradient Light blue → Coral, left to right (deepening urgency over the ladder).
- Y domain 0–60%, ticks 0/20/40/60, unit "%". Data points = `markdown_schedule` array (one value per week).
- Label only where the discount changes: "Full price," "−15%," "−40%" (true minus sign).
- Height 208px, inside a `dashboard` card titled "Markdown ladder for this SKU."

---

## 8. States, interaction & motion

| State | Treatment |
|---|---|
| Hover: primary | Blue → Blue-deep |
| Hover: secondary/chip | Border → Ink |
| Focus (keyboard) | `:focus-visible` → 2px Blue outline, 2px offset. Composer uses Blue border + 20% Blue ring instead. **Never remove focus styles.** |
| Disabled | 40% opacity, `not-allowed` cursor |
| Loading | Composer button spinner; "Analysing…" pill; skeletons (§5.13) |
| Empty | `light` card, 16px Graphite: "No actions matched this request." |
| Error | Cream error callout; stale results cleared first |
| Expanded (accordion / action card) | Chevron rotates 180°; card's "Details +" button becomes "Hide details −" (§6.2.2) |

**Motion:** colour transitions ~150ms, chevron 200ms, gauge marker position 500ms ease-out, scroll-to-`#analysis` smooth-scroll ~400ms. `prefers-reduced-motion: reduce` turns off all animation and smooth scrolling (jumps instead).

**Request sequencing:** a newer request always supersedes an older in-flight one — same rule as v1, still applies since the hero composer and follow-up composer can both fire a request, and a scenario/chip click can arrive while a previous request is in flight.

*(Removed from v1: the "switches Track automatically if the answer mentions perishables" behaviour — there's only one track now. Removed from the prior revision: the view-mode switch's "selected" state — there's no view-mode switch in this shell, §0.)*

---

## 9. Accessibility requirements

- [ ] All text meets the §2.3 pairings.
- [ ] Visible `<label>` for both composers (hero + follow-up); `aria-label` on the icon-only send buttons.
- [ ] "Skip to content" targets the hero composer's input (§6.4).
- [ ] Scenario picker and follow-up chips are `<nav aria-label="Try a scenario">` / a labelled chip group with `<button>` rows, not links (§6.4).
- [ ] Heading order: `h1` (hero "ShelfIQ") → `h2` Analysis section headline → `h3` action card titles.
- [ ] Accordion rows use `aria-expanded` + `aria-controls`; action-card "Details" toggle uses the same pattern.
- [ ] `#analysis` is `aria-live="polite"` and `aria-busy` while loading; errors and the compliance notice use `role="alert"` (§6.4).
- [ ] Every chart (gauge, bar chart, ladder) has value labels + legend + `sr-only` data table. The gauge additionally has `role="img"` + a full-sentence `aria-label` (§5.8).
- [ ] Colour is never the only signal — action, urgency, and gauge zone always carry a word.
- [ ] Touch targets ≥ 40px.
- [ ] `prefers-reduced-motion` respected.

---

## 10. Responsive behaviour

Breakpoints: `sm 640px`, `md 768px`, `lg 1024px`.

| Area | < 640px | 640–1023px | ≥ 1024px |
|---|---|---|---|
| Hero | Single column: composer, then scenario-parameters panel, then scenario picker card | same | Two columns: composer ~7/12 left, scenario picker card ~5/12 right |
| KPI row | 2×2, figure 30px | 2×2, figure 40px | 4 across |
| Action card | Gauge full width, metric strip 2 cols, expanded content stacks to one column | same, metric strip 2–4 cols | Metric strip up to 5 cols, expanded content two columns |
| Weeks-of-cover chart | Horizontal scroll if > 4 SKUs | same | Full width |
| Pinned follow-up composer | Full viewport width minus 16px margins | 680px max-width, centred | same |
| Methodology | Full-width accordion, inline in page flow | same | same |

**Hard rule:** no horizontal page scroll at 375px wide. Minimum side gutter 16px.

---

## 11. Copy & content rules

- Currency symbols are literal characters (`€`, `£`) — never HTML entities.
- True minus sign "−" for discounts, en dash "–" for ranges, curly quotes " ".
- Sentence case everywhere except eyebrows (uppercase via CSS).
- Don't centre-align body copy; body line-height ≥ 1.5.
- Composer placeholder: *"e.g. How is SKU 31526 performing right now?"*
- Scenario picker rows: use the exact copy in §5.12's table — it's the copy already validated inside the Dify app itself, so the UI and the backend's own suggestions never drift apart. Follow-up chips (§5.15) are not from that validated list and may use shorter, more generic phrasing.

---

## 12. Implementation reference

### 12.1 Tailwind v4 theme + global CSS

Unchanged from v1 — reuse `src/index.css`'s `@theme` block verbatim (colour tokens, font vars, type scale, radius, shadow tokens) if migrating to Tailwind; the current `frontend/src/App.css`/`index.css` in this repo can adopt the same custom-property names without a framework change.

```css
:root {
  --color-canvas: #fcf7f5; --color-paper: #ffffff; --color-ink: #0a2150;
  --color-charcoal: #171417; --color-graphite: #48413f; --color-stone: #6b625e;
  --color-field: #978a84; --color-hairline: #f1e2db;
  --color-blue: #0259dd; --color-blue-deep: #0148b8;
  --color-sky: #84affb; --color-sky-tint: #e7effe;
  --color-coral: #ff6648; --color-coral-ink: #b3341c;
  --color-cream: #ffe1d7; --color-blush: #fff0eb;
  --font-display: 'Newsreader', Georgia, serif;
  --font-heading: 'Inter Tight', Inter, ui-sans-serif, system-ui, sans-serif;
  --font-sans: 'Inter', ui-sans-serif, system-ui, sans-serif;
  --radius-card: 16px;
  --shadow-card: 0 4px 24px rgba(10, 33, 80, 0.08);
  --shadow-float: 0 8px 32px rgba(10, 33, 80, 0.12);
}
```

### 12.2 Colour + action-colour constants (TypeScript)

```ts
export const C = {
  ink: '#0A2150', charcoal: '#171417', graphite: '#48413F', stone: '#6B625E',
  hairline: '#F1E2DB', canvas: '#FCF7F5',
  blue: '#0259DD', blueDeep: '#0148B8', sky: '#84AFFB', skyTint: '#E7EFFE',
  coral: '#FF6648', coralInk: '#B3341C', cream: '#FFE1D7', blush: '#FFF0EB', white: '#FFFFFF',
};

export const ACTION_COLOR = { REORDER: C.blue, MARKDOWN: C.coral, HOLD: C.sky, DROP: C.graphite };

// Safety Stock Gauge zone colours — same hues as ACTION_COLOR, by design (§5.8)
export const GAUGE_ZONE_COLOR = { reorder: C.blue, healthy: C.sky, markdown: C.coral };
```

*(This repo's `frontend/src/normalize.ts` already exports an equivalent `ACTION_COLORS` map — keep the two in sync if both exist, or consolidate into one.)*

### 12.3 API → UI field mapping (the actual functional wiring)

**SPECIFIC path** (`/api/query` → `answerJson` when it has a `products` array):

| API field | UI element |
|---|---|
| `status` | `"error"` → error callout instead of cards |
| `products[i].sku`, `.product_name`, `.category` | Card header |
| `products[i].action` | Action pill colour + label (§2.5) |
| `products[i].urgency` | Urgency pill |
| `products[i].recommendation` | Card body, 20px/500 Charcoal |
| `products[i].weeks_of_cover`, and the answer's horizon (default 6, or `understood.weeks_remaining` if present) | Safety Stock Gauge position + zone lines (§5.8) |
| `products[i].units_on_hand`, `.weekly_velocity`, `.margin_pct`, `.trend` | Metric strip |
| `products[i].markdown_schedule` | Markdown ladder chart, if non-null and action is MARKDOWN |
| `products[i].optimal_order_quantity` | Shown as a line in the card body when action is REORDER, and in the expanded card's left column when there's no ladder chart to show instead (§6.2.2) |
| `products[i].units_on_hand × .unit_cost`, summed across `action === "MARKDOWN"` items | KPI row card 3, "Capital at risk" (§5.6) — client-side arithmetic on two real fields, not a returned field itself |
| `products[0].recommendation` (highest-urgency item) | Analysis section's executive finding headline (§6.2) |
| `explanation` | Shared explanation block (§5.10) — **no per-card reasoning/expected-impact on this path** |

**FUZZY path** (`answerJson` has an `actions` array):

| API field | UI element |
|---|---|
| `understood.decision_type`, `.product_scope`, `.weeks_remaining` | "Understood as" tags |
| `understood.constraints` | Additional neutral tags |
| `understood.unhandled` | "I couldn't factor in:" list |
| `is_notice: true` (with `actions: []`) | Skip cards → compliance/decline notice (§6.3) or plain message bubble |
| `actions[i].sku/.product_name/.action/.urgency/.recommendation` | Same as SPECIFIC |
| `actions[i].reasoning` | Reasoning callout (§5.9) — **present here, absent on SPECIFIC** |
| `actions[i].expected_impact` | Expected-impact callout (§5.9) — **present here, absent on SPECIFIC** |
| `actions[i].key_metrics.*` | Metric strip + gauge (same fields, nested one level deeper than SPECIFIC's `products[i]`) |
| `actions[i].key_metrics.units_on_hand × .unit_cost`, summed across `action === "MARKDOWN"` items | KPI row card 3, "Capital at risk" — same computation as the SPECIFIC path, just reading from the nested `key_metrics` object |
| `actions[0].recommendation` (highest-urgency item) | Analysis section's executive finding headline — same rule as the SPECIFIC path |
| `summary`, `no_action_reason` | Shared explanation block / empty-state message |

### 12.4 File map (this repo)

| File | Contains |
|---|---|
| `frontend/src/normalize.ts` | Response-shape parsing — already implements the two §12.3 tables; extend it to also return `weeksRemaining`, the two gauge zone lines, and the computed `capitalAtRisk` total per answer, rather than recomputing them in the component layer |
| `frontend/src/components/SafetyStockGauge.tsx` | Implements §5.8/§7.2 — unchanged by this shell revision |
| `frontend/src/components/HeroComposer.tsx` | **New** — the hero's headline, composer (§5.2), and scenario-parameters panel (inert sliders, §5.12) |
| `frontend/src/components/ScenarioPicker.tsx` | **New**, replaces the removed `Sidebar.tsx` — the "TRY A SCENARIO" card (§5.12). Takes an `onAsk(query: string)` callback — the same function the hero composer's and follow-up composer's submit handlers call, so the click-to-ask contract (§6.2) is enforced by construction rather than by multiple components trying to stay in sync |
| `frontend/src/components/AnalysisSection.tsx` | **New** — `#analysis` (§6.2): quoted question, executive headline, KPI row, action cards, weeks-of-cover chart, explanation block. Renders nothing before the first answer; fully re-renders (not appends) on each new answer |
| `frontend/src/components/ActionCard.tsx` | **New**, replaces `ProductCard.tsx` — one card per item (§6.2.2), with its own expand/collapse state, no `variant` prop needed since there's only one density now |
| `frontend/src/components/WeeksOfCoverChart.tsx`, `MarkdownLadderChart.tsx` | Implement §7.3, §7.4 — unchanged by this shell revision, now always rendered (no Dashboard-mode gate) |
| `frontend/src/components/FollowUpComposer.tsx` | **New** — the pinned floating composer + chips (§5.15). Same `onAsk` callback pattern as `ScenarioPicker.tsx` |
| `frontend/src/components/ViewModeSwitch.tsx` | **Delete** — no view-mode switch in this shell (§0) |
| `frontend/src/App.tsx` | Rebuild as the single-page shell (§6.1): header, `<HeroComposer onAsk={askQuestion} />`, `<ScenarioPicker onAsk={askQuestion} />`, `<AnalysisSection answer={currentAnswer} loading={loading} />`, Methodology accordion, `<FollowUpComposer onAsk={askQuestion} />` (rendered only once `currentAnswer` exists), footer. `askQuestion(text)` is the ONE function that calls the API and **sets** `currentAnswer` (not appends to a list) — every composer and every clickable row calls it, nothing else triggers a query |
| `server/index.js` | No changes needed for this guide — already forwards `{ query }` to Dify and returns `{ answerText, answerJson }` |

### 12.5 Known gap to flag, not silently patch over

SPECIFIC-path cards have no per-item `reasoning`/`expected_impact` — only FUZZY does. Don't invent placeholder text for the missing fields on SPECIFIC cards; the shared `explanation` block is the only narrative text that path provides. If per-item reasoning on SPECIFIC responses is wanted later, that's a backend change (the Dify chatflow's SPECIFIC EXPLAINER node would need to emit it), not something to fake in the frontend.

---

## 13. Do's and don'ts

### Do
- ✅ Use the Safety Stock Gauge's three zone colours consistently with the action-pill colours (Blue/Light blue/Coral) — they're describing the same underlying number.
- ✅ Compute gauge zone lines from the *actual* `weeks_remaining` for that answer, not a hardcoded "6."
- ✅ Label the KPI row as describing "this answer," not a portfolio.
- ✅ Route every question — typed into either composer, clicked from the scenario picker, or tapped as a follow-up chip — through one shared `askQuestion` function, so the Analysis section is always fully replaced by the new answer, never appended to (§6.2, §12.4).
- ✅ Keep the Analysis section empty before the first question — no sample/placeholder data (§6.1).
- ✅ Compute "Capital at risk" from real `units_on_hand × unit_cost` (§5.6, §12.3) — it's honest arithmetic on real fields, unlike the reference's flat total.
- ✅ If a spec item (especially the gauge) proves genuinely hard to build, say so explicitly and ask before shipping a simplified placeholder. A silently simplified component is the specific failure the earlier shell revision existed to prevent (§0), and that discipline still applies here.

### Don't
- ❌ Add a SIM badge, or otherwise imply catalog figures are simulated — they're real CSV data in this build.
- ❌ Render an expected-impact or reasoning callout on a SPECIFIC-path card — the field doesn't exist there.
- ❌ Claim EU AI Act / GDPR "high-risk decision support" compliance in the footer or anywhere else, however prominently the visual reference displays it — only the two implemented guards (§0, §6.5) are real.
- ❌ Add a price-elasticity field, a "Governing Literature" citation, or any other evidence-panel field the reference shows but the API doesn't return — not in the API response (§0).
- ❌ Add a fourth "DROP" zone to the gauge — DROP isn't a cover-based signal (§5.8).
- ❌ Wire the scenario-parameters sliders or a "re-run" button to anything — `/api/query` has no parameter for them yet; keep them honestly inert (§5.12).
- ❌ Implement a scenario-picker row or follow-up chip as "fill the input and stop" — it must submit immediately, unlike the visual reference's own `onclick` handlers (§5.12, §5.15, §6.2).
- ❌ Show two answers on screen at once, or keep any trace of a previous answer once a new one has rendered — there is no thread and no history in this shell (§6.2).
- ❌ Reintroduce the sidebar or the Simple/Dashboard view-mode switch — both were explicitly replaced by this revision (§0).

---

## 14. QA checklist before release

**Visual**
- [ ] Only Blue is used for filled action buttons.
- [ ] Every divider/border is the warm Hairline; no cool grey anywhere.
- [ ] Safety Stock Gauge zone colours match the action-pill palette exactly.
- [ ] `#analysis` is empty/hidden on first page load — no placeholder KPIs or sample cards.
- [ ] No fabricated fields appear anywhere: no price elasticity, no "Governing Literature"/academic citation panel, no SIM badges, no EU AI Act/GDPR compliance claim (§0, §13).

**Functional**
- [ ] Type a question in the hero composer and submit, IN THE RUNNING APP: `#analysis` populates and the page scrolls to it.
- [ ] Click each scenario-picker row, one at a time: each submits immediately (no extra click needed) and **replaces** `#analysis`'s content with the new answer — the previous answer's headline/cards/charts are gone, not appended.
- [ ] Click the compliance-scenario link: renders the cream decline notice in `#analysis`, not action cards.
- [ ] Once the first answer renders, the pinned follow-up composer appears and stays present through every subsequent answer.
- [ ] Type in, or tap a chip on, the follow-up composer: same replace-in-place behaviour as the scenario picker.
- [ ] SPECIFIC responses render their action cards with no reasoning/expected-impact column; FUZZY responses render it.
- [ ] `is_notice: true` responses (both `refusal` and `off_topic`) skip the KPI row and action cards entirely.
- [ ] Markdown ladder chart appears only when `action === "MARKDOWN"` **and** `markdown_schedule` is non-null; REORDER items show the optimal-order-quantity line in that slot instead.
- [ ] Weeks-of-cover bar chart appears only when an answer has ≥ 2 items.
- [ ] KPI card 3 ("Capital at risk") shows a number computed from `units_on_hand × unit_cost` for this answer's markdown items — verify it changes between two different answers rather than staying fixed.
- [ ] Gauge zone lines change when a query implies a different horizon than 6 weeks (check via `understood.weeks_remaining`).
- [ ] The Safety Stock Gauge renders as three distinct coloured zones with a positioned marker — not a single-colour progress bar (the most likely component to get quietly simplified, per §0).

**Responsive & accessibility**
- [ ] No horizontal scroll at 375px / 768px / 1024px / 1440px, either mode.
- [ ] Keyboard-only pass: skip link, visible focus everywhere, logical tab order.
- [ ] Screen reader announces "Analysing…," errors, the compliance notice, and each gauge's full-sentence label.
- [ ] `prefers-reduced-motion` disables gauge marker animation and smooth scroll.
