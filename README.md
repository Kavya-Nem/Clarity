v1.0.0
# jachacks-fintech

**What does sending money home actually cost?**

International remittances cost 5–10% of the amount sent, and most of that cost is
invisible. Providers advertise the transfer fee and earn quietly on the exchange
rate — quoting you a rate a few percent worse than the mid-market rate and
pocketing the difference. On a $200 transfer to a thin corridor, the "$0 fee"
option can be the most expensive one on the page.

This is a comparison tool that prices every provider on a corridor the way the
World Bank does: **total cost = advertised fee + exchange-rate margin**. It shows
the split, ranks by what you actually pay, and flags the low-fee/high-spread trap.
An AI advisor takes the request in plain English and explains the result — using
only numbers the pricing engine computed.

Built in [Jac](https://www.jaseci.org/) for JacHacks. **~72% Jac** (1,775 of 2,467
lines): the graph model, the pricing engine, the API, the AI agent, and the entire
React UI are all Jac. Python holds the CSV loader; CSS holds the styling.

---

## The idea in one screen

```
Send 200 USD to Mexico

  Wise            Bank account → Bank deposit    ▇▏               1.69 USD   0.85%
  Remitly         Bank account → Bank deposit    ▇▇▇▇▇▇           3.54 USD   1.77%
  Western Union   Bank account → Bank deposit    ▏▇▇▇▇▇▇▇▇▇▇▇     7.70 USD   3.85%
                                                 ↑        ↑
                                          fee ───┘        └─── exchange-rate margin
```

Western Union advertises a **$0 fee** and is the most expensive option shown. That
gap is the product.

## What's in it

| Feature | Where |
|---|---|
| Providers, corridors and products modelled as a graph | `corridors.sv.jac` |
| World-Bank-method cost engine (fee + spread, effective rate, ETA) | `corridors.sv.jac` |
| Graph seeding and corridor queries | `market.sv.jac` |
| REST + RPC API | `endpoints.sv.jac` |
| AI advisor: natural-language intent → grounded recommendation | `advisor.sv.jac` |
| React UI (form, stat tiles, stacked cost bars, advisor panel) | `frontend.cl.jac`, `components/` |
| Reference-data loader | `market_data.py` |
| Pricing tests | `pricing_tests.jac` |

### The graph model

Object-Spatial Programming earns its place here. The market is a graph, and
**every `Serves` edge is one purchasable product**:

```
root ── MarketNode ──┬── ProviderNode (Wise, Western Union, …)
                     └── CorridorNode (US → MX, GB → NG, …)

ProviderNode ─:Serves(pay_in, pay_out, fixed_fee, pct_fee_bps,
                      fx_margin_bps, eta_minutes):─> CorridorNode
```

A provider has several edges into the same corridor — bank→bank, debit→cash
pickup, and so on — each with its own fee and spread. Quoting a corridor is just
reading the edges that land on it:

```jac
for provider in [market -->[?:ProviderNode]] {
    for offer in [edge provider ->:Serves:-> corridor] {
        quotes.append(cost_offer(provider, offer, corridor, amount));
    }
}
```

### The cost model

Fee is deducted first, then the remainder is converted at the provider's rate —
the order every major provider uses:

```
fee            = fixed_fee + amount × pct_fee_bps/10000   (floored at min_fee)
converted      = amount − fee
offered_rate   = mid_market_rate × (1 − fx_margin_bps/10000)
received       = converted × offered_rate

fx_spread_cost = converted × fx_margin_bps/10000     ← the hidden part
total_cost     = fee + fx_spread_cost
total_cost_pct = total_cost / amount × 100
```

`pricing_tests.jac` pins this against hand-computed values, including the case
that matters most — a zero-fee provider losing to an honest one.

### The AI advisor

Two LLM steps wrap a deterministic core:

1. **`parse_intent`** — "I need to send $300 to my mum in Manila for rent"
   → `{amount: 300, send_country: "US", receive_country: "PH", priority: "cost"}`
2. **`explain`** — takes the *already computed* quotes and writes the
   recommendation.

The model chooses and explains; **it never invents a fee or a rate**. Every figure
it can cite is computed by the pricing engine and handed to it as context. That is
deliberate: a tool about hidden costs cannot afford hallucinated numbers.

If no model is configured, both steps fall back to deterministic Jac, so the app
works end to end with no API key.

---

## Running it locally

You need [Jac](https://www.jaseci.org/) on your PATH. Built and tested against
`jac 0.34.5`, Python 3.14, Node 26.

```bash
git clone git@github.com:sidbar258/jachacks-fintech.git
cd jachacks-fintech

jac install                   # Python + npm dependencies, from jac.toml
jac start --dev main.jac      # app + API, with hot reload
```

Then open **<http://localhost:8000>**. That is the whole setup — no database to
provision, no API key required, no seed script to run.

| URL | What it is |
|---|---|
| <http://localhost:8000> | The app |
| <http://localhost:8001> | The API |
| <http://localhost:8001/docs> | Swagger, generated from the endpoint signatures |
| <http://localhost:8001/graph> | Live visualiser for the provider/corridor graph |

The dev server proxies API routes, so `/function/...` and `/walker/...` answer on
port 8000 as well — either port works from curl.

**The first comparison is slower than the rest.** The graph seeds itself from
`data/*.csv` on the first query — 8 providers, 45 corridors, 984 `Serves` edges —
and is reused from then on. Seeding is under a second; the wait on a cold start is
Vite building the client.

`--dev` gives hot reload for client files. Editing a **server** file (`.sv.jac`,
`main.jac`, `market_data.py`) needs a restart to take effect.

### The other commands

```bash
jac check .                   # type-check every file (20 files)
jac test pricing_tests.jac    # the pricing tests (8 of them)
jac clean                     # drop build artifacts under .jac/
jac guide jac-core-cheatsheet # bundled Jac language reference
```

### After editing the pricing data

The graph is built once and cached, so a change to `data/*.csv` is not picked up
until you rebuild it. No restart needed:

```bash
curl -X POST localhost:8001/function/refresh_data \
  -H 'Content-Type: application/json' -d '{}'
```

### If it stops working

**"No provider matches those payment and payout choices" on every corridor.**
The cached graph is stale — most likely a `.sv.jac` node archetype gained or lost
a field while a graph built against the old shape was still on disk. Rebuild it
from scratch:

```bash
pkill -f "jac start"          # stop the server first
rm -f .jac/data/*             # the whole directory, not just anchor_store.db
jac start --dev main.jac
```

Delete *everything* in `.jac/data/`. Removing `anchor_store.db` while leaving
`users.db` behind produces `'JacScaleUserManager' object has no attribute '_lock'`.
Nothing is lost either way: the graph is derived entirely from the CSVs.

**The app is served on port 8002, or you are looking at stale results.** An
earlier `jac start` is still holding 8000, and the new one has moved up. Check
the banner it prints on boot, and clear the strays:

```bash
lsof -nP -iTCP -sTCP:LISTEN | grep -E ':(8000|8001|8002)'
pkill -9 -f "jac start"; pkill -9 -f vite
```

### The AI advisor (optional)

Set a key for the provider configured in `jac.toml` under `[byllm.model]`:

```bash
export ANTHROPIC_API_KEY=...        # default: anthropic/claude-opus-5
```

Or switch `default_model` to `gpt-4o-mini` (`OPENAI_API_KEY`), `ollama/llama3`
(local daemon), or a fully local model:

```bash
jac install 'byllm[local]' && jac model pull gemma-4-e4b   # then: local:gemma-4-e4b
```

Without any of these the advisor still answers — from the deterministic fallback.

### Calling the API directly

With the server running:

```bash
# Function RPC — typed objects on the wire
curl -X POST localhost:8001/function/compare -H 'Content-Type: application/json' \
  -d '{"send_country":"US","receive_country":"MX","amount":200,"pay_in":"","pay_out":""}'

# Walker REST
curl -X POST localhost:8001/walker/CompareCorridor -H 'Content-Type: application/json' \
  -d '{"send_country":"US","receive_country":"MX","amount":200}'

# The AI advisor
curl -X POST localhost:8001/function/advise -H 'Content-Type: application/json' \
  -d '{"message":"I want to send $300 from the US to Mexico"}'
```

Every `def:pub` is published at `/function/<name>` and every `walker:pub` at
`/walker/<name>`, so the Swagger page at `/docs` stays in step with the code
without anything being written down twice.

---

## Reference data

Pricing lives in `data/` as CSV and is loaded by `market_data.py`. The graph
seeds itself on first query; `POST /function/refresh_data` rebuilds it after an
edit.

| File | Columns |
|---|---|
| `currencies.csv` | `currency,units_per_usd` |
| `countries.csv` | `code,name,currency,flag,role,region,difficulty` |
| `providers.csv` | `slug,name,blurb,network` |
| `offers.csv` | `provider_slug,send_country,receive_country,pay_in,pay_out,fixed_fee,pct_fee_bps,min_fee,fx_margin_bps,eta_minutes` |

`role` is `send`, `receive`, or `both`. `difficulty` scales how wide a spread
providers charge on a corridor (1.0 = dense and competitive; thin corridors run
higher). Fees are in the send currency; `*_bps` columns are basis points
(100 bps = 1%).

> **The pricing data is illustrative sample data**, modelled on the shape of
> published remittance fees. It is **not** a live quote and must not be used to
> choose a provider for a real transfer. The app says so on screen. To make it
> authoritative, point the loader at real provider data or replace the readers in
> `market_data.py` with a pricing API client.

## Accessibility & design notes

The fee/spread palette is validated for contrast and colour-vision separation
against both the light and dark surfaces (worst-pair ΔE 24.7 protan / 33.6 normal
in light mode). The two cost segments always carry a legend, so identity is never
colour-alone, and the numeric breakdown is printed beside every bar.
