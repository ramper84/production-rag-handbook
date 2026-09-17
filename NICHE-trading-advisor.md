---
title: Trading advisor — Alpaca MCP + CAG/RAG implementation guide
doc_type: niche-deep-dive
scope: time-sensitive
added: 2026-09-06
summary: >
  Phased build guide for a personal trading/investment advisor built on
  PLAYBOOK.md — a paper-trading Alpaca account, a small Alpaca MCP server
  for execution, the order mechanics, and the CAG/RAG advisor project
  itself. Companion to NICHES.md and grounded in two local reference repos:
  tradingview-mcp-jarp (MCP pattern) and AlpacaTradingAgent (Alpaca
  integration, multi-agent pattern to learn from and deliberately not copy).
---

# Trading advisor: Alpaca MCP + CAG/RAG implementation guide

This is the deep-dive for `NICHES.md`'s trading-advisor niche, written because
it's the one you're actually building. It stays deliberately narrower than
`AlpacaTradingAgent` (a fork of the academic *TradingAgents* framework this
guide studied for its Alpaca integration and its debate-heavy multi-agent
pattern) — that repo is the "what a fully-grown version looks like"
reference, not the v1 blueprint. This guide's v1 is a fixed pipeline with one
deterministic gate, per PLAYBOOK.md's own Axis 5 discipline.

## 0. What we're building — a shared library, two front ends, one human gate

**A shared core — `alpaca_client.py`.** The actual `alpaca-py` wrapper (a
port of `AlpacaUtils`, plus the `order_propose`/`order_place` split from
Phase 3). Plain Python functions, no LLM, no MCP, no advice. Both front ends
below import this same module — it is written once, and it is the only code
that ever calls `submit_order`.

**Front end 1 — `trading-advisor`**, the primary path. The `PLAYBOOK.md`
project: reads your thesis and risk rules (CAG), retrieves your decision
history (Axis 2), generates a recommendation with your own LLM API key, and
calls `alpaca_client.order_propose()` **directly, as a Python function call —
no MCP involved.** MCP exists for a client that doesn't know your functions
at compile time; your own backend does, so it just imports them.

**Front end 2 — `alpaca-mcp`**, the secondary, optional path. A thin MCP
wrapper around the same `alpaca_client.py`, for ad-hoc, conversational
access — asking Claude Code (or any other MCP-speaking client) to check a
position or look something up outside of the advisor's own flow. It is not
on the path from a recommendation to an executed trade; it's a convenience
window onto the same account.

**The gate between a recommendation and an execution is a human, in the
advisor's own UI — never a model call, on either front end.** This was
confirmed explicitly rather than assumed: the alternative — the same LLM
call that drafts a recommendation also authorizing its execution — is a
different system (an autotrader, not an advisor) with a much higher safety
bar, and this guide does not build that. See the box at the end of Phase 5.
This mirrors `tradingview-mcp-jarp`'s own open research question — *"what
decisions should always require explicit human confirmation"* — and matches
`AlpacaTradingAgent`'s own default, `auto_execute_trades: False`.

```
 thesis.md, decisions table        alpaca_client.py              Alpaca
        │                          (shared, imported             (paper)
        ▼                          by both front ends)               ▲
┌───────────────────┐                    │                           │
│  trading-advisor   │  order_propose()   │                           │
│  own LLM API key   │───────────────────►│                           │
│  drafts a          │                    │                           │
│  recommendation    │                    │                           │
└─────────┬──────────┘                    │                           │
          │                               │                           │
          ▼                               │                           │
   shown in the advisor's UI              │                           │
          │                               │                           │
          ▼  human clicks "Approve"       │                           │
   advisor backend calls                  │                           │
   order_place() ───────────────────────► │  ─────────────────────────┘
                                           │
                              alpaca-mcp ──┘  (optional, ad-hoc,
                              wraps the same functions for
                              conversational/manual use only)
```

Both front ends wrap the same account and the same functions; neither
requires the other to exist, and `alpaca-mcp` is no longer required for the
advisor's core loop to run end to end.

---

## Phase 1 — Create the Alpaca account

1. Sign up at Alpaca Markets. **Paper trading requires no funding and no
   broker approval** — it's a simulated account with fake cash, built
   specifically for testing algorithmic and AI-driven strategies. This is
   meaningfully lower-friction than `tradingview-mcp-jarp`'s situation
   (undocumented internal APIs, a page of legal caveats) — Alpaca's paper
   API is the sanctioned, intended way to do exactly this.
2. From the dashboard, generate a **paper** API key and secret. Live keys
   are a separate pair, generated separately, and nothing in this guide
   touches them.
3. Set three environment variables (these are the same three
   `AlpacaTradingAgent`'s `env.sample` uses):
   ```bash
   ALPACA_API_KEY=...
   ALPACA_SECRET_KEY=...
   ALPACA_USE_PAPER=True
   ```
4. Verify the account is reachable before building anything on top of it:
   ```python
   from alpaca.trading.client import TradingClient
   client = TradingClient(api_key, secret_key, paper=True)
   account = client.get_account()
   print(account.cash, account.buying_power, account.equity)
   ```
   If this prints numbers (Alpaca seeds paper accounts with $100,000 by
   default), the account is live and ready. Nothing else in this guide
   should be attempted before this works.

You will not need live keys for anything in this guide. If you ever
promote this past paper trading, that is Phase 6's problem, not this one's.

---

## Phase 2 — Build the shared core, then the Alpaca MCP server

### Why Python, and why not reinvent the wrapper

`AlpacaTradingAgent`'s `tradingagents/dataflows/alpaca_utils.py` already has
a working, tested `AlpacaUtils` class covering exactly the surface needed
here — account info, positions, order placement, order cancellation, asset
lookup, and both stock and crypto market data clients. Port its methods
almost line-for-line into `alpaca_client.py` (below) — there is no reason to
re-derive `MarketOrderRequest` construction or the stock/crypto
`TimeInForce` distinction (Phase 3) from scratch.

### Build `alpaca_client.py` as its own small package, not inside the MCP server

Per §0, this module is imported by **both** `trading-advisor` (directly, no
MCP) and `alpaca-mcp` (wrapped as tools). Put it in its own tiny installable
package — `alpaca-core/`, a local `pip install -e` dependency for both other
repos — rather than writing it once inside `alpaca-mcp/` and copying it into
`trading-advisor` later. Two copies of `order_propose`'s guardrail logic
drifting apart is exactly the kind of duplication this whole set of guides
has been keeping to one source of truth throughout.

```
alpaca-core/
├── alpaca_client.py      # AlpacaUtils port: account, positions, orders,
│                          #   assets, market data
├── guardrails.py          # order_propose's deterministic caps (Phase 3)
├── tests/
│   └── test_order_guardrails.py   # tested without hitting Alpaca at all
└── pyproject.toml
```

`alpaca-mcp` becomes thin on top of it:

```
alpaca-mcp/
├── src/
│   ├── server.py            # MCP server entrypoint, stdio transport
│   └── tools/                # each tool is a few lines calling alpaca-core
│       ├── account.py       # account_get
│       ├── positions.py     # positions_get, positions_close
│       ├── orders.py        # orders_get, order_propose, order_place, order_cancel
│       ├── assets.py        # assets_get
│       └── market_data.py   # quote_get, bars_get, portfolio_history_get
├── .env.example
├── .gitignore                # .env, same discipline as PLAYBOOK.md §11.1
├── pyproject.toml            # depends on alpaca-core
└── README.md
```

No `docker-compose.yml` here — an MCP server is a local process an
MCP-speaking client launches directly (see the config snippet below), not a
service with its own containers. That's a real, deliberate difference from
the `trading-advisor` project in Phase 4, which does need one — and per §0,
`alpaca-mcp` itself is now the optional, secondary front end. Build
`alpaca-core` and get `order_propose`'s guardrails right first; `alpaca-mcp`
is a thin, almost mechanical wrapper once that exists.

### Tool reference

| Tool | Does | Safety note |
|---|---|---|
| `alpaca_health_check` | Confirms the API keys work and reports paper/live mode | Call this first, always — mirrors `tv_health_check` |
| `account_get` | Cash, buying power, equity, day's P&L | Read-only |
| `positions_get` | All open positions | Read-only |
| `positions_close` | Liquidate a position, by symbol, optionally partial (`percentage`) | Still a write — same confirmation rule as `order_place` |
| `orders_get` | List orders, filterable by status/symbol | Read-only |
| `assets_get` | Is this symbol tradable? Fractionable? Shortable? | Read-only — call before proposing an order on an unfamiliar symbol |
| `quote_get` / `bars_get` | Latest quote / historical bars, stock or crypto | Read-only |
| `portfolio_history_get` | Equity curve over time | Read-only — feeds Phase 4's evals |
| `order_propose` | **New, not in `AlpacaUtils`.** Prices an order and runs the deterministic caps below, returns a proposal + a confirmation token. Places nothing. | The only way to reach `order_place` |
| `order_place` | Submits the order — **requires the token `order_propose` just returned** | See Phase 3 |
| `order_cancel` | Cancels an open order by id | Write, but reversible in intent (undoing an action, not taking one) |

`order_propose` doesn't exist in `AlpacaTradingAgent` — it's this guide's
addition, and it's the answer to "how do we buy safely" in Phase 3.

### Configuration

```bash
# .env.example
ALPACA_API_KEY=
ALPACA_SECRET_KEY=
ALPACA_USE_PAPER=True

# Deterministic order guardrails — enforced in code, never left to the model
MAX_ORDER_NOTIONAL_USD=500       # no single order above this without editing config
MAX_OPEN_POSITIONS=10            # order_propose refuses a new symbol past this count
MIN_CASH_BUFFER_USD=1000         # order_propose refuses if it would breach this
PROPOSAL_TOKEN_TTL_SECONDS=300   # a stale proposal can't be replayed as a confirmation
```

These four caps belong in **`alpaca-core/guardrails.py`**, read from its own
config with these as defaults — not duplicated separately inside
`alpaca-mcp` and `trading-advisor`. They are `order_propose`'s own version of
Axis 4's deterministic Critic — code, not a model call, checked before
Alpaca ever sees the order, and enforced identically no matter which front
end called it. Get them wrong here and no amount of good advice upstream in
Phase 4 saves you from a fat-fingered or hallucinated order size.

### Wiring the optional ad-hoc interface into Claude Code

```json
{
  "mcpServers": {
    "alpaca": {
      "command": "python",
      "args": ["/absolute/path/to/alpaca-mcp/src/server.py"]
    }
  }
}
```

Same shape as `tradingview-mcp-jarp`'s own `.mcp.json` entry — merge with
whatever servers are already configured, don't overwrite them.

### Verify

1. Restart Claude Code, ask it to run `alpaca_health_check`.
2. Ask it to run `account_get` and confirm the numbers match Phase 1's
   verification script.
3. Do **not** attempt a real order yet — that's Phase 3, deliberately kept
   separate so the confirmation flow gets exercised once, on purpose, before
   it's ever exercised by a real recommendation from Phase 4.

This verifies the optional, ad-hoc path only. The primary path —
`trading-advisor` calling `alpaca-core` directly — is verified in Phase 5,
and needs no Claude Code session, MCP config, or restart at all: it's an
`import` and a function call inside a Python process you already control.

---

## Phase 3 — How to buy (and sell)

### The mechanics, exactly as `alpaca-py` expects them

```python
from alpaca.trading.requests import MarketOrderRequest
from alpaca.trading.enums import OrderSide, TimeInForce

is_crypto = "/" in symbol.upper()
order_request = MarketOrderRequest(
    symbol=symbol.upper().replace("/", ""),   # crypto symbols drop the slash
    side=OrderSide.BUY,                        # or OrderSide.SELL
    time_in_force=TimeInForce.GTC if is_crypto else TimeInForce.DAY,
    notional=100.0,   # dollar amount — OR qty=1, never both
)
order = trading_client.submit_order(order_request)
```

Two things worth stating because they're easy to get wrong: crypto orders on
Alpaca only accept `TimeInForce.GTC`, stocks default to `TimeInForce.DAY`;
and `notional` (a dollar amount, enabling fractional shares) and `qty` (a
share count) are mutually exclusive on the same request. `AlpacaUtils`
already branches on this correctly — reuse it rather than re-deriving it.

### The two-call safety pattern — this is the actual answer to "how do we buy"

Never let one tool call both decide the size and submit the order. Split it:

**Call 1 — `order_propose`**
```json
{"symbol": "NVDA", "side": "buy", "notional": 100.0}
```
returns
```json
{
  "success": true,
  "estimated_shares": 0.612,
  "estimated_price": 163.40,
  "checks": {
    "within_notional_cap": true,
    "open_positions_after": 4,
    "cash_buffer_after": 8412.11,
    "symbol_tradable": true
  },
  "confirmation_token": "prop_8f3a...",
  "expires_at": "2026-09-06T14:35:00Z"
}
```
If any `checks` entry is false, `success` is false and no token is issued —
the deterministic caps from Phase 2 run here, not after.

**Call 2 — `order_place`**
```json
{"confirmation_token": "prop_8f3a..."}
```
This is the only call that reaches `submit_order`. It re-validates the token
hasn't expired (`PROPOSAL_TOKEN_TTL_SECONDS`) and hasn't already been used,
then places the order and returns the Alpaca order id.

**Why two calls and not one `place_order(symbol, side, amount)`:** a single
call conflates "here's what I want to do" with "do it," and an LLM
hallucinating or misreading a suggested size has nothing to catch it. Two
calls give you a moment — the proposal is a real, inspectable object a human
(or you, reading Claude's response) can see before the second call ever
fires. This is the concrete, buildable version of `AlpacaTradingAgent`'s
`auto_execute_trades: False` default and `tradingview-mcp-jarp`'s open
question about human-in-the-loop boundaries.

### Selling / closing

`positions_close` takes the same shape, with an optional `percentage`
(`AlpacaUtils.close_position` already supports partial closes) — treat it
with the same propose/confirm split as buying; closing a position is still a
write against real (paper) capital.

### What this phase deliberately does not build

No path from a recommendation straight to `order_place` without a human
reading the `order_propose` output first. If you want scheduled, unattended
execution later, that is a explicit, named decision to make once the advisor
has a track record — not a default this guide ships with.

---

## Phase 4 — The advisor implementation

This is a real `PLAYBOOK.md` project. Run its process for real rather than
skipping to code — the architecture decision below is the output of that
process, not a given.

### Architecture decision (PLAYBOOK.md §2, walked through)

| Axis | Outcome | Why |
|---|---|---|
| 1 — CAG | **Yes**, for your thesis and risk rules | Small, stable, written once, revised deliberately — same shape as `rules.json` / `risk_rules` in both reference repos |
| 2 — SQL-retrieval RAG | **Yes**, for decision history | "How did my last 10 BUY calls perform" names its entities exactly (symbol, date) — no embedding needed |
| 3 — Vector RAG | **Deferred** | News/fundamentals text is genuinely paraphrastic, but start by passing the day's retrieved articles straight into context (live-assembled CAG) before building an index — only add real chunking+embeddings against a measured gap |
| 4 — Agentic (Critic) | **Yes, deterministic** | A risk Critic checking a proposed recommendation against your own risk rules is code with a correct answer, not a model call |
| 5 — Orchestration | **No** | `AlpacaTradingAgent`'s 5-analyst debate looks like orchestration but the specialist set is fixed and always all five — that's Phase composition, not Axis 5's dynamic routing. A fixed pipeline is enough |
| Live pass-through | **Yes**, for prices/positions | Quotes and positions change continuously; never persist the live tick, only the durable exhaust — the recommendation, and later, the realized outcome |

### Project tier

**Lean.** One primary composition (CAG rules + SQL history feeding one
generation call, gated by one deterministic Critic) — this does not need a
conductor or a `generation/{cag,rag,agentic}/` split. Revisit only if a
second genuinely separate capability appears (e.g., a portfolio-wide
rebalancing advisor distinct from a per-symbol one) — decide the composition
strategy explicitly at that point, per §2's "Composing more than one answer."

### Directory skeleton

```
trading-advisor/
├── app/
│   ├── config.py
│   ├── schemas.py
│   ├── main.py
│   ├── services/
│   │   └── llm_service.py       # the one generation call — alpaca_client
│   │                             #   comes from the alpaca-core dependency,
│   │                             #   not a local copy (see Phase 2)
│   ├── rules/
│   │   └── thesis.md            # CAG: your investment thesis + risk rules, versioned
│   ├── guardrails/
│   │   └── risk_critic.py       # Axis 4: deterministic checks against rules/thesis.md
│   ├── ingest/
│   │   └── decision_log_store.py  # Axis 2: the Postgres read/write layer
│   ├── prompts/
│   │   └── advisor/v1/{system,user}.j2
│   └── routers/
│       └── advise.py            # POST /advise — the one endpoint
├── migrations/                  # Alembic — decision_log's schema, see below
├── evals/
│   └── golden_queries.json      # seeded from this guide's example queries
├── tests/
├── streamlit_app.py             # shows a proposal, a "looks right" button — no execution here
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── ARCHITECTURE.md
└── pyproject.toml
```

### `rules/thesis.md` — a starter template, for when you don't know trading yet

Not knowing trading is not a blocker to writing v1 of this file — it's
arguably the right starting position. Risk rules matter more than market
opinions, and the standard, well-evidenced default for someone without
special expertise is diversified funds over individual stock-picking, not
a personal edge you don't have yet. Fill this in, adjust only with a stated
reason, and let real history in the `decisions` table justify loosening it
later — never the other way around. This is educational scaffolding, not
personalized financial advice; adjust the defaults to your actual situation
or a licensed advisor's guidance, not this document's word for it.

```markdown
# Investment thesis and risk rules
version: v1
last_reviewed: [FILL ME: today's date]

## Time horizon and purpose
[FILL ME — if unsure, this default is a reasonable starting point: "Long-term
(5+ years). This is money I could afford to have tied up, and could afford
to lose, without it changing my life."]

## What this account is allowed to hold
[FILL ME — if unsure, this default is a reasonable starting point:
"Diversified ETFs (e.g. a total-market or S&P 500 fund) as the default
holding. Individual stocks only up to [FILL ME: a small, clearly bounded %,
e.g. 10%] of the account, and only in companies I've actually looked into."]

## Risk rules — matter more than any market opinion, and need no expertise
- Never risk more than [FILL ME, default: 1-2%] of total account equity on
  a single position.
- No single sector above [FILL ME, default: 25%] of the account.
- Maintain a cash buffer of at least [FILL ME, default: 10%] at all times.
- No margin, no shorting, until [FILL ME: a stated milestone — e.g. "12
  months of reviewed paper-trading history"].
- If a position drops [FILL ME, default: 15%] from entry, the advisor flags
  it for review. It does not auto-sell — Phase 5's human gate still applies.
- Maximum [FILL ME, default: 5] open individual-stock positions at once.
- **Default to HOLD.** Absent a specific, checkable reason to act, the
  advisor's job is to say nothing needs to change — not to find a reason to
  trade. `risk_critic.py` should treat an unjustified BUY/SELL as a bigger
  red flag than a HOLD.

## What would make me override the advisor's own caution
[FILL ME — fine to leave blank for now. This section exists so a future
override is a decision made once, deliberately, and written down before the
moment it's needed under pressure — not a pattern of ignoring risk rules
whenever the market gets stressful.]

## Revision log — ADR-style; add an entry only when a rule actually changes
- [FILL ME: today's date] — v1 created.
```

If you'd rather draft your own version with help than fill in the defaults
above cold, that's a five-question conversation, not research: time
horizon, how you'd feel about a 20% drawdown, individual stocks vs. funds,
anything you'd want excluded, and how much of your capital this account is
allowed to touch. Answer those in plain language and the file above writes
itself.

### The decision log — Axis 2's actual table

```sql
CREATE TABLE decisions (
    id              BIGSERIAL PRIMARY KEY,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    symbol          TEXT NOT NULL,
    asset_class     TEXT NOT NULL,              -- 'stock' | 'crypto'
    rules_version   TEXT NOT NULL,               -- which thesis.md revision produced this
    source_reads    JSONB NOT NULL,              -- the live pass-through snapshot: quote, indicators, news read
    recommendation  TEXT NOT NULL,               -- BUY | SELL | HOLD
    confidence      NUMERIC,
    rationale       TEXT NOT NULL,
    critic_verdict  JSONB NOT NULL,              -- risk_critic.py's structured output
    human_decision  TEXT,                        -- approved | rejected | ignored, filled in later
    alpaca_order_id TEXT,                        -- filled in only if approved and executed
    outcome_5d      NUMERIC,                     -- filled in by a scheduled job, once knowable
    outcome_20d     NUMERIC,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Every "how often was I right" query (Phase 4's whole reason for existing)
is a `WHERE`/`GROUP BY` over this one table — no vector search involved. It
is also, directly, `tradingview-mcp-jarp`'s flat-JSON-file gap fixed exactly
the way that repo's own `RESEARCH.md` never got around to.

### The deterministic risk Critic

Concrete checks, all code, none a model call, all sourced from
`rules/thesis.md`'s own risk-rules section (mirroring `rules.json`'s
`risk_rules` and `AlpacaTradingAgent`'s risk-management team's *intent*
without its three-way LLM debate):

- Proposed position size vs. a max % of current equity.
- Sector/symbol concentration vs. a stated cap.
- Cash buffer maintained after the trade.
- A cool-down rule ("no new position in a symbol closed at a loss in the
  last N days") if your thesis states one.

This runs **after** the recommendation is generated and **before** it's
shown in the Streamlit UI — a recommendation that fails the critic is shown
as such, with the specific rule it broke, never silently upgraded or hidden.

### Freshness

`alpaca-core`'s `alpaca_client.py`, imported here, is the live pass-through
client from `PLAYBOOK.md`'s cross-cutting note — it reads Alpaca's
quote/position endpoints fresh on every `/advise` call, behind a short-TTL
Redis cache matching the stated freshness budget (seconds, not "as fresh as
possible").
Nothing about the live quote itself is stored; only the resulting row in
`decisions` is.

### `docker-compose.yml`

Use `PLAYBOOK.md` §6's template as-is: `postgres` (the `decisions` table),
`redis` (the freshness-budget cache), `api`, and `streamlit`. No new
services — this project does not need pgvector active (Axis 3 was deferred),
though `pgvector/pgvector:pg16` as the image costs nothing to keep per §3's
"one fewer moving part" reasoning if you expect to add the news index later.

---

## Phase 5 — Wiring one full cycle

1. `POST /advise` with a symbol → `alpaca_client.py` (from `alpaca-core`)
   reads a live quote and position snapshot → `llm_service.py`, using the
   advisor's own LLM API key, generates a recommendation grounded in
   `thesis.md` (CAG) and the last N relevant rows from `decisions` (Axis 2)
   → the recommendation calls `alpaca_client.order_propose()` **directly, as
   a Python function call** — no MCP, no separate process — to get real
   pricing and run the deterministic caps → `risk_critic.py` gates the
   whole thing → the row is written to `decisions` with `human_decision`
   null.
2. The Streamlit UI shows the recommendation, its rationale, the priced
   proposal, and the critic's verdict. A person reads it.
3. **If they approve, that click is the entire authorization.** The
   "Approve" button in the advisor's own UI calls `POST /advise/{id}/approve`
   on the advisor's backend, which calls `alpaca_client.order_place()` with
   the proposal's confirmation token — still no MCP, no LLM call in this
   step, on purpose. `alpaca-mcp` is not part of this path at all; it exists
   only for the separate, ad-hoc case of asking Claude Code to look at or
   act on the account outside of a recommendation the advisor generated.
4. The resulting Alpaca order id is recorded back onto the `decisions` row
   (`human_decision`, `alpaca_order_id`) in the same request — no manual
   copy-paste step, because there is no longer a context switch to another
   tool for this to happen across.
5. Days later, a scheduled job (or a manual query against `portfolio_history_get`
   / Alpaca's own account activity) fills in `outcome_5d` / `outcome_20d`.
   This is what makes Phase 4's golden queries answerable with real numbers
   instead of vibes.

> **Where the human gate actually lives.** Step 1's LLM call only ever
> reaches `order_propose` — pricing and guardrail-checking, never
> `submit_order`. Step 3's `order_place` call is triggered by an HTTP
> request from a browser button click, not by any model output. No prompt,
> no tool-call response, and no automated retry loop can reach `order_place`
> on its own; only a human's button click can. That is the one property this
> whole guide is built to hold onto — replace the UI, replace the LLM
> provider, replace `alpaca-mcp` entirely, and this still has to be true.

---

## Phase 6 — Safety, scope, and cost

- **Paper only, until a deliberate, separate decision to promote.** Nothing
  in either repo should read live keys by default; flipping
  `ALPACA_USE_PAPER` is a loud, manual, one-line change you make once you
  trust the track record in `decisions`, not a config default anyone
  inherits by accident.
- **This is personal-use tooling advising its own operator**, the same
  posture both reference repos state explicitly in their own disclaimers —
  not investment advice offered to anyone else, and not wired for unattended
  execution.
- **Cost.** One generation call per `/advise` request is cheap. Resist
  `AlpacaTradingAgent`'s instinct to fan out to five parallel analyst calls
  per symbol before you've measured that the single-call version is actually
  insufficient — that fan-out is exactly the kind of speculative complexity
  `PLAYBOOK.md`'s Axis 5 and s10-02's "measure the gap first" discipline
  argue against building on day one.

---

## Build order

Phase 1 → Phase 2 → Phase 3 (verify the propose/confirm loop manually, with
no advisor involved yet) → Phase 4, scaffolded for real via *"create a
project based on PLAYBOOK.md"* → Phase 5 → Phase 6 review before you trust
it with anything past a handful of paper trades.
