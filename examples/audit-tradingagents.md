# LLM call audit — TauricResearch/TradingAgents

**Date:** 2026-05-17
**Scope:** OpenAI + Anthropic SDKs (Python + TS/JS) and raw HTTP — extended here to `langchain_openai`, `langchain_anthropic`, `langchain_google_genai`, `langchain_ollama` because all calls go through LangChain wrappers (per `references/detection.md` § "wrapper frameworks").
**Inventory method:** ripgrep for `from langchain_*` + `ChatPromptTemplate.from_messages` + `\.invoke\(` across `tradingagents/agents/` and `tradingagents/graph/`. Wrapper grep returned the call sites; the prompts themselves live in each `create_<role>` factory.
**Calls found:** 14 active LLM call sites across 13 files (plus 1 already-de-LLM'd site retained for evidence).

## At-a-glance

This is a **mature multi-agent LangGraph trading system** (analysts → researchers → risk debaters → trader → portfolio manager → reflection). Each agent is a `create_<role>(llm)` factory returning a graph node that invokes the LLM on a curated prompt + state. The architecture is intentionally LLM-heavy: the entire value proposition is "specialized agents debate and synthesize", which is exactly what the skill's rubric marks 🔴.

**The honest finding: 12 of 14 calls are 🔴 — keep them.** The two 🟡 are sub-tasks _inside_ otherwise-LLM prompts that could be lifted out and made deterministic to make the surrounding LLM calls cheaper and more reliable, not removed entirely. There are **zero 🟢** in this codebase.

> The 🟢 verdict has already been redeemed once in this repo — see "Evidence" below.

## Verdict summary

| Verdict | Count | Notes |
|---|---|---|
| 🟢 Replace now | 0 (1 already done) | None active. One call site was already de-LLM'd by the maintainers — strong evidence the methodology fits this codebase. |
| 🟡 Worth trying | 2 (both as **sub-task extraction**, not full removal) | Indicator selection in `market_analyst`; StockTwits bull/bear-ratio bucketing in `sentiment_analyst`. |
| 🔴 Keep | 12 | Genuine generation, agentic tool-loops, multi-source reasoning, structured-output synthesis. |
| **Total active** | **14** | |

## Evidence — the team already applied this skill

**`tradingagents/graph/signal_processing.py`** used to call an LLM to extract a 5-tier rating ("Buy / Overweight / Hold / Underweight / Sell") from a Portfolio Manager decision string. That call has been **removed** and replaced with `tradingagents/agents/utils/rating.py::parse_rating` — a 25-line regex + token-match function. The old `quick_thinking_llm` parameter is kept for backwards compatibility but is documented as unused:

> "The LLM argument is accepted for backwards compatibility but no longer used: the PM's structured output guarantees the rating is parseable from the rendered markdown without a second LLM call."

This is a textbook 🟢 outcome and confirms the rubric works on this codebase. Use this anecdote when proposing the 🟡 splits below.

---

## 🟡 Worth trying

These are **sub-task extractions** — pull a deterministic decision out of an otherwise-LLM prompt and run it in code _before_ the LLM call. The LLM call itself stays, but it gets shorter input, structured signal, and more reliable downstream behaviour.

### 🟡 `tradingagents/agents/analysts/market_analyst.py:76` — indicator selection sub-task

**Category:** classification / routing (pick ≤8 from a fixed set of 11), buried inside an agentic report-writer
**Confidence:** medium
**Underlying LLM call:** agentic (🔴 — keeps), but the **selection step** is replaceable

**Current shape**

```python
system_message = """You are a trading assistant tasked with analyzing financial markets. Your role is to select the most relevant indicators for a given market condition or trading strategy from the following list. The goal is to choose up to 8 indicators that provide complementary insights without redundancy.

Moving Averages:
- close_50_sma: 50 SMA: A medium-term trend indicator. Usage: Identify trend direction and serve as dynamic support/resistance.
- close_200_sma: 200 SMA: A long-term trend benchmark. Usage: Confirm overall market trend and identify golden/death cross setups.
- close_10_ema: 10 EMA: A responsive short-term average...
... (8 more indicators with usage notes)

- Select indicators that provide diverse and complementary information. Avoid redundancy (e.g., do not select both rsi and stochrsi)."""
```

**Why it's 🟡 (sub-task only):** the indicator universe is closed (11 options), the selection criteria are stated, and "no redundancy" is a hardcoded rule. The wrapping LLM task (writing the trend narrative) genuinely is generation, but the picking is decision-table territory. A 30-line regime detector (ATR for volatility, ADX for trend strength, recent return for direction) can pick the 8 indicators deterministically and keep the picker stable across runs — currently the LLM picks differently on every call.

**Proposed split**

```python
# Step 1 — deterministic regime + indicator picker, runs before LLM
def pick_indicators(price_df) -> list[str]:
    atr = price_df["atr14"].iloc[-1] / price_df["close"].iloc[-1]
    adx = price_df["adx14"].iloc[-1]
    is_trending = adx >= 25
    is_volatile = atr >= 0.025  # ~2.5% daily ATR
    base = ["close_50_sma", "close_200_sma", "atr"]  # always include
    if is_trending:
        base += ["macd", "macds", "macdh"]
    if is_volatile:
        base += ["boll_ub", "boll_lb"]
    else:
        base += ["rsi", "boll"]
    return base[:8]

# Step 2 — LLM gets a shorter prompt focused on narrative, not selection
system_message = f"Analyze these pre-selected indicators for {ticker}: {picked}. ..."
```

**Effect on the rest of the agent:**
- The "select 8 indicators" instruction block (~3500 tokens of indicator usage notes) shrinks to a few hundred tokens.
- The LLM call stays — it still does the agentic tool calling and the report writing — but the input is smaller and the indicator choice becomes stable across re-runs (good for reproducibility / backtest comparisons).

**Risks**

- The regime detector needs to handle edge cases (insufficient data, gaps). Use the same fallback the LLM had: include the always-on set + RSI + BBands.
- A user who _wants_ different indicators on different runs (i.e. exploring) loses that flexibility — expose `pick_indicators(strategy_hint="...")` if needed.

**Per-call cost win:** ~3500 input tokens saved per `market_analyst` call. On Claude Sonnet 4 ($3/M input), that's ~$0.01/call × 1 call per trading decision per ticker.

---

### 🟡 `tradingagents/agents/analysts/sentiment_analyst.py:89` — StockTwits bull/bear-ratio bucketing

**Category:** classification (sentiment-ratio → bucket label), embedded in a generation prompt
**Confidence:** medium-high
**Underlying LLM call:** generation (🔴 — keeps), but the **arithmetic** is replaceable

**Current shape** (excerpt from the system prompt, lines 116-117 of the file):

> "Read the StockTwits Bullish/Bearish ratio as a leading retail-sentiment signal. A **70/30 bullish/bearish split is moderately bullish; ≥90/10 may indicate over-extension and contrarian risk; 50/50 is uncertainty**. Sample size matters — base rates on the actual message count, not percentages alone."

**Why it's 🟡:** the prompt is asking the LLM to count Bullish/Bearish labels in a string blob and apply a hardcoded heuristic (70/30, 90/10, 50/50). Both halves of that task — the count and the bucket — are deterministic. The LLM should be receiving a pre-computed `{"bull": 23, "bear": 7, "untagged": 12, "ratio": 0.77, "bucket": "moderately bullish", "sample_warn": false}` and synthesizing it with the news/Reddit narratives. Currently the LLM is re-doing arithmetic on every call, and weaker models (Haiku, Mini) routinely miscount when the message list runs long.

**Proposed split**

```python
# fetch_stocktwits_messages already returns the messages — extend to compute stats
from collections import Counter

def stocktwits_sentiment_summary(messages: list[dict], small_sample: int = 10) -> dict:
    counts = Counter(m.get("sentiment", "untagged") for m in messages)
    bull = counts.get("Bullish", 0)
    bear = counts.get("Bearish", 0)
    total = bull + bear
    if total == 0:
        return {"bucket": "no labeled signal", "bull": bull, "bear": bear, "sample_warn": True}
    ratio = bull / total
    if ratio >= 0.9:
        bucket = "over-extended bullish (contrarian risk)"
    elif ratio >= 0.65:
        bucket = "moderately bullish"
    elif ratio <= 0.1:
        bucket = "over-extended bearish (contrarian risk)"
    elif ratio <= 0.35:
        bucket = "moderately bearish"
    else:
        bucket = "uncertain / mixed"
    return {
        "bull": bull, "bear": bear, "ratio": round(ratio, 2),
        "bucket": bucket, "sample_warn": total < small_sample,
    }
```

Then in `_build_system_message`, replace the StockTwits block intro with:

```
### StockTwits messages — retail-trader social platform
Pre-computed sentiment stats: bull={bull}, bear={bear}, ratio={ratio}, bucket="{bucket}"{warn}
Raw messages follow for context only — the ratio is authoritative.
```

The LLM still reads the raw messages for context (which themes/cashtag activity) but no longer has to do the bucket judgment. Same as how the team handled `parse_rating` for the Portfolio Manager.

**Risks**

- Whoever updates the bucket thresholds (currently 70/30 and 90/10) now has to edit two places (prompt + Python). Centralize the thresholds in a constant block at the top of the file, similar to `RATINGS_5_TIER` in `rating.py`.
- Untagged messages (no Bullish/Bearish tag) are ignored — same as the current LLM heuristic. Document this in the function docstring.

**Per-call cost win:** marginal (~50 tokens saved). The actual win is **reliability** — bucket label is identical across re-runs of the same data; LLM output isn't. Same kind of win `signal_processing.py` delivered for the 5-tier rating.

---

## 🔴 Keep — multi-agent core

All 12 sites below are 🔴. The justification is one of: agentic tool-loop (category 10), open-ended generation of analyst-style prose (category 8), or multi-source reasoning over compiled debate history (category 9). Listed for completeness.

| # | Location | Function | Dominant category | Why kept |
|---|---|---|---|---|
| 1 | `agents/analysts/fundamentals_analyst.py:57` | `create_fundamentals_analyst` | agentic (10) | `bind_tools` over balance-sheet/cashflow/income tools + writes comprehensive markdown report |
| 2 | `agents/analysts/market_analyst.py:76` | `create_market_analyst` | agentic (10) | Same shape as #1; technical-indicator narrative. See 🟡 for sub-task split. |
| 3 | `agents/analysts/news_analyst.py:50` | `create_news_analyst` | agentic (10) | Tool-loop over `get_news`, `get_global_news` + macro narrative |
| 4 | `agents/analysts/sentiment_analyst.py:89` | `create_sentiment_analyst` | generation (8) + reasoning (9) | Multi-source synthesis (news + StockTwits + Reddit). See 🟡 for sub-task split. |
| 5 | `agents/researchers/bull_researcher.py:35` | `create_bull_researcher` | generation/reasoning + agentic debate | Adversarial debate; explicitly told to "engage in dynamic debate" |
| 6 | `agents/researchers/bear_researcher.py:37` | `create_bear_researcher` | same as #5 | Mirror role of #5 |
| 7 | `agents/risk_mgmt/aggressive_debator.py:34` | `create_aggressive_debator` | same | Three-way debate over trader's plan |
| 8 | `agents/risk_mgmt/conservative_debator.py:34` | `create_conservative_debator` | same | Mirror of #7 |
| 9 | `agents/risk_mgmt/neutral_debator.py:34` | `create_neutral_debator` | same | Mirror of #7 |
| 10 | `agents/managers/research_manager.py:45` | `create_research_manager` | reasoning (9) + structured output | Synthesizes bull/bear debate into typed `ResearchPlan` via `with_structured_output`; **promoter P1 fires but the input is unstructured debate prose — keeps the verdict at 🔴.** |
| 11 | `agents/trader/trader.py:51` | `create_trader` | reasoning (9) + structured output | Same shape as #10 — typed `TraderProposal` |
| 12 | `agents/managers/portfolio_manager.py:66` | `create_portfolio_manager` | reasoning (9) + structured output | Same shape — typed `PortfolioDecision`. Output is what `parse_rating` later consumes. |
| 13 | `graph/reflection.py:57` | `Reflector.reflect_on_final_decision` | generation (8) | 2–4 sentence post-trade lesson; prompt is tight and well-engineered but output is genuinely free-form prose for a decision log |

**On the structured-output trio (#10, #11, #12).** Each uses `llm.with_structured_output(Schema)` (promoter P1 in the rubric). The output _is_ schema-bound, which would normally lift the verdict. But the **input** is the full debate history (free-form prose generated by debators upstream), and the task is to synthesize that prose into a decision. That's the rubric's "reasoning over free-form input" — keeps the verdict 🔴 even with structured output on the response side.

**On `Reflector.reflect_on_final_decision` (#13).** The prompt has a structured 3-question ordered template:
> "1. Was the directional call correct? (cite the alpha figure) 2. Which part of the investment thesis held or failed? 3. One concrete lesson..."

Question 1 is trivially deterministic (`"correct" if alpha_return > 0 else "incorrect"`). The remaining two genuinely need the LLM. Worth a one-line code change before the LLM call — pre-state the directional verdict — but not a full call-site swap.

---

## Aggregate impact estimate

Per single trading-decision run (one ticker, one date):

- ~4 analyst calls (fundamentals / market / news / sentiment), each ~10–40k tokens in / 2–5k out
- 2 researcher rounds × bull/bear = ~4 calls, each ~15–30k in / 2–4k out
- 1 research manager
- 1 trader
- 3 risk debaters (1 round) — could be more rounds
- 1 portfolio manager
- 1 reflection (post-outcome)

Rough total per run: **~15–25 LLM calls**, depending on debate rounds.

**The 🟡 splits don't cut call count** — they cut tokens per call (mostly on `market_analyst`) and improve reliability on the bucket labels. The dollar impact depends on which provider is configured:

| Provider configured | Per-run cost (est., conservative) | After 🟡 splits |
|---|---|---|
| Claude Sonnet 4 ($3/$15 per M) | ~$0.50–1.50/run | ~$0.49–1.45/run (–~3%) |
| GPT-4o-mini ($0.15/$0.60 per M) | ~$0.04–0.10/run | ~$0.04–0.10/run (–~3%) |
| Local model (Ollama) | $0 + GPU time | unchanged |

**Reading:** the cost win is small. The **reliability** win on `sentiment_analyst` (bucket label stable across re-runs) and the **reproducibility** win on `market_analyst` (same indicator set selected for same regime) are the real reasons to do these. Same pattern as the team already validated with `parse_rating`.

**Token counting method:** estimates derived from prompt length × Claude tokenizer ratio (~0.25 tokens per character for English markdown). Run Anthropic's `count_tokens` API on a representative `messages=[...]` payload for a real number; the methodology stands either way.

## Next steps

1. **Lift indicator selection out of `market_analyst`** — write `pick_indicators(price_df)` as a sibling module to `rating.py`, parameterize the LLM prompt to receive a pre-selected list. ~2 hours, mirrors the `parse_rating` pattern.
2. **Pre-compute StockTwits bull/bear bucket in `sentiment_analyst`** — extend `fetch_stocktwits_messages` or wrap it, drop the ratio rules from the prompt, leave the messages in the prompt for narrative context only. ~30 minutes.
3. **(Trivial) Pre-state directional verdict in `Reflector`** — one line replacing question 1 of the reflection prompt with the computed fact.

Do these in three separate PRs, one per call site, so the regression surface stays small.

## Architectural compliment

This codebase already does most of what a de-LLM audit would recommend:

- ✅ `with_structured_output(Schema)` for the three synthesis agents (Trader / Research Manager / Portfolio Manager) — typed `BaseModel` outputs instead of `ast.literal_eval(prompt_output)`.
- ✅ Graceful fallback to free-text when a provider lacks structured-output (`invoke_structured_or_freetext` in `agents/utils/structured.py`).
- ✅ Deterministic `parse_rating` for the 5-tier label — the de-LLM win the maintainers already shipped.
- ✅ Pre-fetched data sources in `sentiment_analyst` (no LLM-driven scraping; the issue tracker explicitly cites #557 as the reason this was rebuilt).
- ✅ Central `RATINGS_5_TIER` vocabulary used by 4 different call sites — exactly the kind of dedup an audit would push for.

The 🟡 findings extend the existing pattern; they don't conflict with it.

> _Audit produced by `llm-buster` skill v0.1.0 (as `de-llmer`, pre-rebrand)._
