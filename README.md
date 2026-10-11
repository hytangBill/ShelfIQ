# ShelfIQ

**A B2B retail merchandising copilot.** Ask it about a SKU, a markdown, or a reorder, and it answers
from your catalog using management science — Newsvendor critical ratios and Gallego & van Ryzin
markdown ladders — not from a language model's guess.

![status](https://img.shields.io/badge/status-working%20prototype-0259DD)
![stack](https://img.shields.io/badge/stack-React%20·%20Vite%20·%20Express%20·%20Dify-444)
![tests](https://img.shields.io/badge/automated%20tests-none%20yet-B3341C)
![license](https://img.shields.io/badge/license-unlicensed-888)

---

## Status at a glance

The app **works end to end today**: you can run it locally, ask a question, and get a real answer
rendered as action cards. It is a working prototype, not a hardened product.

| Area | State | Notes |
|---|---|---|
| Backend logic (classifier, engine, narrator) | ✅ Built & validated | Lives in Dify, not in this repo — see [Architecture](#architecture) |
| Express proxy (`server/`) | ✅ Working | One route + health check, 194 lines, holds the API key |
| Frontend (`frontend/`) | ✅ Working | 14 components, React 18 + Vite + TS + Recharts |
| Both Dify response shapes handled | ✅ Confirmed against live responses | SPECIFIC and FUZZY, plus decline notices |
| Runs on one local URL | ✅ `http://localhost:8787` | Server serves both API and built UI |
| Deploy config | ✅ Ready, ⬜ not deployed | `render.yaml` blueprint written; needs a GitHub remote first |
| Pushed to GitHub | ⬜ Not yet | No `git remote` configured — 9 commits sit local-only |
| Automated tests | ❌ None | **The biggest known gap.** See [Known gaps](#known-gaps) |
| Auth | ⚠️ Optional shared password | HTTP Basic via `APP_PASSWORD`; off by default locally |

**Next three things worth doing**, in order: push to GitHub → deploy on Render with
`APP_PASSWORD` set → add the regression test suite.

---

## Architecture

ShelfIQ is a **thin client around an already-built, already-tested Dify chatflow**
(`ShelfIQ_V2.yml`). This repo does *not* reimplement the 4-way classifier, the Newsvendor /
markdown math, or the narrator — all of that runs inside Dify.

```
Browser (React)  →  /api/query  →  Express proxy (server/)  →  Dify Chat Messages API
                                         (holds the API key)
```

- **`server/`** — Express, one route: `POST /api/query`. Forwards the question to Dify's
  `/chat-messages` and returns the answer. The **only** place the Dify API key lives; it never
  reaches the browser. Also `GET /api/health`, and it serves `frontend/dist` in production.
- **`frontend/`** — a single-page chat UI. One input box, result cards for MARKDOWN / REORDER /
  DROP / HOLD.

Inside Dify, the pipeline is: a 4-way classifier (`UNSAFE` / `OFF_TOPIC` / `SPECIFIC` / `FUZZY`)
routes to either an exact SKU/name lookup or a hybrid-retrieval path; both call the same
deterministic Python engine; a narrator LLM then writes prose **but never emits or edits a
number**. Every numeric field traces back to the engine's code node.

> ⚠️ **Worth knowing:** because the engine lives in Dify rather than in this repo, that
> "no LLM ever produces a number" guarantee is enforced *there* and can't be verified by reading
> or testing this codebase alone. A Dify-independent rebuild would fix that — see below.

### Why not a Python/FastAPI backend?

[`ORIGINAL_SPEC_FASTAPI_PLAN.md`](ORIGINAL_SPEC_FASTAPI_PLAN.md) — this README's previous content —
specified rebuilding the entire backend standalone in Python/FastAPI so the app wouldn't depend on
Dify at all. **That plan was deliberately not used.** The course deliverable's Phase 1 needed only
"Dify + a frontend," which was already the validated, tested setup; rebuilding would have redone
working logic for no benefit before the deadline. The plan is kept in full for reference, as the
basis for a post-course Dify-independent version.

---

## Running it locally

One-time setup:

```bash
cd server && npm install && cp .env.example .env
```

Then edit `server/.env` and paste in your Dify API key (Dify app → Publish → Access API).

```bash
cd frontend && npm install
```

Every time you want to run it:

```bash
cd frontend && npm run build
```

```bash
cd server && npm start
```

Open **`http://localhost:8787`** and ask something, e.g. *"How is SKU 31526 performing right now?"*
Re-run the build after changing frontend code; the URL never changes.

If `frontend/dist` is missing, the API still runs and page requests return a short note telling you
to build first.

On Windows there's also `Start ShelfIQ.bat`, plus `scripts/install-autostart.ps1` to run it at login.

### Dev workflow (hot reload, two ports)

```bash
cd server && npm run dev
```

```bash
cd frontend && npm run dev
```

Vite serves `:5173` with hot reload and proxies `/api` to the Express server on `:8787`. The
single-port mode above is the "give me one link to open" mode.

---

## Deploying to a public URL

`localhost:8787` only works on this machine, while the server runs. The repo is already set up for
hosting: `render.yaml` plus the root `package.json` let a host build and start everything with
`npm run build` / `npm start`.

Two steps need your own accounts:

**1. Push to GitHub.** There's no remote yet. Create an empty repo (no README, no `.gitignore` —
this repo has both), then:

```bash
git remote add origin https://github.com/<you>/shelfiq.git
```

```bash
git push -u origin master
```

**2. Deploy on Render** (free tier is enough; Railway or Fly work the same way):

1. render.com → **New** → **Blueprint** → pick the repo.
2. Render reads `render.yaml` and asks for two secrets:
   - `DIFY_API_KEY` — the key from `server/.env`
   - `APP_PASSWORD` — pick any password (see below)
3. Deploy. You get a permanent URL like `https://shelfiq.onrender.com`.

After that, `git push` redeploys and the URL stays the same.

### Set `APP_PASSWORD` — this matters

Every question costs Dify credits on **your** key. A public URL with no password is an open tap:
anyone who finds the link can spend your quota. `APP_PASSWORD` puts the whole app behind one shared
password (HTTP Basic — the browser asks once and remembers).

It's one shared password, not user accounts, and it's only safe over HTTPS, which Render gives you
automatically. Leave it unset locally and nothing changes.

**Never commit the Dify key.** `server/.env` is git-ignored and `render.yaml` holds no secret
values — the host prompts for them and stores them itself.

### Free-tier note

Render's free tier sleeps a service after ~15 minutes of no traffic, so the first request after a
quiet spell takes ~30–60s to wake. The URL still works; it's just slow on that first hit.

---

## The UI

Built against [`ShelfIQ_UI_Design_Guide_v2.md`](ShelfIQ_UI_Design_Guide_v2.md): the brand colour and
type system, and the **Safety Stock Gauge** — a zoned gauge showing each item's weeks of cover
against its reorder (0.5×) and markdown (1.5×) lines.

It's a **single scrolling page that shows one answer at a time**: sticky header, a hero with the
composer and a "Try a scenario" card, an `#analysis` section, an always-present methodology
accordion, and a pinned follow-up composer once there's an answer.

Asking anything — typing in either composer, clicking a scenario row, tapping a follow-up chip —
**replaces** `#analysis` entirely. There is no conversation thread and no history to scroll back
to; that's a deliberate choice, not an omission. Every question goes through one `askQuestion()` in
`App.tsx`, which is also why a scenario row submits immediately instead of just filling the input.

Capital at risk is shown in euros. The catalog carries no currency of its own — `list_price` and
`unit_cost` are bare numbers — so the euro is a *stated fact* about what those columns mean,
recorded in `frontend/src/format.ts`.

Components live in `frontend/src/components/`. `frontend/src/normalize.ts` parses both Dify response
shapes and derives the gauge lines, so the gauge and the weeks-of-cover chart can't disagree.

---

## Response shapes

Both are confirmed against live responses from the deployed Dify app. Dify answers with a JSON
**string** rather than plain chat text, in a couple of different shapes depending on the internal
path; `normalize.ts` converts whichever arrives into one format the UI renders.

- **SPECIFIC** — `{ status, products: [...], explanation }`. Each product carries `action`,
  `urgency`, `recommendation`, `pressure_ratio`, `critical_ratio`, `optimal_order_quantity`,
  `markdown_schedule`, and the raw metrics. There is **no** per-item `reasoning` or
  `expected_impact` on this path — the shared `explanation` is its only narrative, and the UI does
  not fake the missing fields.
- **FUZZY** — `{ understood, actions: [...], summary, no_action_reason }`, each action nesting its
  metrics under `key_metrics` and carrying `reasoning` and `expected_impact`.
- **UNSAFE / OFF_TOPIC** — the FUZZY shape with `actions: []` and `is_notice: true`; rendered as the
  cream decline notice instead of cards.

---

## Known gaps

Honest list of what isn't done:

- **No automated tests at all.** No test runner is even installed. The original spec defined a
  10-case regression suite — including the critical one that scans the narrator's `explanation` for
  any number not present in `products` (the "narrator silently changed a number" bug). None of it
  is ported. This is the single biggest risk in the project.
- **Not on GitHub.** 9 commits, no remote. Nothing is backed up off this machine.
- **Not deployed.** The config is ready; the deploy hasn't happened.
- **The engine isn't in this repo**, so its guarantees aren't independently verifiable here.
- **Exact-match only on SKU lookup** — no typo tolerance, no fuzzy string matching. A deliberate
  v1 scope cut.
- **FUZZY intent rests on one LLM call.** A misclassified `decision_type` surfaces the wrong subset
  of items. Known, accepted for v1.
- **DROP thresholds are catalog-specific** (`velocity < 80`, `margin < 55`, the 25th percentiles of
  this catalog — measured: 83.6 and 55.0). Swapping datasets means recomputing them.
- **Uncommitted work in progress:** `frontend/src/components/HeroComposer.tsx` has an unstaged
  headline change (adding the design guide's italic accent, *"How much, at what price?"*).

---

## Repo contents

| Path | What it is |
|---|---|
| `server/` | Express proxy — one route, keeps the Dify key server-side |
| `frontend/` | React + Vite + TypeScript single-page UI |
| `ShelfIQ_V2.yml` | Dify chatflow export — **the real backend logic**. Import into Dify to recreate the app |
| `fast_fashion_product_profiles.csv` | The 200-SKU catalog Dify reads from |
| `ShelfIQ_UI_Design_Guide_v2.md` | The UI spec this frontend implements |
| `ShelfIQ_UI_Design_Guide.md.txt` | v1: an earlier, broader concept (Track A/B). Superseded; kept for the visual system |
| `ORIGINAL_SPEC_FASTAPI_PLAN.md` | The unused Python/FastAPI rebuild plan, kept for reference |
| `render.yaml` | Render blueprint for one-click deploy |
| `brand/` | Logo source and SVGs |
| `scripts/`, `Start ShelfIQ.bat` | Windows convenience launchers |

---

## The science

- **Arrow, Harris & Marschak (1951)**, *Optimal Inventory Policy* — Newsvendor order quantities.
  The optimal quantity isn't the forecast; Q\* sits at the critical ratio Cu / (Cu + Co).
- **Gallego & van Ryzin (1994)**, *Optimal Dynamic Pricing of Inventories with Stochastic Demand
  over Finite Horizons* — monotone markdown ladders; prices only ever fall.
- **Articles 101/102 TFEU** and **EU Omnibus Directive 2019/2161** — the classifier declines
  competitor price coordination, fake reference prices, and below-cost undercutting outright,
  without sending them to an LLM for a substantive answer.
