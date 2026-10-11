# ShelfIQ

**A B2B retail merchandising copilot.** Ask it about a SKU, a markdown, or a reorder, and it answers
from your catalog using management science — Newsvendor critical ratios and Gallego & van Ryzin
markdown ladders — not from a language model's guess.

![status](https://img.shields.io/badge/status-working%20prototype-0259DD)
![backend](https://img.shields.io/badge/backend-Dify%20chatflow-0259DD)
![app%20code](https://img.shields.io/badge/app%20code-not%20in%20this%20repo-B3341C)
![tests](https://img.shields.io/badge/automated%20tests-none-B3341C)
![license](https://img.shields.io/badge/license-none-888)

---

## What this repo currently holds

> **Read this first.** This repository is at present a **specification-and-backend-export repo**,
> not a runnable application. It contains four files:

| File | What it is |
|---|---|
| `ShelfIQ_Dify_11102026.yml` | **The real backend.** The Dify chatflow export — classifier, retrieval, engine, narrator. Import into Dify to recreate the working app. |
| `ShelfIQ_UI_Design_Guide_v2.md` | The UI specification: brand colour/type system, components, the Safety Stock Gauge, and the API-to-UI field mapping. |
| `product_profiles.csv` | A 150-SKU homeware catalog. ⚠️ **The chatflow does not read this file** — see [Data sources](#data-sources). |
| `README.md` | This file. |

**The web app's source code is not here.** The React frontend and Express proxy described below
exist in a local working copy only (9 commits on a `master` branch with no configured remote) and
have never been pushed. Anyone cloning this repo gets the spec and the backend export — enough to
rebuild the app in Dify, but not enough to run the web UI.

---

## Status at a glance

| Area | State | Notes |
|---|---|---|
| Backend logic (classifier, engine, narrator) | ✅ Built & validated | Runs in Dify; exported here as `ShelfIQ_Dify_11102026.yml` |
| Backend export published | ✅ In this repo | 51.8 KB, importable into Dify |
| UI specification published | ✅ In this repo | 62.0 KB |
| Frontend source (`frontend/`) | ⬜ **Local only** | React 18 + Vite + TS + Recharts, 14 components — not pushed |
| Express proxy (`server/`) | ⬜ **Local only** | 194 lines, one route + health check, holds the API key — not pushed |
| Deploy config (`render.yaml`) | ⬜ **Local only** | Blueprint written, not pushed, never deployed |
| Catalog shipped here matches the one in use | ❌ **No** | Repo has a homeware catalog; the flow fetches a fast-fashion one |
| Automated tests | ❌ None | No test runner installed anywhere; zero test files |
| Public deployment | ⬜ Not deployed | No live URL |
| Auth | ⚠️ Optional shared password | `APP_PASSWORD` Basic auth in the unpublished server |

**Next three things worth doing**, in order:

1. **Push the app code** (`server/`, `frontend/`, `render.yaml`, root `package.json`) so this repo is
   runnable. Note the local branch is `master` while GitHub's default is `main`.
2. **Resolve the catalog mismatch** — either remove `product_profiles.csv` or repoint the flow at it
   and recalibrate the thresholds.
3. **Add the regression test suite**, above all the narrator-number check.

---

## Architecture

ShelfIQ is a **thin client around a Dify chatflow**. The intelligence lives in Dify; the web app
(unpublished) is just a UI and a key-holding proxy.

```
Browser (React)  →  /api/query  →  Express proxy  →  Dify Chat Messages API
                                   (holds the key)          │
                                                            ▼
                                              HTTP-fetches the catalog CSV
                                                   from GitHub at runtime
```

Inside the chatflow: a 4-way classifier (`UNSAFE` / `OFF_TOPIC` / `SPECIFIC` / `FUZZY`) routes to
either an exact SKU/name lookup or a hybrid-retrieval path; both reach a deterministic Python code
node implementing the science; a narrator LLM then writes prose **but never emits or edits a
number**. Every numeric field traces back to that code node.

> ⚠️ Because the engine lives in Dify rather than in source here, that "no LLM ever produces a
> number" guarantee is enforced *there* and cannot be verified by reading or testing this
> repository. A Dify-independent rebuild in Python/FastAPI was specified and deliberately deferred:
> the course deliverable's Phase 1 needed only Dify plus a frontend, which was already the
> validated setup, so rebuilding would have redone working logic before the deadline.

### Data sources

This is the most confusing thing about the repo as it stands, so it is worth stating plainly.

The chatflow does **not** read the CSV in this repository. It fetches its catalog over HTTP at
runtime from a hardcoded URL in a *different* repo:

```
https://raw.githubusercontent.com/hytangBill/AI-BI-Nyenrode-MSc34/refs/heads/main/fast_fashion_product_profiles.csv
```

That URL is live (HTTP 200, 24,292 bytes) and serves a **200-SKU fast-fashion catalog** —
categories `tops, bottoms, accessories, outerwear, dresses, knitwear, footwear, seasonal`.

The `product_profiles.csv` committed here is something else entirely: a **150-SKU homeware
catalog** — categories `decor, garden, general, kitchen, seasonal, storage`, with products like
*REGENCY CAKESTAND 3 TIER*. Nothing reads it.

Consequences, measured:

- The engine's DROP thresholds (`weekly_velocity < 80`, `margin_pct < 55`) are calibrated to the
  fast-fashion catalog's 25th percentiles — **measured 83.6 and 55.0**. ✅ Correct for the data
  actually in use.
- Those same thresholds are **badly miscalibrated for the homeware CSV in this repo**, whose 25th
  percentiles are **38.1 and 35.8**. If the flow were ever repointed at it without recalibration,
  DROP would over-fire on roughly the bottom half of the catalog instead of the bottom quarter.
- The canonical example SKU in the spec and in older docs, `31526` (*ZIP THROUGH HOODIE BLACK*),
  exists in the fast-fashion catalog but **not** in `product_profiles.csv`.

Both catalogs share the same 16-column schema and the same four `trend` values, so the shape is
compatible — only the contents and calibration differ.

---

## Running it

**From this repo alone, you cannot run the web app** — the source isn't here. What you *can* do:

1. Import `ShelfIQ_Dify_11102026.yml` into Dify.
2. Publish it and take the API key from **Publish → Access API**.
3. Ask questions through Dify's own chat interface.

With the unpublished app code, the flow is: `npm install` in `server/` and `frontend/`, put the
Dify key in `server/.env`, `npm run build` in `frontend/`, then `npm start` in `server/` — which
serves both the API and the built UI on a single port, `http://localhost:8787`. A `render.yaml`
blueprint deploys the same thing to Render, where `APP_PASSWORD` should be set to keep a public URL
from spending Dify credits on your key.

---

## The UI specification

`ShelfIQ_UI_Design_Guide_v2.md` is the authoritative visual spec. Its headline element is the
**Safety Stock Gauge** — a zoned gauge showing each item's weeks of cover against its reorder
(0.5×) and markdown (1.5×) lines.

The specified app is a **single scrolling page that shows one answer at a time**: sticky header, a
hero with the composer and a "Try a scenario" card, an `#analysis` section, an always-present
methodology accordion, and a pinned follow-up composer once an answer exists. Asking anything
**replaces** `#analysis` entirely — there is no conversation thread and no scrollback. That is a
deliberate choice, not an omission.

Capital at risk is shown in euros. The catalog carries no currency of its own — `list_price` and
`unit_cost` are bare numbers — so the euro is a *stated assumption* about what those columns mean.

---

## Response shapes

The chatflow answers with a JSON **string** rather than plain chat text, in one of three shapes
depending on the path taken. Any client has to normalise them.

- **SPECIFIC** — `{ status, products: [...], explanation }`. Each product carries `action`,
  `urgency`, `recommendation`, `pressure_ratio`, `critical_ratio`, `optimal_order_quantity`,
  `markdown_schedule`, and the raw metrics. There is **no** per-item `reasoning` or
  `expected_impact` on this path; the shared `explanation` is its only narrative.
- **FUZZY** — `{ understood, actions: [...], summary, no_action_reason }`, each action nesting its
  metrics under `key_metrics` and carrying `reasoning` and `expected_impact`.
- **UNSAFE / OFF_TOPIC** — the FUZZY shape with `actions: []` and `is_notice: true`, intended to
  render as a decline notice rather than cards.

---

## Known gaps

- **The app source is not in this repo.** The single biggest gap: the repo is not runnable as a web
  app, and the only copy of the frontend and server is one unpushed local checkout.
- **No automated tests anywhere.** No test runner is installed; a filesystem search finds zero test
  files. The specified 10-case regression suite — including the critical check that scans the
  narrator's `explanation` for any number absent from `products` — is unimplemented. For a system
  whose core promise is "no LLM ever invents a number," this is the most serious omission.
- **An unused, mismatched catalog is committed here**, inviting the assumption that it's the data in
  use. See [Data sources](#data-sources).
- **The engine is duplicated inside the chatflow.** Two separate code nodes carry the same science,
  one per path. The spec called for a single shared implementation; the export still has both, so a
  fix must be applied twice or they will drift.
- **The chatflow falls back to hardcoded demo rows** when retrieval returns nothing, rather than
  returning a not-found error. The spec explicitly called this a prototype-only convenience that
  should not survive into production; it is still present.
- **Fast-fashion assumptions are baked into the prompts** (example categories such as `knitwear`),
  which would need rewriting for a different catalog.
- **Exact-match only on SKU lookup** — no typo tolerance, no fuzzy string matching. A deliberate v1
  scope cut.
- **FUZZY intent rests on a single LLM call.** A misclassified `decision_type` silently surfaces the
  wrong subset of items. Known and accepted for v1.
- **No license file**, so nobody can legally reuse this code.

---

## The science

- **Arrow, Harris & Marschak (1951)**, *Optimal Inventory Policy* — Newsvendor order quantities.
  The optimal quantity is not the forecast; Q\* sits at the critical ratio Cu / (Cu + Co).
- **Gallego & van Ryzin (1994)**, *Optimal Dynamic Pricing of Inventories with Stochastic Demand
  over Finite Horizons* — monotone markdown ladders; prices only ever fall.
- **Articles 101/102 TFEU** and **EU Omnibus Directive 2019/2161** — the classifier declines
  competitor price coordination, fake reference prices, and below-cost undercutting outright,
  without sending them to an LLM for a substantive answer.
