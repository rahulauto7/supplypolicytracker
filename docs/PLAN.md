# TariffShift — Dynamic Tariff Calculation & Supply Chain Re-Routing Engine

Hack Apertus Online Hackathon (1–16 Oct 2026) — **Track 2B: Own Project**

> **Hard deadline: 16 Oct 2026, 12:00 CEST. No extensions.**
> Required submission: **technical report + code repository + dataset (if relevant)**.
> Must be built with **Apertus 1.5 (8B or 70B)**.

---

## 1. Problem

Tariff and trade policy now changes week to week (US Section 232/301 and IEEPA tariffs, EU
countermeasures and CBAM, export controls, FTA renegotiations). A procurement team usually
learns about a change late, works out the landed-cost impact in a spreadsheet, and then
spends weeks finding another supplier. During that time the margin is already lost.

The pitch numbers in the idea brief ("margins down up to 15%", "32% of executives defer
expansion") **need a cited source in the report**. Otherwise, replace them with numbers you can source.

## 2. Solution in one sentence

A platform that **reads trade-policy changes in any language with Apertus**, maps them to
the user's products (HS codes) and lanes, **recalculates landed cost with a deterministic engine**,
and **ranks other suppliers, origins and routes** that protect margin, with every
claim linked to its source document.

### Core design principle (say this to the jury)
**The LLM never does the math.** Apertus reads, classifies, extracts, explains and orchestrates.
A deterministic, unit-tested engine does tariff stacking, landed cost and optimisation. This is
how you avoid hallucinated duty rates, and it makes the system auditable.

## 3. Why Apertus (judges will ask)

| Need | Why Apertus fits |
|---|---|
| Policy texts in DE / FR / IT / EN (Swiss FOCBS / BAZG, EU OJ, US Federal Register) | Strongly multilingual (1000+ languages in pre-training, Swiss languages emphasised) |
| Confidential BOMs, supplier prices, margins | Open weights let you self-host, so data never leaves the company. Data sovereignty is the Swiss angle |
| Auditability for compliance | Fully open model + open training data → explainable for regulated industry |
| Cost | 8B for high-volume extraction/classification, 70B only for reasoning/reports |

**Swiss angle for the story:** Switzerland abolished industrial tariffs on 1 Jan 2024. Swiss
exporters (watches, machinery, medtech) were then hit by sharply higher US tariffs in 2025, and the
rate has changed since. *Check the current US rate on Swiss goods before you put it in the deck.*
Switzerland/EFTA also has a large FTA network (EU, China, Japan, etc.). Using **FTA preference + rules of
origin** to cut duty is a re-routing lever most tools ignore.

## 4. Features (MVP → stretch)

### MVP (must have by 15 Oct)
1. **Policy Radar.** Ingest policy notices (Federal Register API, EU/Swiss notices, news RSS).
   Apertus extracts a structured `PolicyEvent` JSON: affected countries, HS codes/chapters,
   measure type (ad valorem, specific, quota, ban), rate change, effective date, source URL and the source quote.
2. **HS Code Classifier.** Takes a free-text product description and returns the top-3 HS-6 candidates with reasoning.
   Apertus is grounded by retrieval over the HS nomenclature, and the user confirms the result.
3. **Landed Cost Engine** (deterministic Python, see §6).
4. **Re-Routing / Re-Sourcing Recommender.** For a SKU, ranks candidate (supplier country ×
   route × mode × FTA use) by landed cost, lead time and risk. Shows margin delta vs. the current set-up.
5. **What-if simulator.** A slider for "tariff on CN→US chapter 85 = X%" that re-ranks options live.
6. **Explain & cite.** Apertus writes a short briefing in the user's language (EN/DE/FR/IT)
   with citations to the source notices.

### Stretch
- Alerts (email/Slack) when a new `PolicyEvent` hits a tracked SKU.
- Allocation optimiser (split volume across suppliers with capacity constraints; PuLP / OR-Tools).
- Rules-of-origin checker (does the product qualify for FTA preference given its BOM origins?).
- Map view of lanes (Leaflet / deck.gl) with risk heat.
- Voice briefing (ElevenLabs, a Track 2B prize sponsor, which makes a nice touch for the demo).

## 5. Architecture

```
            ┌────────────────────────────────────────────┐
 Sources →  │ Ingestion (cron)                           │
 FedReg API │  fetch → clean → chunk → store raw text    │
 EUR-Lex    └───────────────┬────────────────────────────┘
 BAZG/news                  ▼
            ┌────────────────────────────────────────────┐
            │ Apertus 8B: extraction → PolicyEvent JSON   │──► Postgres/SQLite
            │ (JSON-schema constrained, quote-grounded)   │    + vector index
            └───────────────┬────────────────────────────┘
                            ▼
 User SKUs ─► HS classifier (Apertus + RAG over HS nomenclature)
                            ▼
            ┌────────────────────────────────────────────┐
            │ Deterministic engine (pure Python, tested) │
            │  tariff stack → landed cost → ranking/opt  │
            └───────────────┬────────────────────────────┘
                            ▼
            Apertus 70B agent (tool calling): "Why did my margin drop,
            what should I do?" → calls engine tools → cited briefing
                            ▼
            FastAPI  ◄──►  Web UI (React/Vite or Streamlit)
```

**Stack recommendation (fast for a team of 1–5, 8 days):**
- Backend: Python 3.11, FastAPI, Pydantic, SQLite (Postgres optional), `pytest`.
- LLM: Apertus 1.5 through an OpenAI-compatible endpoint (see §8), so the client code is the same whatever the host.
- RAG: `sentence-transformers` multilingual embeddings + FAISS/Chroma.
- Routing: `searoute` (Python sea-route distances) + simple air/rail/road tables.
- Optimisation: PuLP (stretch).
- Frontend: Streamlit if you want speed. React + Vite + Recharts + Leaflet if you want polish.

## 6. Landed cost model (the core math)

For SKU *s*, supplier origin *o*, destination *d*, route/mode *r*:

```
customs_value   = FOB                               (US)
                = FOB + freight + insurance (CIF)   (EU, CH and most others)

duty            = Σ applicable measures on (HS, o, d, date):
                  MFN or FTA-preferential base rate
                + additional duties (e.g. US Sec. 301 / 232 / IEEPA, EU safeguards / AD-CVD)
                  ad valorem: rate × customs_value
                  specific:   rate × qty (kg, units)  ← Swiss agri tariffs are by gross weight
fees            = e.g. US MPF (ad valorem, min/max cap) + HMF (sea only)
import_VAT      = vat_rate × (customs_value + duty [+ fees, per country rules])
                  (often recoverable, so show it but exclude it from margin by default, as a toggle)
freight         = f(distance, mode, weight/volume, current index such as FBX lane rate)
transit_cost    = unit_cost × annual_holding_rate × transit_days / 365
fx_adjust       = price in supplier currency × FX (+ optional hedge buffer)

landed_cost     = unit_price + freight + insurance + duty + fees + brokerage
                  + transit_cost (+ import_VAT if non-recoverable)
margin          = sell_price − landed_cost
```

**Ranking score** (user-weighted):
`score = w1·landed_cost_norm + w2·lead_time_norm + w3·policy_risk + w4·supplier_risk`
where `policy_risk` comes from the pending/announced `PolicyEvent`s on that lane (Apertus-extracted)
and country volatility.

**Tariff stacking rules matter.** US additional duties stack on top of MFN, and some are mutually
exclusive or exempt (e.g. USMCA-qualifying goods). Encode them as explicit rules with tests, not prompts.

## 7. Data sources (free/public, verify access early, on day 1)

| Data | Source |
|---|---|
| US tariff schedule (HTS) | USITC HTS — hts.usitc.gov (has a REST/export API) |
| US policy notices | Federal Register API — federalregister.gov/developers (free, no key) |
| US classification rulings (labelled data for HS eval) | CBP CROSS rulings — rulings.cbp.gov |
| EU tariff | EU TARIC / Access2Markets; EU notices via EUR-Lex |
| Swiss tariff | Tares (xtares.admin.ch), BAZG/FOCBS notices (DE/FR/IT) |
| Global MFN/preferential rates | WTO Tariff & Trade Data, WITS (World Bank), Market Access Map (ITC) |
| Trade flows (alternative supplier discovery) | UN Comtrade API (free tier with key) |
| Freight | Freightos Baltic Index (public lane indices), Drewry WCI (weekly headline) |
| Sea distance | `searoute` Python package |
| Country risk | World Bank LPI / WGI |

**Hackathon shortcut:** do not try to cover the whole world. Pick **3 destinations (US, EU, CH)**,
**~5 origins (CN, VN, MX, IN, DE/CH)** and **2–3 HS chapters** (e.g. 85 electronics, 91 watches,
90 medtech, 87 e-bike parts). Hand-curate a clean `tariff_rules.yaml` for those and show that the
pipeline extends to more.

## 8. Accessing Apertus 1.5

Options (confirm on day 1 which one actually gives you API keys):
1. **Public AI**: platform.publicai.co (API) / chat.publicai.co (chat). Check for an OpenAI-compatible endpoint.
2. **Infomaniak**: offers Apertus 1.5 70B via API (Swiss hosted).
3. **Self-host**: `swiss-ai/Apertus-v1.5-8B` / `swiss-ai/Apertus-v1.5-70B` on Hugging Face, served
   with **vLLM** (OpenAI-compatible server). 8B runs on one 24 GB GPU; 70B needs multi-GPU or a quantised build.
4. **Local dev**: 8B with mlx-lm (Apple Silicon) or vLLM.
5. Ask the hackathon mentors (bookable sessions) if participant credits/endpoints exist.

Keep everything behind one config:
```env
LLM_BASE_URL=...        # OpenAI-compatible
LLM_API_KEY=...
LLM_MODEL_FAST=<apertus 8B model id>
LLM_MODEL_SMART=<apertus 70B model id>
```

### Prompting patterns that make an 8B model reliable
- **JSON-schema constrained output** (vLLM guided decoding / `response_format`) for `PolicyEvent`.
- **Quote grounding:** require a verbatim `evidence_quote` for every field. Reject the output if the quote is
  not a substring of the source.
- **Retrieve, then classify** for HS codes: give the model the top-20 candidate headings from vector search.
- **Self-consistency:** run 3 samples and take the majority HS code. Low agreement means "needs human review".

## 9. Evaluation (this is what makes the technical report strong)

| What | How | Metric |
|---|---|---|
| Policy extraction | Hand-label ~50–100 notices (EN + DE/FR/IT) | field-level P/R/F1, date & rate exact match |
| HS classification | ~200 CBP CROSS rulings (description → HS6) | top-1 / top-3 accuracy at HS-4 and HS-6 |
| Multilingual robustness | Same notices translated / original Swiss DE-FR-IT | F1 drop vs. EN |
| 8B vs 70B | Same benchmarks | accuracy vs. latency vs. cost |
| Hallucination guard | % outputs rejected by quote-check, error rate after guard | |
| Engine correctness | Unit tests against hand-computed landed costs | 100% pass |
| Business impact | Demo scenario: margin before/after re-routing | CHF/USD per unit & per year |

**Publish the labelled datasets on Hugging Face.** "Dataset where relevant" is a listed deliverable,
and an open multilingual trade-policy extraction set is a real contribution.

## 10. Demo scenario (3–4 min)

> *"Alpine E-Bikes AG" (fictional), a Swiss company assembling e-bikes and selling into the US and EU, sources motors
> & battery packs from China.*

1. Dashboard: 12 SKUs, current margins.
2. A new US notice lands (real Federal Register document, or a German-language Swiss notice).
   Apertus extracts it into a structured event with the quote highlighted, within seconds.
3. Alert: "3 SKUs affected, margin on Motor-250W drops from 22% → 9%."
4. Recommender: switch the motor to a Vietnam supplier via Hai Phong → LA, or use an EU supplier and claim EU–CH
   FTA preference for the EU market. Show the table: landed cost, lead time, risk, margin.
5. What-if slider: "If the rumoured +10% passes…" and the ranking reshuffles live.
6. Ask the agent in French: *"Pourquoi ma marge a baissé et que dois-je faire ?"* It answers with
   citations and calls the engine tools visibly.
7. Close: open model, self-hostable, data stays in Switzerland.

## 11. Repo structure

```
supplypolicytracker/
├── backend/
│   ├── app/main.py              # FastAPI
│   ├── app/llm.py               # OpenAI-compatible Apertus client
│   ├── app/extract.py           # PolicyEvent extraction + quote grounding
│   ├── app/hs_classifier.py     # RAG + self-consistency
│   ├── app/engine/tariffs.py    # rule stacking
│   ├── app/engine/landed.py     # landed cost
│   ├── app/engine/routing.py    # searoute + mode tables
│   ├── app/engine/rank.py       # scoring / PuLP
│   ├── app/agent.py             # tool-calling agent
│   └── tests/
├── data/
│   ├── tariff_rules.yaml
│   ├── suppliers.csv            # demo suppliers
│   ├── skus.csv
│   └── eval/                    # labelled notices, HS test set
├── frontend/
├── eval/run_eval.py
├── docs/PLAN.md                 # this file
└── docs/TECHNICAL_REPORT.md
```

## 12. 8-day timeline (today = 8 Oct)

| Day | Date | Goal |
|---|---|---|
| 1 | Wed 8 Oct | Confirm Apertus API access; scope (destinations/origins/chapters); repo skeleton; book mentor |
| 2 | Thu 9 Oct | Landed cost engine + tariff rules YAML + unit tests; demo data (SKUs, suppliers) |
| 3 | Fri 10 Oct | Policy ingestion (Federal Register API + 1 Swiss/EU source) + Apertus extraction w/ JSON schema + quote check |
| 4 | Sat 11 Oct | HS classifier (RAG) + recommender/ranking + what-if API |
| 5 | Sun 12 Oct | Frontend: dashboard, alert, recommendation table, slider |
| 6 | Mon 13 Oct | Agent with tool calling + multilingual briefings; label eval data |
| 7 | Tue 14 Oct | Run evals (8B vs 70B, languages); fix failures; publish dataset to HF |
| 8 | Wed 15 Oct | Technical report, README, demo video (optional but helps), polish |
| — | Thu 16 Oct **before 12:00 CEST** | Submit. Aim to submit on the 15th evening |

**Team split (if 3–4 people):** (1) engine + data, (2) LLM extraction/HS/agent + evals,
(3) frontend + demo, (4) report + dataset + pitch.

## 13. Technical report outline

1. Problem & motivation (sourced statistics)
2. System overview & architecture diagram
3. Role of Apertus (why an open, multilingual, sovereign model)
4. Methods: extraction, grounding, HS RAG, agent tools, deterministic engine
5. Data & datasets (sources, licences, labelling protocol)
6. Evaluation results (tables from §9), 8B vs 70B
7. Demo case study with numbers
8. Limitations & risks (not customs/legal advice; data freshness; rules of origin complexity;
   classification needs human confirmation)
9. Future work & business model (SaaS per SKU tracked; on-prem for regulated firms)

## 14. Judging angles to hit

The jury mixes industry (retail/e-commerce, startups), academia (FHNW, HSG) and the Swiss AI ecosystem:
- **Real business impact.** Show CHF numbers, not just a dashboard.
- **Rigor.** Evals, open dataset, honest limitations.
- **Meaningful Apertus use.** Multilingual Swiss sources, sovereignty, and 8B vs 70B trade-off analysis.
- **Responsible AI.** Grounding, human-in-the-loop for classification, no hallucinated rates.
