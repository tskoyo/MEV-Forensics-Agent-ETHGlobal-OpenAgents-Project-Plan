# MEV Forensics Agent — ETHGlobal OpenAgents Project Plan

*Version 2.0 · MVP scope · Updated 2026-04-21*


## Table of contents
1. [The problem](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan/blob/master/README.md#1-the-problem)
2. [What is MEV (primer)](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#2-what-is-mev-brief-primer)
3. [Why we're building this](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#3-why-were-building-this)
4. [Existing solutions and where they stop](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#4-existing-solutions-and-where-they-stop)
5. [How we go one step further](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#5-how-we-go-one-step-further)
6. [Scope](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#6-scope)
7. [Failure taxonomy](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#7-failure-taxonomy)
8. [Architecture](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#8-architecture)
9. [Tech stack](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#9-tech-stack)
10. [Tool layer](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#10-tool-layer)
11. [Agent design](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#11-agent-design)
12. [Data & demo strategy](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#12-data--demo-strategy)
13. [Execution plan (hackathon week)](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#13-execution-plan-hackathon-week)
14. [Risks and mitigations](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#14-risks-and-mitigations)
15. [Pitch narrative](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#15-pitch-narrative-5-min)
16. [Open questions](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#16-open-questions)

---

## 1. The problem

DEX trading bots — especially MEV searchers running arbitrage, liquidation, or sandwich strategies — operate in a brutally adversarial, high-frequency environment. Every day these bots submit thousands of transactions. Some win. Many lose money or underperform expectations. And when they do, **the operator has no fast way to find out why.**

Today, when a searcher sees a trade that earned $50 instead of the expected $75, their workflow is:

1. Open Etherscan to the tx
2. Copy the block number, open it in another tab for neighboring txs
3. Open Tenderly to load the trace
4. Manually scan each competing tx in the same block looking for anyone who touched the same pool
5. Manually compute what pool state *would* have given them the expected outcome
6. Try to re-simulate the trade against different historical states
7. Write themselves a note and move on

**This takes 15–30 minutes per trade.** A searcher running 500 trades/day cannot investigate more than a handful. Most post-mortems never happen. Patterns go unnoticed. Money leaks invisibly.

**The problem we are solving:** MEV searchers lose money and don't know why, because investigating a single trade is manual, slow, and requires switching between four or five different tools. There is no autonomous analyst that does the detective work for them.

---

## 2. What is MEV (brief primer)

**MEV = Maximal Extractable Value.** The profit sophisticated actors can extract from the order in which transactions are included in a blockchain block.

Because txs on Ethereum (and most chains) are executed in a specific order decided by block builders, and because DeFi pool state (prices, liquidity) changes with every executed tx, being in the right position in a block can mean the difference between profit and loss.

**The main flavors of MEV:**
- **Arbitrage** — price discrepancies between DEX pools (e.g., ETH is $2,000 on Uniswap but $2,005 on Sushi); bots buy low and sell high in a single atomic transaction
- **Sandwich attacks** — a bot sees your pending swap, front-runs it to push the price against you, lets your trade execute at the worse price, then back-runs to sell at a profit
- **Liquidations** — repaying bad loans on lending protocols in exchange for collateral at a discount

Thousands of bots compete for these opportunities. The game is won and lost in **tx ordering within a single block**, measured in gas priority fees and milliseconds. A bot that sees a $500 opportunity may realize $200 because another bot got there first, or because pool state moved between detection and inclusion.

**The forensic question that matters:** "I saw a $500 opportunity. I got $200. Where did the other $300 go — and could I have captured it?"

That's what this tool answers.

---

## 3. Why we're building this

1. **The pain is real and underserved.** Solo MEV searchers are a small but technically sophisticated user base. They manually investigate trades today because no tool does it for them. Classic agentic automation shape: bounded domain, clear tools, verifiable outputs.
2. **LLM tool-use only recently became reliable enough.** Chaining 5–8 tool calls with Tenderly, RPC lookups, and simulations used to be a pipe dream. Claude handles it today. New product shape enabled by recent capability.
3. **ETHGlobal OpenAgents is literally about agents.** Matches the theme: autonomous tool use, multi-step reasoning, verifiable results, DeFi domain, obvious Tenderly + Uniswap Foundation sponsor fit.

---

## 4. Existing solutions and where they stop

| Tool | What it does | What it doesn't do |
|---|---|---|
| **EigenPhi** | Classifies MEV (arb, sandwich, liquidation), profit per searcher, victim labeling, real-time stream | No counterfactuals, no expected-vs-realized, no per-tx frontrun attribution, no NL explanation, no chat |
| **libMEV** | Searcher leaderboards (ELO), bundle classification, aggregate stats | Same gaps; explicitly framed as "detective work for you to do" |
| **Tenderly** | Simulation, call traces, debugger, alerts | Powerful *evidence layer*, but a tool — it doesn't reason, you drive it |
| **Etherscan / Phalcon** | Raw tx + decoded traces | Manual; no reasoning |
| **Nansen AI / Arkham** | Wallet labeling + some LLM search over aggregate data | Surface-level; no trace-level reasoning, no simulation, no MEV forensics depth |

EigenPhi's own docs describe their arbitrage profile as *"retrospective summary of what happened, not analytical depth about why outcomes diverged from alternatives."* That sentence is the gap.

---

## 5. How we go one step further

We put a disciplined, evidence-citing agent on top of the existing data layer.

1. **Per-trade investigation**, not aggregate stats. Point at a tx, ask "why?"
2. **Expected vs realized PnL reconstruction.** The agent simulates the trade against pool state at block N-1 to compute what it *should* have earned, then explains the gap.
3. **Counterfactuals.** "A competing tx at a lower block index consumed the liquidity before you. Here's the simulation."
4. **Evidence discipline.** Every claim backed by a specific tool result. No general-knowledge reasoning. If unconfirmed, agent says `unknown` — and that's a feature.
5. **Conversational.** Multi-turn follow-ups reuse prior tool results.
6. **Live tool-call timeline.** The UI streams agent actions in real time.

**Positioning in one sentence:**
> EigenPhi tells you what class of MEV a tx was. Tenderly gives you the trace. None of them tell you **why your specific trade underperformed** or **what would have won it**. We're the layer on top — an autonomous investigator that uses those same data sources plus simulation to produce evidence-cited forensic reports, conversationally.

---

## 6. MVP scope

### In scope (hackathon)
- Single chain: **Base**
- DEX coverage: **Uniswap v2 + v3** only
- Read-only analysis of historical trades
- **2 curated demo trades** from real MEV wallets
- Multi-turn conversational interface
- Web dashboard (frontend engineer owns it)
- **5 investigation tools** backed by Tenderly + RPC
- **2 investigation paths** from the failure taxonomy (see section 7)
- Streaming tool-call timeline in the UI

### Out of scope (hackathon)
- Live trade execution, wallet/key handling
- Multi-chain support
- Curve, Balancer, or any AMM beyond Uni v2/v3
- Automated remediation
- Real-time alerting / push notifications
- Aggregate cross-trade dashboards
- Auth, multi-user, any infra beyond single-machine demo

### Explicit "won't build" list
- Our own MEV bot
- Smart contracts
- Mempool monitoring
- Private order flow integration
- A backtester
- Any "predict future trades" feature

---

## 7. Failure taxonomy

The agent reasons over a bounded taxonomy. Every trade gets classified into one **outcome** and (optionally) one **root cause**.

### Outcome categories

| Code | Name | MVP |
|---|---|---|
| A2 | `success_underperformed` | ✅ **Primary case.** Tx succeeded but realized < expected. Triggers root-cause search. |
| A9 | `unknown` | ✅ **Trust signal.** Investigated, ruled out known causes, cannot confidently classify. |
| A1 | `success_as_expected` | 🔮 Future — short-circuit, nothing to investigate |
| A3 | `reverted_trivial` | 🔮 Future — short-circuit on revert string |
| A4 | `reverted_nontrivial` | 🔮 Future — requires deep trace-level debugging |
| A5 | `not_landed` | 🔮 Future — requires mempool data |

### Root cause categories

| Code | Name | MVP |
|---|---|---|
| B1 | `frontrun_same_block` | ✅ **Primary root cause.** Competitor tx at lower index, same pool, consumed liquidity. The headline demo. |
| B9 | `unknown` | ✅ **Humility signal.** Investigated and ruled out B1. First-class answer. |
| B2 | `frontrun_prior_block` | 🔮 Future |
| B3 | `sandwiched` | 🔮 Future — if ahead of schedule on day 4, add this |
| B4 | `stale_quote` | 🔮 Future |
| B5 | `gas_underpriced_landed_late` | 🔮 Future |
| B6 | `gas_underpriced_missed_block` | 🔮 Future |
| B7 | `slippage_set_too_tight` | 🔮 Future |
| B8 | `route_suboptimal` | 🔮 Future — complex math, low demo payoff |

### Why `unknown` (B9) matters
Every other agent at the hackathon will be overconfident. Yours admitting uncertainty — with receipts showing what it ruled out — is a memorable trust signal. Build one `unknown` demo case and close the pitch with it.

### The 2 demo paths

```
Path 1: A2 → B1  (frontrun_same_block)
  "Why did I earn $50 when I expected $75?"
  → simulate at N-1 → reconstruct expected PnL
  → scan block txs → find competitor at lower index
  → trace competitor → confirm same pool touched
  → "This tx took your $25."

Path 2: A2 → B9  (unknown)
  "Why did this trade underperform?"
  → simulate at N-1 → reconstruct expected PnL
  → scan block txs → no competitor found on same pool
  → check pool state delta → within normal variance
  → "I investigated and cannot confidently classify this. Here's what I ruled out."
```

---

## 8. Architecture

```
           ┌─────────────────────────────┐
           │    Web Dashboard (Next.js)  │
           │  - trade list               │
           │  - chat interface           │
           │  - evidence panel           │
           │  - tool-call timeline       │
           └──────────────┬──────────────┘
                          │ HTTP / SSE
                          ▼
           ┌─────────────────────────────┐
           │    Axum API Server          │
           │  - /trades                  │
           │  - /investigate (streaming) │
           │  - /reports                 │
           └──────────────┬──────────────┘
                          │
                          ▼
           ┌─────────────────────────────┐
           │   observability_agent       │
           │  - Claude tool-use loop     │
           │  - system prompt            │
           │  - tool budget enforcement  │
           │  - evidence citation check  │
           └──┬───────────────────────┬──┘
              │                       │
              ▼                       ▼
   ┌──────────────────┐   ┌──────────────────────┐
   │  tenderly_client │   │     rpc_client       │
   │  - simulate      │   │  - get_block         │
   │  - trace         │   │  - get_tx_receipt    │
   │  - state override│   │  - get_logs          │
   └──────────────────┘   └──────────────────────┘
              │                       │
              ▼                       ▼
       Tenderly API            Base RPC (Alchemy)
                          │
                          ▼
           ┌─────────────────────────────┐
           │      QuestDB (reuse)        │
           │  - trade_reports            │
           │  - investigation_logs       │
           └─────────────────────────────┘
```

**Crates:**

- `shared_types` — `TradeRef`, `TradeReport`, `FailureCategory`, `Evidence`, `ToolCall` records
- `tenderly_client` — REST wrapper. Simulation, traces, state overrides.
- `rpc_client` — thin wrapper around `alloy` for block/tx/receipt/logs calls
- `pool_math` — Uniswap v2/v3 math for expected-PnL calculation. Use existing crate (`uniswap_v3_math`), don't roll your own
- `observability_agent` — the Claude loop, tool definitions, system prompt, tool-call budget enforcement, citation verification
- `db_quest` — reuse from polymarket-bot
- `api` — axum server, SSE streaming for live tool-call timelines

---

## 9. Tech stack

**Backend:**
- **Rust** — workspace layout mirroring polymarket-bot
- **alloy** — Ethereum RPC client library
- **Tenderly REST API** — simulations, traces, state overrides
- **Alchemy** — Base RPC node
- **axum** — HTTP + SSE for streaming agent responses
- **tokio** — async runtime
- **Claude API** (`claude-sonnet-4-6`) — the reasoning layer; tool use is native
- **QuestDB** — reuse existing setup for report storage
- **tracing** — reuse `logging` crate from polymarket-bot

**Frontend:**
- **Next.js + TailwindCSS**
- **Server-Sent Events** — live tool-call streaming
- Components: trade list + chat + evidence panel + tool-call timeline

**External:**
- **EigenPhi** — source real MEV wallet addresses for the demo dataset
- **Tenderly** — sponsor; lean into it visibly in the demo

---

## 10. Tool layer

5 tools. Each one backs a specific evidence need from the 2 MVP paths.

| Tool | Signature (sketch) | Backed by | Serves |
|---|---|---|---|
| `get_trade` | `(tx_hash) → {from, to, block, index, value, gas, status, calldata, logs, realized_pnl}` | RPC + decoder | Both paths |
| `simulate_at_state` | `(tx_hash, block, state_override?) → {simulated_pnl, success, revert_reason}` | Tenderly | Both paths — **most important tool** |
| `get_block_txs` | `(block_number) → [{index, tx_hash, from, to, touched_pools}]` | RPC | B1, B9 |
| `get_tx_trace` | `(tx_hash) → call_tree` | Tenderly | B1 confirmation |
| `get_pool_state` | `(pool_address, block) → {reserves \| ticks, sqrt_price, liquidity}` | RPC + pool_math | B9 (ruling out stale quote) |

**Dropped for MVP:** `get_gas_distribution`, `get_address_history`

**Tool budget:** max 8 tool calls per investigation turn. If the agent hasn't resolved the case by then, it must report `unknown` with what it tried.

**Rate limits:** Cache every Tenderly simulation result keyed by `(tx_hash, block, state_override_hash)`. Pre-warm all demo trades before the live demo.

---

## 11. Agent design

### System prompt (outline)
```
You are a forensics agent for MEV searchers. Your job is to investigate
a single historical trade and classify its outcome and root cause using
this fixed taxonomy.

MVP taxonomy:
- Outcomes: A2 (success_underperformed), A9 (unknown)
- Root causes: B1 (frontrun_same_block), B9 (unknown)

Investigation protocol:
1. Call get_trade to load the tx.
2. Call simulate_at_state at block N-1 to establish expected PnL.
3. Compare expected vs realized. If within 5%: report A2 with no clear
   root cause and stop.
4. Call get_block_txs to find all txs in the same block touching the
   same pool, at a lower index than the target tx.
5. If a candidate is found: call get_tx_trace on it to confirm it
   touched the same pool. If confirmed: report A2 → B1 with evidence.
6. If no candidate found: call get_pool_state at N-1 and N to measure
   pool movement. If movement is within normal variance: report A2 → B9
   with a list of what you ruled out.
7. If still unresolved after 8 tool calls total: report A2 → B9.

Evidence rules (NON-NEGOTIABLE):
- Every claim must cite a specific tool result from this conversation.
- If a tool returned nothing or failed, say so. Do not guess.
- Do not use general knowledge about MEV to fill gaps.
- If the user asks a follow-up, reuse prior tool results — do not re-fetch.
```

### Loop behavior
- **Max tool calls per turn:** 8 (hard stop)
- **Citation verification:** post-process the output and strip or flag any uncited claims
- **Multi-turn:** conversation history is preserved; follow-ups reason over cached results
- **Streaming:** tool calls stream via SSE to the frontend in real time

### Response schema
```rust
struct TradeReport {
    tx_hash: String,
    outcome: OutcomeCategory,      // A2 or A9
    root_cause: Option<RootCause>, // B1 or B9
    expected_pnl: Option<Decimal>,
    realized_pnl: Decimal,
    pnl_delta: Option<Decimal>,
    confidence: f32,               // 0.0..1.0
    evidence: Vec<Evidence>,       // structured citations
    counterfactuals: Vec<Counterfactual>,
    narrative: String,             // the LLM's explanation
    tool_calls: Vec<ToolCall>,     // audit trail
}
```

---

## 12. Data & demo strategy

### Dataset
- **2 real MEV wallets** sourced from EigenPhi leaderboards
- Hand-curate **2 demo trades** — one per path. Verify both by hand before the hackathon starts.
- Pre-warm Tenderly cache for both trades. The live demo must never hit a cold API call.

### Demo set

| # | Trade | Path | Demo purpose |
|---|---|---|---|
| 1 | Arb that underperformed | A2 → B1 | **Headline demo: "$50 vs $75"** |
| 2 | Ambiguous loss | A2 → B9 | **Humility signal** — rules things out, admits limits |

### Demo narrative
1. Open the dashboard. Show the trade list.
2. Click trade #1. User types: *"Why did this earn $50 when I expected $75?"*
3. Watch tool-call timeline stream live: `get_trade → simulate_at_state(N-1) → get_block_txs → get_tx_trace(competitor)`
4. Agent produces cited report in ~20 seconds: *"This tx at index 14 touched the same pool before you and consumed the liquidity."*
5. User asks follow-up: *"Who was that?"* Agent reasons from existing context — no re-fetch.
6. Switch to trade #2. Agent works through the same steps, rules out B1, reports B9. *"I investigated and cannot confidently classify this. Here's what I ruled out."*
7. **That's the close.** The unknown case is your most memorable moment.

---

## 13. Execution plan (5-day async)

| Day | Focus | Done when |
|---|---|---|
| **1** | `get_trade` + `simulate_at_state` working end-to-end on one real Base tx. Verify simulated PnL against Etherscan manually. | You can print expected vs realized PnL for one real trade. |
| **2** | `get_block_txs` + `get_tx_trace` + agent loop running the B1 path (Path 1) end-to-end. Curate both demo trades. | Agent correctly identifies the frontrunning tx on demo trade #1. |
| **3** | B9 path (Path 2) + `get_pool_state` + citation verification layer. Pre-warm Tenderly cache. Record backup video. | Both paths working. Backup video recorded. |
| **4** | Frontend integration + SSE streaming tool-call timeline + evidence panel. Polish chat UX. | Real data flowing into UI. Demo runnable end-to-end in browser. |
| **5** | Demo rehearsal (10+ runs). Pitch deck. README. Submission polish. **No new features.** | Submitted. |

**The rule for days 1–3:** don't touch the frontend. Don't add new tools. Don't read about B3. The only question is: do both paths work reliably on your two curated trades?

---

## 14. Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Tenderly API flaky during demo | Medium | High | Cache everything + recorded backup video |
| Uniswap v3 tick math eats a day | Medium | High | Use `uniswap_v3_math` crate; verify day 1 |
| Agent loops / over-investigates | Medium | High | Hard 8-call budget, strict system prompt |
| Agent hallucinates uncited claims | Medium | High | Post-process citation check; strip uncited lines |
| `simulate_at_state` doesn't match Etherscan | Medium | High | Verify both demo trades manually on day 1; if broken, fix before building anything else |
| Judges don't know MEV | Low-Med | Medium | Open pitch with pain, not tech; show the 20-min manual workflow |
| Scope creep | High | Medium | 2 paths, 5 tools. Refer to this list daily. |

---

## 15. Pitch narrative (~5 min)

**Slide 1 — The pain (~45s)**
A DEX searcher at 2am staring at four browser tabs: Etherscan, Tenderly, a block explorer, a spreadsheet. They just lost $25 on a trade and have no idea why. This is their reality.

**Slide 2 — The manual process (~45s)**
Walk through the 8-step manual workflow. 20 minutes per trade. Most post-mortems never happen. Money leaks invisibly.

**Slide 3 — The gap (~45s)**
Show the competitive landscape matrix. EigenPhi, Tenderly, libMEV, Etherscan. All dashboards. All tell you *what*. None tell you *why*. EigenPhi's own docs confirm this.

**Slide 4 — Live demo: the "$50 vs $75" trade (~90s)**
Paste the tx hash, ask the question, watch the tool-call timeline stream live, receive the cited report. 20 seconds vs 20 minutes.

**Slide 5 — Live demo: the `unknown` (~60s)**
Second trade. Agent works through the same steps, rules out the frontrun hypothesis, and says: *"I don't know — but here's what I ruled out."* This is the slide judges remember. No other agent at this hackathon will do this.

**Slide 6 — What's next (~30s)**
Production: live stream of all new trades, sandwich detection, gas counterfactuals, aggregate insights across sessions. Tonight's build is the forensic core.

**Slide 7 — Thanks / team / Q&A**

---

## 16. For the future

Everything below is real and valuable — deliberately cut from the hackathon MVP to keep the demo bulletproof. Revisit post-submission.

**Additional outcome categories:**
- A1 `success_as_expected` — short-circuit case, low investigation value
- A3 `reverted_trivial` — short-circuit on revert string
- A4 `reverted_nontrivial` — requires deep call-frame trace parsing
- A5 `not_landed` — requires mempool / builder data

**Additional root causes:**
- B2 `frontrun_prior_block` — tx in N-1 moved pool before you landed
- B3 `sandwiched` — two txs from same actor bracket yours (visceral demo moment; add first after MVP)
- B4 `stale_quote` — pool moved between detection and submission, no adversarial cause
- B5 `gas_underpriced_landed_late` — right block, wrong index
- B6 `gas_underpriced_missed_block` — didn't land in target block
- B7 `slippage_set_too_tight` — reverted at slippage check
- B8 `route_suboptimal` — hard math, low demo payoff

**Additional tools:**
- `get_gas_distribution` — needed for B5/B6 gas counterfactuals
- `get_address_history` — needed for B3 sandwich-bot identification

**Product features:**
- Live stream of all new trades from watched wallets
- Alerts on pattern shifts across sessions
- Aggregate cross-trade dashboards
- Multi-chain support (mainnet, Arbitrum, Optimism)
- Curve, Balancer, other AMM support beyond Uni v2/v3
- Gas counterfactual: "if you'd bid +3 gwei, you'd have landed at index 4"