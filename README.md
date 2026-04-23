# MEV Forensics Agent — ETHGlobal OpenAgents Project Plan

*Version 3.1 · MVP scope · Updated 2026-04-23*


## Table of contents
1. [The problem](#1-the-problem)
2. [What is MEV (primer)](#2-what-is-mev-brief-primer)
3. [Why we're building this](#3-why-were-building-this)
4. [Existing solutions and where they stop](#4-existing-solutions-and-where-they-stop)
5. [How we go one step further](#5-how-we-go-one-step-further)
6. [Scope](#6-scope)
7. [Failure taxonomy](#7-failure-taxonomy)
8. [Architecture](#8-architecture)
9. [Tech stack](#9-tech-stack)
10. [Tool layer](#10-tool-layer)
11. [Agent design](#11-agent-design)
12. [Data & demo strategy](#12-data--demo-strategy)
13. [Execution plan (12-day)](#13-execution-plan-12-day)
14. [Risks and mitigations](#14-risks-and-mitigations)
15. [Pitch narrative](#15-pitch-narrative-5-min)
16. [Open questions](#16-open-questions)

---

## 1. The problem

DEX trading bots — especially MEV searchers running arbitrage, liquidation, or sandwich strategies — operate in a brutally adversarial, high-frequency environment. Every day these bots submit thousands of transactions. Some win. Many lose money or underperform expectations. And when they do, **the operator has no fast way to find out why.**

Today, when a searcher sees a trade that earned $50 instead of the expected $75, their workflow is:

1. Open Etherscan to the tx
2. Copy the block number, open it in another tab for neighboring txs
3. Manually scan each competing tx in the same block looking for anyone who touched the same pool
4. Manually compute what pool state *would* have given them the expected outcome
5. Try to re-simulate the trade against different historical states
6. Write themselves a note and move on

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

> **Note on Base vs Ethereum mainnet:** Base uses a centralized sequencer (Coinbase) with FCFS ordering. Traditional mempool-based frontrunning (bot observes your pending tx and outbids on gas) does not exist on Base. What *does* happen: two bots independently detect the same opportunity and both land in the same block, with one at a lower index by sequencer timing. The effect on your realized PnL is identical — pool state is consumed before you — but the mechanism is sequencer-ordering preemption, not adversarial gas-war frontrunning. The demo uses Ethereum mainnet where the classic frontrun story is accurate and familiar to judges.

---

## 3. Why we're building this

1. **The pain is real and underserved.** Solo MEV searchers are a small but technically sophisticated user base. They manually investigate trades today because no tool does it for them. Classic agentic automation shape: bounded domain, clear tools, verifiable outputs.
2. **LLM tool-use only recently became reliable enough.** Chaining 5–8 tool calls with Tenderly, RPC lookups, and simulations used to be a pipe dream. Claude handles it today. New product shape enabled by recent capability.
3. **ETHGlobal OpenAgents is literally about agents.** Matches the theme: autonomous tool use, multi-step reasoning, verifiable results, DeFi domain, obvious Uniswap Foundation sponsor fit.

---

## 4. Existing solutions and where they stop

| Tool | What it does | What it doesn't do |
|---|---|---|
| **EigenPhi** | Classifies MEV (arb, sandwich, liquidation), profit per searcher, victim labeling, real-time stream | No counterfactuals, no expected-vs-realized, no per-tx frontrun attribution, no NL explanation, no chat |
| **libMEV** | Searcher leaderboards (ELO), bundle classification, aggregate stats | Same gaps; explicitly framed as "detective work for you to do" |
| **Tenderly** | Simulation, call traces, debugger, alerts | Powerful *evidence layer*, but a tool — it doesn't reason, you drive it |
| **Etherscan / Phalcon** | Raw tx + decoded traces | Manual; no reasoning |
| **Nansen AI / Arkham** | Wallet labeling + some LLM search over aggregate data | Surface-level; no trace-level reasoning, no simulation, no MEV forensics depth |

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
- Single chain: **Ethereum mainnet**
- DEX coverage: **Uniswap v2 + v3** only
- Read-only analysis of historical trades
- **2 curated demo trades** from real MEV wallets
- Multi-turn conversational interface
- Web dashboard (Next.js)
- **5 investigation tools** backed by Tenderly + RPC
- **2 investigation paths** from the failure taxonomy (see section 7)
- Streaming tool-call timeline in the UI (SSE)

### Out of scope (hackathon)
- Live trade execution, wallet/key handling
- Multi-chain support
- Curve, Balancer, or any AMM beyond Uni v2/v3
- Automated remediation
- Real-time alerting / push notifications
- Aggregate cross-trade dashboards
- Auth, multi-user, any infra beyond single-machine demo

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

### The 2 MVP investigation flows

Both flows share the same opening steps. The agent always starts by loading the trade and simulating expected PnL — the first decision gate determines how far it goes.

---

#### Flow 1 — `A2 → B1` (frontrun confirmed)

```
1. get_trade(tx_hash)
   → load tx, logs, realized PnL

2. simulate_at_state(tx_hash, block N-1)
   → compute expected PnL at pre-block state

3. PnL gap > 5%?
   → NO  → early exit: report A2 → B9 "normal variance, nothing to investigate" (see Flow 2a)
   → YES → continue

4. get_block_txs(block_number)
   → find all txs in the same block that touched the same pool
   → filter to txs at a lower block index than the target tx

5. Competitor found at lower index?
   → NO  → continue to Flow 2b
   → YES → continue

6. get_tx_trace(competitor_tx_hash)
   → confirm competitor touched the same pool

7. RESULT: A2 → B1
   "Tx at index N touched pool X before you and consumed the liquidity.
    Expected: $75. Realized: $50. Delta: -$25."
```

**What the agent says:** *"This tx at index 14 got there first and took your $25."*

---

#### Flow 2 — `A2 → B9` (unknown — two variants)

**Variant 2a — Early exit (gap within normal variance)**

```
1. get_trade(tx_hash)
2. simulate_at_state(tx_hash, block N-1)

3. PnL gap > 5%?
   → NO → early exit

4. RESULT: A2 → B9
   "Realized PnL is within 5% of expected. No meaningful underperformance
    to investigate. This is normal variance."
```

No further tool calls. Short-circuit and stop.

---

**Variant 2b — Full investigation, no cause found**

```
1. get_trade(tx_hash)
2. simulate_at_state(tx_hash, block N-1)

3. PnL gap > 5%?  → YES → continue

4. get_block_txs(block_number)
   → scan for same-pool txs at lower index

5. Competitor found?  → NO → continue

6. get_pool_state(pool_address, block N-1)
   get_pool_state(pool_address, block N)
   → compare price/reserves between N-1 and N
   → was there significant organic market movement?

7. RESULT: A2 → B9
   Two possible narratives depending on pool state delta:
   - Large delta: "No frontrunner found, but the pool price moved X%
     between block N-1 and N from unrelated trades. Your quote was stale."
   - Small delta: "I investigated and cannot confidently classify this.
     Here is what I ruled out: [B1 checked and not found, pool state
     within normal variance]."
```

**Why `get_pool_state` matters here:** without it, the agent can only say "no competitor found." With it, the agent can distinguish between *stale quote* (market moved organically) and *genuinely unknown*. Both still stamp `B9` in the MVP — `B4 stale_quote` is a future root cause code — but the narrative in the report is meaningfully different, and that detail is what searchers actually care about.

---

#### Summary

| Flow | Gate 1 (gap > 5%?) | Gate 2 (competitor?) | Result | Narrative |
|---|---|---|---|---|
| 1 | YES | YES | `A2 → B1` | "This tx frontran you" |
| 2a | NO | — | `A2 → B9` | "Normal variance, not worth investigating" |
| 2b | YES | NO | `A2 → B9` | "Investigated, nothing found — here's what I ruled out" |

**The two `B9` exits are not the same answer.** Variant 2a is a short-circuit with no further investigation. Variant 2b is a full audit trail that earns the agent's credibility — the searcher sees every tool call that ran and every hypothesis that was ruled out. That distinction is the core trust signal of this tool.

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
           │    Hono API Server          │
           │  - GET  /trades             │
           │  - POST /investigate (SSE)  │
           │  - GET  /reports/:id        │
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
   │  tenderlyClient  │   │      rpcClient       │
   │  - simulate      │   │  - getBlock          │
   │  - trace         │   │  - getTransactionR.. │
   │  - stateOverride │   │  - getLogs           │
   └──────────────────┘   └──────────────────────┘
              │                       │
              ▼                       ▼
       Tenderly API         Ethereum RPC (Alchemy)
                          │
                          ▼
           ┌─────────────────────────────┐
           │      reports/ (JSON)        │
           │  - {tx_hash}.json           │
           │  - investigation_log.ndjson │
           └─────────────────────────────┘
```

**Packages (monorepo):**

- `packages/shared` — shared TypeScript types: `TradeRef`, `TradeReport`, `FailureCategory`, `Evidence`, `ToolCall`
- `packages/tenderly-client` — typed fetch wrapper for Tenderly REST API: simulation, traces, state overrides
- `packages/rpc-client` — thin wrapper around `viem` for block/tx/receipt/log calls and ABI decoding
- `packages/pool-math` — Uniswap v2/v3 expected-PnL calculation using `@uniswap/v3-sdk`. Do **not** roll your own tick math.
- `packages/agent` — the Claude tool-use loop, tool definitions, system prompt, tool-call budget enforcement, citation verification
- `apps/api` — Hono server, SSE streaming for live tool-call timelines
- `apps/web` — Next.js frontend

---

## 9. Tech stack

**Backend:**
- **TypeScript + Node.js** — monorepo via `pnpm workspaces`
- **viem** — Ethereum RPC client: block fetching, log decoding, ABI parsing, pool state reads
- **Tenderly REST API** — simulations, traces, state overrides (plain `fetch`, no official SDK needed)
- **Alchemy** — Ethereum mainnet RPC node
- **Hono** — lightweight HTTP server + native SSE streaming
- **@anthropic-ai/sdk** — Claude API with native tool-use support; typed `tool_use` / `tool_result` blocks out of the box
- **@uniswap/v3-sdk + @uniswap/sdk-core** — tick math, pool state calculations, expected PnL
- **JSON files on disk** — report storage during hackathon; no database overhead

**Frontend:**
- **Next.js + TailwindCSS**
- **Server-Sent Events** — live tool-call streaming from Hono → Next.js
- Components: trade list + chat + evidence panel + tool-call timeline

**External:**
- **EigenPhi** — source real MEV wallet addresses for the demo dataset
- **Tenderly** — sponsor; lean into it visibly in the demo UI

**Monorepo layout:**
```
mev-forensics/
├── apps/
│   ├── api/          # Hono server
│   └── web/          # Next.js dashboard
├── packages/
│   ├── shared/       # types shared across api + web
│   ├── agent/        # Claude tool-use loop
│   ├── tenderly-client/
│   ├── rpc-client/
│   └── pool-math/
├── pnpm-workspace.yaml
└── turbo.json        # optional, speeds up builds
```

---

## 10. Tool layer

5 tools. Each one backs a specific evidence need from the 2 MVP paths.

| Tool | Signature (sketch) | Backed by | Serves |
|---|---|---|---|
| `get_trade` | `(tx_hash) → {from, to, block, index, value, gas, status, calldata, logs, realized_pnl}` | viem RPC + ABI decoder | Both paths |
| `simulate_at_state` | `(tx_hash, block, state_override?) → {simulated_pnl, success, revert_reason}` | Tenderly | Both paths — **most important tool** |
| `get_block_txs` | `(block_number) → [{index, tx_hash, from, to, touched_pools}]` | viem getLogs (Swap event filter) | B1, B9 |
| `get_tx_trace` | `(tx_hash) → call_tree` | Tenderly | B1 confirmation |
| `get_pool_state` | `(pool_address, block) → {reserves \| ticks, sqrt_price, liquidity}` | viem + @uniswap/v3-sdk | B9 (ruling out stale quote) |

**Dropped for MVP:** `get_gas_distribution`, `get_address_history`

**Tool budget:** max 8 tool calls per investigation turn. If the agent hasn't resolved the case by then, it must report `unknown` with what it tried.

**Rate limits:** Cache every Tenderly simulation result in memory keyed by `${tx_hash}:${block}:${stateOverrideHash}`. Pre-warm all demo trades before the live demo.

**Key implementation note for `get_block_txs`:** `touched_pools` requires filtering Swap event logs for the block using viem's `getLogs` with Uniswap V2/V3 Swap topic hashes — not just address matching. Budget this; it's a half-day with viem but the ABI decoding is clean.

---

## 11. Agent design

### Claude tool-use loop (TypeScript)

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

async function runInvestigation(txHash: string): Promise<TradeReport> {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: `Investigate this trade: ${txHash}` }
  ];

  let toolCallCount = 0;

  while (true) {
    const response = await client.messages.create({
      model: "claude-sonnet-4-6",
      max_tokens: 4096,
      system: SYSTEM_PROMPT,
      tools: TOOL_DEFINITIONS,
      messages,
    });

    // Stream tool-call events to frontend via SSE here

    if (response.stop_reason === "end_turn") {
      return parseReport(response);
    }

    if (response.stop_reason === "tool_use") {
      const toolUseBlocks = response.content.filter(
        (b): b is Anthropic.ToolUseBlock => b.type === "tool_use"
      );

      toolCallCount += toolUseBlocks.length;
      if (toolCallCount >= 8) {
        // Force unknown report — inject budget-exceeded message
        messages.push({ role: "assistant", content: response.content });
        messages.push({
          role: "user",
          content: "Tool budget exhausted. Report A2 → B9 with what you tried.",
        });
        continue;
      }

      const toolResults = await Promise.all(
        toolUseBlocks.map(async (block) => ({
          type: "tool_result" as const,
          tool_use_id: block.id,
          content: JSON.stringify(await dispatchTool(block.name, block.input)),
        }))
      );

      messages.push({ role: "assistant", content: response.content });
      messages.push({ role: "user", content: toolResults });
    }
  }
}
```

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
3. Compare expected vs realized. If within 5%: report A2 → B9 (no clear
   root cause) and stop.
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
- **Streaming:** tool call events stream via SSE to the Next.js frontend in real time

### Response schema
```typescript
interface TradeReport {
  tx_hash: string;
  outcome: "A2" | "A9";
  root_cause: "B1" | "B9" | null;
  expected_pnl: number | null;      // USD
  realized_pnl: number;             // USD
  pnl_delta: number | null;         // USD
  confidence: number;               // 0.0–1.0
  evidence: Evidence[];             // structured citations
  counterfactuals: Counterfactual[];
  narrative: string;                // the agent's explanation
  tool_calls: ToolCall[];           // full audit trail
}
```

---

## 12. Data & demo strategy

### Dataset
- **2 real MEV wallets** sourced from EigenPhi leaderboards
- Hand-curate **2 demo trades** — one per path. Verify both by hand before the hackathon starts.
- Pre-warm Tenderly simulation cache for both trades. The live demo must never hit a cold API call.

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
6. Switch to trade #2. Agent works through the same steps, runs `get_block_txs` (no competitor), then `get_pool_state` to check for organic price movement, reports B9. *"I investigated and cannot confidently classify this. Here's what I ruled out."*
7. **That's the close.** The unknown case is your most memorable moment.

---

## 13. Execution plan (12-day)

| Day | Focus | Done when |
|---|---|---|
| **1** | Monorepo scaffolding (`pnpm workspaces`, shared types, Hono server shell, Next.js shell). `rpc-client` wired to Alchemy. `get_trade` tool returning real data for one tx. | You can print a real tx's block, index, and logs to stdout. |
| **2** | `simulate_at_state` via Tenderly API. Validate simulated PnL against Etherscan realized PnL for a *winning* trade (clean case first). Pin block number semantics — confirm Tenderly `block_number: N` gives post-N state. | Simulated PnL matches realized within 2% on a clean trade. |
| **3** | `get_block_txs` with viem `getLogs` Swap topic filtering. `touched_pools` extraction from decoded logs. Curate both demo trades. | You can list all Uniswap txs in a block with their touched pools. |
| **4** | Claude tool-use loop end-to-end (Path 1: A2 → B1). Agent correctly identifies the frontrunning tx on demo trade #1. | Agent produces a cited B1 report on demo trade #1. |
| **5** | `get_pool_state` + Path 2 (A2 → B9). Citation verification post-processor. Pre-warm Tenderly cache for both demo trades. Record backup video. | Both paths working. Backup video recorded. |
| **6** | SSE streaming from Hono → Next.js. Tool-call timeline component in the UI. Real data flowing into the dashboard. | Live tool-call stream visible in browser during investigation. |
| **7** | Evidence panel UI. Chat interface polish. Shared types between `packages/shared` → `apps/web`. | End-to-end demo runnable in browser with real data. |
| **8** | Buffer day — fix whatever broke. If ahead: add Tenderly branding to tool-call timeline (sponsor visibility). | Demo runs 5 times in a row without failure. |
| **9** | Demo rehearsal. Pitch deck. README final pass. | 10 clean runs. Pitch deck done. |
| **10** | Submission polish. No new features. | Submitted. |
| **11–12** | Reserve / overflow. Do not plan features here. | — |

**The rule for days 1–5:** don't touch the frontend beyond scaffolding. Don't add new tools. Don't read about B3. The only question is: do both paths work reliably on your two curated trades?

---

## 14. Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Tenderly API flaky during demo | Medium | High | Cache everything in memory + pre-warm + recorded backup video |
| `simulate_at_state` off-by-one on block state | Medium | High | Validate on day 2 against a known clean trade before building anything else |
| V3 tick math wrong (expected PnL) | Medium | High | Use `@uniswap/v3-sdk` exclusively — don't roll your own. Validate on day 2. |
| Agent loops / over-investigates | Medium | High | Hard 8-call budget enforced in the loop, not the prompt |
| Agent hallucinates uncited claims | Medium | High | Post-process citation check; strip uncited lines before sending to UI |
| `get_block_txs` slow on dense blocks | Medium | Medium | Filter logs by Swap topic first; don't fetch full tx bodies for every tx in block |
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
- Multi-chain support (mainnet, Arbitrum, Optimism, Base)
- Curve, Balancer, other AMM support beyond Uni v2/v3
- Gas counterfactual: "if you'd bid +3 gwei, you'd have landed at index 4"
- Database upgrade: replace JSON files with QuestDB for aggregate analytics
