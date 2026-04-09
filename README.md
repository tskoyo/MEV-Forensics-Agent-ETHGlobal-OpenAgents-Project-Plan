# MEV Forensics Agent — ETHGlobal OpenAgents Project Plan

*Version 1.0 · Drafted 2026-04-10*

## Table of contents
1. [The problem](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan/blob/master/README.md#1-the-problem)
2. [What is MEV (primer)](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#2-what-is-mev-brief-primer)
3. [Why we're building this](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#3-why-were-building-this)
4. [Existing solutions and where they stop](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#4-existing-solutions-and-where-they-stop)
5. [How we go one step further](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#5-how-we-go-one-step-further)
6. [Scope](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#6-scope)
7. [Failure taxonomy](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#7-failure-taxonomy)
8. [Architecture](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#7-failure-taxonomy)
9. [Tech stack](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#9-tech-stack)
10. [Tool layer](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#10-tool-layer)
11. [Agent design](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#10-tool-layer)
12. [Data & demo strategy](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#10-tool-layer)
13. [Execution plan (hackathon week)](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#10-tool-layer)
14. [Risks and mitigations](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#10-tool-layer)
15. [Pitch narrative](https://github.com/tskoyo/MEV-Forensics-Agent-ETHGlobal-OpenAgents-Project-Plan?tab=readme-ov-file#10-tool-layer)
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

## 3. Why we're building this

1. **The pain is real and underserved.** Solo MEV searchers are a small but technically sophisticated user base. They manually investigate trades today because no tool does it for them. Classic agentic automation shape: bounded domain, clear tools, verifiable outputs.
2. **LLM tool-use only recently became reliable enough.** Chaining 8–10 tool calls with Tenderly, RPC lookups, and simulations used to be a pipe dream. Claude handles it today. New product shape enabled by recent capability.
3. **ETHGlobal OpenAgents is literally about agents.** Matches the theme: autonomous tool use, multi-step reasoning, verifiable results, DeFi domain, obvious Tenderly sponsor fit.

## 4. Existing solutions and where they stop

| Tool | What it does | What it doesn't do |
|---|---|---|
| **EigenPhi** | Classifies MEV (arb, sandwich, liquidation), profit per searcher, victim labeling, real-time stream | No counterfactuals, no expected-vs-realized, no per-tx frontrun attribution, no NL explanation, no chat |
| **libMEV** | Searcher leaderboards (ELO), bundle classification, aggregate stats | Same gaps; explicitly framed as "detective work for you to do" |
| **Tenderly** | Simulation, call traces, debugger, alerts | Powerful *evidence layer*, but a tool — it doesn't reason, you drive it |
| **Etherscan / Phalcon** | Raw tx + decoded traces | Manual; no reasoning |
| **MEV-Inspect (Flashbots)** | Open-source MEV classification library | Batch classifier, not a product |
| **Nansen AI / Arkham** | Wallet labeling + some LLM search over aggregate data | Surface-level; no trace-level reasoning, no simulation, no MEV forensics depth |

EigenPhi's own docs describe their arbitrage profile as *"retrospective summary of what happened, not analytical depth about why outcomes diverged from alternatives."* That sentence is the gap.

## 5. How we go one step further

We put a disciplined, evidence-citing agent on top of the existing data layer.

1. **Per-trade investigation**, not aggregate stats. Point at a tx, ask "why?"
2. **Expected vs realized PnL reconstruction.** The agent simulates the trade against pool state at block N-k (k=1,2,3) to compute what it *should* have earned, then explains the gap.
3. **Counterfactuals.** "If your priority fee had been +3 gwei, you'd have landed at index 4 and captured the full $75."
4. **Evidence discipline.** Every claim backed by a specific tool result. No general-knowledge reasoning. If unconfirmed, agent says `unknown` — and that's a feature.
5. **Conversational.** Multi-turn follow-ups reuse prior tool results.
6. **Live tool-call timeline.** The UI streams agent actions in real time as trust theater.

**Positioning in one sentence:**
> EigenPhi tells you what class of MEV a tx was. Tenderly gives you the trace. libMEV ranks who the top searchers are. None of them tell you **why your specific trade underperformed** or **what would have won it**. We're the layer on top — an autonomous investigator that uses those same data sources plus simulation to produce evidence-cited forensic reports, conversationally.

## 6. Scope

### In scope (v1)
- Single chain: **Base**
- DEX coverage: **Uniswap v2 + v3** only
- Read-only analysis of historical trades
- 5–10 curated demo trades from real MEV wallets
- Multi-turn conversational interface
- Web dashboard (frontend engineer on team owns it)
- ~7 investigation tools backed by Tenderly + RPC
- Failure taxonomy (5 outcomes + 8 root causes + `unknown`)
- 2–3 reliable counterfactuals (gas priority, slippage, stale quote)

### Out of scope (v1)
- Live trade execution, wallet/key handling
- Multi-chain support
- Curve, Balancer, other AMMs beyond Uni v2/v3
- Route optimization (B8)
- Automated remediation ("auto-tune your bot")
- Real-time alerting / push notifications
- Aggregate cross-trade dashboards (nice-to-have, cut if time-pressured)
- Auth, multi-user, any infra beyond single-machine demo

### Explicit "won't build" list (anti-scope-creep)
- Our own MEV bot
- Smart contracts
- Mempool monitoring
- Private order flow integration
- A backtester
- Any "predict future trades" feature

## 7. Failure taxonomy

The spine of the project. Every trade gets classified into one outcome + (optionally) one root cause. The taxonomy bounds the agent's reasoning and designs the tool layer.

### Outcome categories (A)
- **A1 `success_as_expected`** — realized PnL within X% of expected. Short-circuit, don't investigate.
- **A2 `success_underperformed`** — main case. Tx succeeded but realized < expected. Triggers root-cause search.
- **A3 `reverted_trivial`** — tx reverted with a clear self-explanatory revert string. Short-circuit — the string *is* the answer.
- **A4 `reverted_nontrivial`** — custom error, opaque revert, or panic. Investigate which call frame reverted and why.
- **A5 `not_landed`** — tx submitted but never included. Investigate gas market.

### Root cause categories (B)
- **B1 `frontrun_same_block`** — competitor tx at lower index, same pool, consumed your liquidity
- **B2 `frontrun_prior_block`** — tx in N-1 moved pool against you before you landed
- **B3 `sandwiched`** — two txs from same actor bracket yours
- **B4 `stale_quote`** — pool moved between detection and submission; no adversarial cause
- **B5 `gas_underpriced_landed_late`** — landed in right block but at a higher index than optimal
- **B6 `gas_underpriced_missed_block`** — didn't land in the target block at all
- **B7 `slippage_set_too_tight`** — reverted at slippage check; volatility was within normal range
- **B8 `route_suboptimal`** — **SKIP for v1.** Hard math, low demo payoff.
- **B9 `unknown`** — investigated, ruled out the above, cannot confidently classify. First-class answer. Trust signal.

### Why `unknown` matters
Every other agent at the hackathon will be over-confident. Yours admitting uncertainty — with receipts showing what it ruled out — is a memorable trust signal. Build at least one `unknown` case into the demo.

## 8. Architecture

Mirrors the polymarket-bot workspace pattern.

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

- `shared_types` — `TradeRef`, `TradeReport`, `FailureCategory`, `Evidence`, `ToolCall` records. Mirrors polymarket-bot's `shared_types` crate.
- `tenderly_client` — REST wrapper. Simulation, traces, state overrides.
- `rpc_client` — thin wrapper around `alloy` (or `ethers-rs`) for block/tx/receipt/logs calls.
- `pool_math` — Uniswap v2/v3 math for expected-PnL calculation. Prefer existing crate (e.g., `uniswap_v3_math`) over rolling your own.
- `observability_agent` — the Claude loop, tool definitions, system prompt, tool-call budget enforcement, citation verification.
- `db_quest` — reuse from polymarket-bot. Store reports and investigation logs.
- `api` — axum server, SSE streaming for live tool-call timelines.

## 9. Tech stack

**Backend:**
- **Rust** — workspace layout mirroring polymarket-bot
- **alloy** (or ethers-rs) — Ethereum RPC client library, actively maintained
- **Tenderly REST API** — simulations, traces, state overrides
- **Alchemy or Infura** — Base RPC node
- **axum** — HTTP + SSE for streaming agent responses
- **tokio** — async runtime
- **Claude API** (`claude-opus-4-6`) — the reasoning layer; tool use is native
- **QuestDB** — reuse existing setup for report storage
- **tracing** — reuse `logging` crate from polymarket-bot

**Frontend (teammate):**
- **Next.js + TailwindCSS** — standard hackathon stack
- **Server-Sent Events** — live tool-call streaming
- Simple component layout: trade list + chat + evidence panel + tool-call timeline

**External:**
- **EigenPhi** — to source real MEV wallet addresses for the demo dataset
- **Libmev / Dune** — cross-reference searcher behavior
- **Etherscan / Basescan** — manual verification during learning phase

**Why Rust:** existing Rust muscle memory from polymarket-bot, crate layout mirrors perfectly, faster than Python or TypeScript given existing patterns. LLM orchestration doesn't need Python — `anthropic-rs` or direct HTTP works fine.

## 10. Tool layer

~7 tools. Each backs one or more evidence needs from the taxonomy.

| Tool | Signature (sketch) | Backed by | Serves |
|---|---|---|---|
| `get_trade` | `(tx_hash) → {from, to, block, index, value, gas, status, calldata, logs, realized_pnl}` | RPC + decoder | Every investigation |
| `get_block_txs` | `(block_number) → [{index, tx_hash, from, to, touched_pools}]` | RPC | B1, B2, B3 |
| `get_tx_trace` | `(tx_hash) → call_tree` | Tenderly | A4, B1, B3 |
| `simulate_at_state` | `(tx_hash, block, state_override?) → {simulated_pnl, success, revert_reason}` | Tenderly | A2, B2, B4, B7 — all counterfactuals |
| `get_pool_state` | `(pool_address, block) → {reserves \| ticks, sqrt_price, liquidity}` | RPC + pool_math | B2, B4 |
| `get_gas_distribution` | `(block_number) → {base_fee, priority_fees_by_index[]}` | RPC | B5, B6 |
| `get_address_history` | `(address, window) → {tx_count, known_label, pattern}` | RPC + curated labels | B3 (sandwich-bot identification) |

**Tool budget:** max 10 tool calls per investigation turn. If the agent hasn't resolved the case by then, it must report `unknown` with what it tried.

**Rate limits:** Tenderly free tier has caps. For the demo, cache every simulation result keyed by `(tx_hash, block, state_override_hash)`. Curated demo trades should be pre-warmed on disk so the live demo never hits a cold API call.

## 11. Agent design

### System prompt (outline)
```
You are a forensics agent for MEV searchers. Your job is to investigate
a single historical trade and classify its outcome and root cause using
this fixed taxonomy: [A1-A5, B1-B9].

Investigation protocol:
1. Call get_trade to load the tx.
2. Determine outcome category (A1-A5).
3. If A1 or A3: report immediately. Do not investigate further.
4. If A2/A4/A5: call simulate_at_state at block N-1 to establish
   expected PnL. Compare to realized.
5. Form a hypothesis (B1-B8). Call tools to confirm or refute.
6. If confirmed: report with evidence. If refuted: try the next
   hypothesis.
7. If no hypothesis fits after 10 tool calls total: report B9 (unknown)
   with a list of what you ruled out.

Evidence rules (NON-NEGOTIABLE):
- Every claim you make must cite a specific tool result from this
  conversation.
- If a tool returned nothing or failed, say so. Do not guess.
- Do not use general knowledge about MEV to fill in gaps. Only reason
  over tool outputs.
- Prefer short, evidence-dense reports over long narratives.
- If the user asks a follow-up, reuse prior tool results in context —
  do not re-fetch.
```

### Loop behavior
- **Max tool calls per turn:** 10 (hard stop)
- **Citation verification:** every sentence in the final report must reference at least one prior tool call. Post-process the output and strip or flag any uncited claims.
- **Multi-turn:** conversation history is preserved; follow-ups reason over cached results.
- **Streaming:** tool calls stream via SSE to the frontend in real time for the trust theater.

### Response schema
```rust
struct TradeReport {
    tx_hash: String,
    outcome: OutcomeCategory,      // A1..A5
    root_cause: Option<RootCause>, // B1..B9
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

## 12. Data & demo strategy

### Dataset
- **2–3 real MEV wallets** sourced from EigenPhi/Libmev leaderboards. Include at least one known sandwich bot (jaredfromsubway.eth on mainnet, or its Base equivalent if available).
- Pull their last ~200 txs.
- Hand-curate **5–10 demo trades** covering each outcome and root-cause category.
- Pre-warm Tenderly cache for all curated trades.

### Demo set (5 trades, one per category)
| # | Trade | Category | Demo purpose |
|---|---|---|---|
| 1 | Arb that underperformed | A2 → B1 (frontrun_same_block) | **Headline demo: "$50 vs $75"** |
| 2 | Sandwich victim | A2 → B3 (sandwiched) | **"jaredfromsubway got you"** — visceral, judge recognition |
| 3 | Late tx | A2 → B5 (gas_underpriced_landed_late) | Gas counterfactual showcase |
| 4 | Non-trivial revert | A4 → B7 (slippage too tight) | Agent reads traces, not just receipts |
| 5 | Ambiguous loss | A2 → B9 (unknown) | **Humility signal** — rules things out, admits limits |

### Demo narrative
1. Open the dashboard. Show the trade list.
2. Click trade #1. User types: "Why did this earn $50 when I expected $75?"
3. Watch tool-call timeline stream live: `get_trade → simulate_at_state(N-1) → get_block_txs → get_tx_trace(competitor) → simulate_at_state(after_competitor)`
4. Agent produces report in ~20 seconds with clickable citations.
5. User asks follow-up: "What if I'd used +3 gwei priority?"
6. Agent reasons from existing context (no re-fetch), produces counterfactual.
7. Switch to trade #2 for the sandwich — big visual moment.
8. Close with trade #5 (`unknown`) to demonstrate trust discipline.

## 13. Execution plan (hackathon week)

### Pre-hackathon (this week)

**Learning track — parallel with scaffolding**
- Day 1: Read 1 real arb + 1 sandwich + 1 revert on Etherscan + Tenderly.
- Day 2: Read 3 more. Focus on Base + Uniswap v3.
- Day 3: Predict-then-verify on 3 unseen txs. Validate intuition.

**Scaffolding track — parallel**
- Day 1: Tenderly API account, first simulation call working, repo skeleton
- Day 2: `get_trade` + `simulate_at_state` tools with real responses verified against Etherscan
- Day 3: `get_block_txs` + `get_tx_trace` + first Claude tool-use loop
- Day 4: Pool math via existing crate, expected-PnL reconstruction working
- Day 5: Curate demo dataset (5 trades chosen and verified by hand)

**Team coordination — by end of this week**
- Lock data contract with frontend engineer (`TradeReport` JSON schema)
- Agree on SSE streaming format for tool calls
- Frontend engineer starts building UI against stub data from day 1 of hackathon

### Hackathon (assume 36–48 hours)

**Hour 0–6: De-risk the core**
- All 7 tools responding correctly on 2 canonical trades
- Tenderly rate limits understood, caching in place
- Backup video recorded on day 1 while core is fresh

**Hour 6–18: Agent loop + polish**
- System prompt iteration
- Citation verification layer
- Counterfactual logic (gas, slippage, stale-quote)
- End-to-end on all 5 demo trades

**Hour 18–30: Frontend integration**
- Real data flowing into UI
- Tool-call timeline streaming
- Evidence panel with clickable citations
- Polish the chat experience

**Hour 30–40: Rehearsal + fallbacks**
- Demo rehearsal (aim for 10+ full run-throughs)
- Backup video refreshed
- Curated trades pre-warmed in cache
- Test on a different network to rule out wifi flakiness

**Hour 40–end: Submission polish**
- Pitch deck
- Project page
- README
- Final rehearsal

## 14. Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Tenderly API flaky during demo | Medium | High | Cache everything + recorded backup video |
| Uniswap v3 tick math eats a day | Medium | High | Use `uniswap_v3_math` crate; verify day 1 |
| Agent loops / over-investigates | Medium | High | Hard 10-call budget, strict system prompt |
| Agent hallucinates uncited claims | Medium | High | Post-process citation check; strip uncited lines |
| Judges don't know MEV | Low-Med | Medium | Open pitch with pain, not tech; show the 20-min manual workflow |
| Team coordination with FE dev | Medium | Medium | Lock data contract before hackathon starts |
| Scope creep | High | Medium | Explicit "won't build" list; refer to it daily |
| EVM fluency | Medium | Medium | 3-day parallel learning plan |
| Simulation against historical state fails | Medium | High | Pick demo trades where simulation is known to work; verify day 1 |
| Base RPC provider rate limits | Low | Medium | Alchemy paid tier for the weekend; cache aggressively |

## 15. Pitch narrative (~5 min)

**Slide 1 — The pain (~45s)**
A DEX searcher at 2am staring at four browser tabs: Etherscan, Tenderly, a block explorer, a spreadsheet. They just lost $25 on a trade and have no idea why. This is their reality.

**Slide 2 — The manual process (~45s)**
Walk through the 8-step manual workflow. 20 minutes per trade. Most post-mortems never happen. Money leaks invisibly.

**Slide 3 — The gap (~45s)**
Show the competitive landscape matrix. EigenPhi, Tenderly, libMEV, Etherscan. All dashboards. All tell you *what*. None tell you *why*. EigenPhi's own docs confirm this.

**Slide 4 — Live demo: the "$50 vs $75" trade (~90s)**
Paste the tx hash, ask the question, watch the tool-call timeline stream live, receive the cited report. Click a citation to jump to the Tenderly trace. 20 seconds vs 20 minutes.

**Slide 5 — Live demo: the sandwich (~45s)**
Second trade. Agent identifies jaredfromsubway. Visceral moment.

**Slide 6 — Live demo: the `unknown` (~30s)**
Third trade. Agent says "I don't know — but here's what I ruled out." Trust signal. This is the slide judges remember.

**Slide 7 — What's next (~30s)**
Production: live stream of all new trades, alerts on pattern shifts, aggregate insights across sessions. Tonight's build is the forensic core.

**Slide 8 — Thanks / team / Q&A**

## 16. Open questions

These are the things not yet fully resolved — close out before or during the early hackathon hours:

1. **Which chain, really?** Base is locked in conceptually, but verify Tenderly's simulation coverage on Base is as good as mainnet. If gaps exist, fall back to mainnet.
2. **Which MEV wallets specifically?** Need to pick 2–3 by name, with URLs, and verify each has a good mix of winner/loser trades. Target this during the learning days.
3. **Does the agent compute expected PnL at a single k or a range?** Leaning toward range (N-1, N-2, N-3) because it's more honest and gives the agent more to say. Confirm day 1 of build.
4. **Citation verification — post-process or inline?** Post-process is simpler. Inline (agent self-checks before emitting) is more elegant but riskier. Start with post-process.
5. **Is there a "staged" live sandwich moment in the demo**, or are all 5 trades historical? All historical is safer. Staged is more memorable. Decide based on time.
6. **Does the frontend engineer need a mock data API on day 1?** Yes — prepare a static JSON fixture of a canned `TradeReport` so they can build the UI in parallel without waiting on backend.
