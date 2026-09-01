# StockPulse — AI Inventory & Dynamic Pricing Engine

A reactive commerce advisor. When inventory crosses a reorder threshold or demand velocity
spikes, StockPulse detects the signal automatically, asks an LLM for a price adjustment **and**
a reorder quantity, and queues both for merchandising approval. Nothing goes live without a human
pressing accept.

```
order / stock change  →  event  →  async advisor (AI, rule-based fallback)
                                      ↓
                          PENDING pricing + reorder suggestions
                                      ↓
                        merchandising accepts  →  price published / stock received
```

## Stack

| Layer | Choice |
| --- | --- |
| Backend | Java 21, Spring Boot 3.5, Maven, Spring Data JPA, Bean Validation, H2 (in-memory) |
| LLM | Groq + `openai/gpt-oss-20b` (replacement for retired `llama-3.1-8b-instant`) |
| Frontend | React 18 + Vite |

## Architecture

Four layers, not three. Plain controller/service/repository tends to collapse pricing rules,
reorder maths, prompt building, event handling, and persistence into one service class, so the
commerce logic lives in its own layer behind an interface.

```
api/         controllers + DTOs        HTTP only: deserialise, delegate, serialise
service/     orchestration             transactions, load/mutate/announce - no commerce rules
engine/      commerce contracts        PricingAdvisor, ReorderAdvisor + implementations
domain/      entities                  invariants and state machines live with the data
repository/  persistence               Spring Data JPA + Specifications
```

Two properties fall out of that split:

- **One contract, two callers.** The on-demand HTTP endpoints and the async agentic loop both go
  through `SuggestionService.generate*`, which resolves an advisor from `CommerceAdvisorRegistry`.
  Neither caller knows or cares which implementation ran.
- **Runtime switchable.** The registry indexes every `PricingAdvisor` bean by `name()` and re-reads
  the active name from config on every resolve, so `PATCH /admin/commerce-strategy` changes
  behaviour on the next recommendation with no restart and no rewiring.

Advisors receive a `CommerceContext` and never touch a repository, which is why they can be
unit-tested with a plain constructor call and no Spring context. That context object is also the
sprint 2 seam: competitor prices and margin floors get added there, and existing advisors keep
compiling.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/products` | Create a product |
| `GET` | `/products?status=&category=` | Filterable catalog |
| `GET` | `/products/{id}` | Single product |
| `PATCH` | `/products/{id}/stock` | Absolute stock correction; returns immediately |
| `POST` | `/products/{id}/orders` | Simulate a sale; returns immediately |
| `POST` | `/products/{id}/suggest-pricing` | On-demand pricing suggestion (`MANUAL`) |
| `POST` | `/products/{id}/suggest-pricing/stream` | Same, as an SSE stream of the model's reasoning |
| `POST` | `/products/{id}/suggest-reorder` | On-demand reorder suggestion (`MANUAL`) |
| `GET` | `/pricing-suggestions?status=` | Suggestion queue for the console |
| `GET` | `/reorder-suggestions?status=` | Suggestion queue for the console |
| `PATCH` | `/pricing-suggestions/{id}` | Accept/reject; accept publishes the price |
| `PATCH` | `/reorder-suggestions/{id}` | Accept/reject; accept receives stock |
| `GET`/`PATCH` | `/admin/commerce-strategy` | Inspect and switch the active advisor at runtime |

Errors are RFC 7807 `ProblemDetail`: `400` for validation, `404` for unknown ids, `409` for domain
conflicts such as overselling or re-deciding a settled suggestion.

## Prerequisites

- **JDK 21+** — a full JDK, not just a JRE (`javac -version` must work)
- **Maven 3.9+** (or use the bundled `mvnw` / `mvnw.cmd`)
- **Node 20+**
- A **Groq API key** from [console.groq.com](https://console.groq.com) — free tier is enough

## Quick start (both servers, with API key)

Two terminals. Backend first; frontend second.

**Terminal 1 — backend**

```powershell
# PowerShell
$env:LLM_API_KEY = "gsk_your_groq_key_here"
cd backend
.\mvnw.cmd spring-boot:run
```

```bash
# bash / zsh
export LLM_API_KEY=gsk_your_groq_key_here
cd backend && ./mvnw spring-boot:run
```

Wait until you see `Started StockPulseApplication`. Backend: `http://localhost:8080`.

**Terminal 2 — frontend**

```powershell
cd frontend
npm install
npm run dev
```

Console: `http://localhost:5173`.

### Demo in the console (under 5 minutes)

1. Open the console — `PRD-003` already has an initial price suggestion waiting.
2. On the catalog, click **Sell 1** on `Organic Cotton T-Shirt` (`PRD-003`). Stock is already below
   threshold; the agentic loop queues inventory-low pricing + reorder suggestions (poll ~4s).
3. In **Awaiting approval**, read the reasoning, then **Accept & publish price**.
4. Optional: click **Ask live** on any row to watch the model stream its reasoning over SSE.
5. Optional spike path: **Sell 5** / enough units on `Hoodie — Heather Grey` (`PRD-008`) until
   velocity clears 3× peer average.

Without `LLM_API_KEY` the app still starts: every AI call fails fast and falls back to rules.

## Run the backend (detail)

```powershell
$env:LLM_API_KEY = "gsk_your_groq_key_here"
cd backend
.\mvnw.cmd spring-boot:run
```

Backend comes up on `http://localhost:8080`, seeded with the 8 demo products by
`bootstrap/DataSeeder`.

- H2 console: `http://localhost:8080/h2-console` — JDBC URL `jdbc:h2:mem:stockpulse`, user `sa`, no password
- Tests: `.\mvnw.cmd test` — 109 tests, no key or network required

With the backend up, `.\demo.ps1` from the repo root walks the whole system: both agentic triggers,
idempotency under repeated sales, the human approval checkpoint, runtime strategy switching, and
the input guardrails.

## Run the frontend (detail)

```powershell
cd frontend
npm install
npm run dev
```

Console opens on `http://localhost:5173`, already allow-listed in the backend CORS config. Point it
elsewhere with `VITE_API_BASE=http://host:port` if the backend is not on `localhost:8080`.

## The merchandising console

Two panels, in the order a merchandiser needs them.

**Awaiting approval** is the whole point of the page: every recommendation the system queued on its
own, grouped by product rather than split into two tables, because when one order both drained stock
and spiked demand the price and reorder questions for that product are one decision. Each card
carries the trigger badge, the model's confidence, its plain-English reasoning, and accept/reject.
Auto-triggered badges are filled and saturated while requested ones stay outlined — the single most
important thing the screen communicates is that nobody asked for this. Cards also show whether the
**AI** or the **rule** engine produced them, so a fallback after an LLM timeout is visible rather
than passing itself off as a model answer.

**Catalog** is where you drive the demo. `Sell 1` / `Sell 5` simulate orders, which drain stock and
raise demand velocity — exactly what the backend listens for. `Restock` sets stock back to three
times the threshold. **Ask live** opens an SSE stream
(`POST /products/{id}/suggest-pricing/stream`) so you can watch the model's reasoning assemble
token by token before the suggestion is queued. Alongside the required stock, price, velocity, and
status columns it shows gross margin, days of cover, a stock heat bar marked with the reorder
threshold, category filters, and a sparkline of approved price moves reconstructed from accepted
suggestions.

The console polls every four seconds for auto-triggered suggestions. It can simulate demand,
request an opinion (blocking or streamed), and decide a suggestion — and nothing else. There is no
control to edit a price directly, because there is no endpoint to do so: accepting a suggestion is
the only path to a live price change.

### Live reasoning stream (SSE bonus)

```
POST /products/{id}/suggest-pricing/stream
  event: status      { phase, advisor, streaming }
  event: token       { text }          # human-readable reasoning fragments only
  event: fallback    { reason, advisor }  # if the model failed mid-stream
  event: suggestion  <PricingSuggestionResponse>
  event: error       { message }
```

The stream never silent-drops: every path ends in `suggestion` or `error`. A fallback is announced
before the rule-based answer arrives. Accept/reject in the live panel hits the same `PATCH` as the
queue — the human checkpoint is unchanged. Details: [ADR-8](ADR.md).

### Verifying the console without a browser

```powershell
cd frontend
npm run smoke     # backend must be running
```

`scripts/render-smoke.mjs` server-renders every component against live backend payloads and asserts
that the badges, reasoning, confidence meters, accept/reject controls, stock heat bars,
simulate-sale buttons, and the Ask-live control actually appear. It also renders the trigger and
provenance permutations from fixtures, so the checks hold regardless of what the database currently
contains.

## Configuration

All tunables live in `backend/src/main/resources/application.properties`.

| Property | Default | Purpose |
| --- | --- | --- |
| `commerce.active-strategy` | `aiAdvisor` | Active advisor. Switchable at runtime, no restart. |
| `commerce.agentic-loop-enabled` | `true` | Master switch for the automatic trigger loop |
| `commerce.low-stock-increase-pct` | `10` | Rule-based price bump when stock is below threshold |
| `commerce.velocity-premium-pct` | `5` | Rule-based bump when velocity beats the category average |
| `commerce.velocity-premium-multiplier` | `2.0` | Velocity multiple that earns the premium |
| `commerce.demand-spike-multiplier` | `3.0` | Velocity multiple that fires the DEMAND_SPIKE trigger |
| `commerce.reorder-target-multiplier` | `3` | Reorder target: `(threshold × N) − stock` |
| `commerce.max-ai-price-change-pct` | `50` | Largest LLM price move accepted before it is rejected as absurd |
| `llm.provider` | `groq` | `groq`, `gemini`, or `ollama` |
| `llm.model` | `openai/gpt-oss-20b` | Groq model id (Llama 3.1 instant was retired for free/dev tiers) |
| `llm.api-key` | `${LLM_API_KEY:}` | Resolved from the environment |
| `llm.max-tokens` | `2048` | Completion budget (`max_completion_tokens` on Groq); keep high for reasoning models |
| `llm.timeout-seconds` | `30` | HTTP read timeout for LLM calls |

## The agentic loop

Nothing below asks for a recommendation. A sale is simulated, and suggestions appear on their own.

```
POST /products/{id}/orders          returns immediately
        │
        ├─ Product.recordSale()     stock down, demand velocity up
        └─ publish InventorySignalEvent
                 │  (after commit, on the commerce-* thread pool)
                 ▼
        classify the signal
           stock < reorderThreshold          -> INVENTORY_LOW
           velocity > 3x peer average        -> DEMAND_SPIKE
                 │
                 ├─ skip if a PENDING suggestion already covers this product+trigger+type
                 ├─ active advisor (aiAdvisor), falling back to ruleBased on any failure
                 ▼
        PENDING pricing + reorder suggestions
                 │
                 ▼
        merchandiser accepts  ->  price published / stock received
```

Both triggers are evaluated in one pass, because a single order can satisfy both at once.

## Demo paths (API)

Both products are seeded for this. Suggestions arrive asynchronously, so allow a second (or a few,
with a live LLM) before querying.

**Inventory low** — `PRD-003` starts at stock 8 against a threshold of 15, so one sale keeps it
below and fires `INVENTORY_LOW`:

```bash
curl -X POST http://localhost:8080/products/PRD-003/orders \
     -H "Content-Type: application/json" -d '{"quantity":1}'

curl "http://localhost:8080/pricing-suggestions?status=PENDING"
curl "http://localhost:8080/reorder-suggestions?status=PENDING"
```

**Demand spike** — `PRD-008` sits at velocity 15 in APPAREL against a peer average of 7.0, so the
3x bar is 21. An order of 7 crosses it and fires `DEMAND_SPIKE`:

```bash
curl -X POST http://localhost:8080/products/PRD-008/orders \
     -H "Content-Type: application/json" -d '{"quantity":7}'
```

Note the peer average deliberately **excludes** the product being assessed. Counting a product in
its own category average makes the spike condition `v > 3 x average` unsatisfiable in a small
category — see the corrections section of [ADR.md](ADR.md).

**Accept, and watch the price move:**

```bash
curl -X PATCH http://localhost:8080/pricing-suggestions/1 \
     -H "Content-Type: application/json" -d '{"status":"ACCEPTED"}'

curl http://localhost:8080/products/PRD-003
```

**Stream a price opinion (SSE):**

```bash
curl -N -X POST http://localhost:8080/products/PRD-001/suggest-pricing/stream
```

**Prove the fallback** by switching engines at runtime with no restart:

```bash
curl -X PATCH http://localhost:8080/admin/commerce-strategy \
     -H "Content-Type: application/json" -d '{"activeStrategy":"ruleBased"}'
```

Every suggestion carries `generatedBy`, so `aiAdvisor` versus `ruleBased` is visible in the
response — a fallback can never masquerade as a working AI call.

## A note on the LLM

`commerce.active-strategy` defaults to `aiAdvisor`. With `LLM_API_KEY` set, recommendations come
from Groq (`openai/gpt-oss-20b`) with real reasoning. Without it, every call fails fast and degrades to the
rule-based baseline with a logged warning:

```
WARN  Pricing advisor 'aiAdvisor' failed for PRD-003 (no API key configured for provider groq);
      falling back to 'ruleBased'
```

That is the intended behaviour, not a misconfiguration — the app is fully demonstrable with no key
at all, and the fallback path is exercised constantly rather than only during an incident.

### Exercising the AI path without a key

`tools/fake-llm.js` is a stand-in for an OpenAI-compatible provider, so the real HTTP path — request
construction, streaming, response parsing, bounds validation, fallback — can be driven end to end
offline. Requests with `"stream": true` receive chunked SSE deltas.

```powershell
node tools\fake-llm.js ok

cd backend
.\mvnw.cmd spring-boot:run "-Dspring-boot.run.arguments=--llm.provider=ollama --llm.base-url=http://localhost:11434"
```

`tools\verify-ai.ps1` then walks every failure mode and asserts the outcome of each:

| stub mode | model behaviour | expected |
| --- | --- | --- |
| `ok` | valid JSON within bounds | `generatedBy=aiAdvisor` |
| `fenced` | valid JSON wrapped in markdown and chatter | `generatedBy=aiAdvisor` — parsed anyway |
| `prose` | refuses, returns no JSON | falls back to `ruleBased` |
| `absurd` | recommends 40x the current price | rejected by bounds, falls back |
| `error` | HTTP 429 quota exhausted | falls back |
| `slow` | never responds | read timeout at 12s, falls back |
| `truncated` | streams half a document then stops | parse fails; stream falls back to rules |

Every fallback logs the specific reason, so a degraded call is never silent:

```
WARN  Pricing advisor 'aiAdvisor' failed for PRD-001 (model recommended 999999.00 against a
      current price of 79.99, a 1250055.02% change that exceeds the 50% guardrail);
      falling back to 'ruleBased'
```

Architecture decisions, tradeoffs, SSE streaming, and the sprint 2 extension seams are recorded in
[ADR.md](ADR.md).
