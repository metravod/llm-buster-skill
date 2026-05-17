# LLM call audit — dileepkanumuri/FinSight-AI-Agent

**Date:** 2026-05-17
**Skill version:** v0.2.0 (with ❌ Out-of-scope + 🟢 sub-task extraction modes)
**Scope:** OpenAI + Anthropic SDKs (Python + TS/JS), raw HTTP, LangChain wrappers
**Inventory method:** ripgrep over the repo (5 .py files at root); manual read of each
**Calls found:** 12 across 5 files

## Verdict summary

| Verdict | Count | Notes |
|---|---|---|
| 🟢 Replace now | 2 | One is a **no-op LLM call** (rephrases its own input); one is template-able query generation |
| 🟡 Worth trying | 1 | Query generation from free-form feedback prose |
| 🔴 Keep | 4 | Genuine analyst-style generation across the LangGraph pipeline |
| **❌ Out of scope** | **5** | 4 files of tutorial/scaffolding code (Nelson Mandela smoke test, ReAct planet_mass tutorial, LangGraph hello-world × 3) |
| **Total** | **12** | |

This repo has an unusually clean split: one production file (`finance_agent.py`) with 7 real calls, and four "I was following a LangChain tutorial" scratch files left in the root. The new ❌ verdict cleanly carves out the latter so the actionable audit is just the 3 calls in `finance_agent.py`.

## Per-file pivot

| File | Calls | Verdict mix | Status |
|---|---|---|---|
| `finance_agent.py` | 7 | 2 🟢 / 1 🟡 / 4 🔴 | **In scope — actionable findings below** |
| `simple_agent.py` | 2 | 2 ❌ | Tutorial: "who is Nelson Mandela?" smoke test + ReAct `planet_mass`/`calculate` demo |
| `simple_agent_hum_in_loop.py` | 1 | 1 ❌ | LangGraph "human-in-the-loop with interrupt" tutorial, hardcoded `"I'm learning about astrology..."` input |
| `simple_agent_lngraph.py` | 1 | 1 ❌ | LangGraph hello-world chatbot, `while True: input()` REPL |
| `simple_agent_lngraph_tools.py` | 1 | 1 ❌ | LangGraph `bind_tools` tutorial with Tavily |

The 4 ❌ files are tutorial/learning code, not part of the deployed agent. The Streamlit `main()` entry point is in `finance_agent.py`; the others are never imported from it. Reasonable cleanup recommendation: move them under `examples/` or `tutorials/` so future readers (and audits) don't have to re-classify them.

---

## 🟢 Replace now

### 🟢 `finance_agent.py:69` — `gather_financials_node` (no-op LLM call)

**Category:** rephrasing / pass-through
**Confidence:** high
**Model:** `gpt-3.5-turbo`

**Current call**

```python
GATHER_FINANCIALS_PROMPT = """You are an expert financial analyst. Gather the financial data for the given company. Provide detailed financial data."""

def gather_financials_node(state: AgentState):
    csv_file = state["csv_file"]
    df = pd.read_csv(StringIO(csv_file))
    financial_data_str = df.to_string(index=False)
    combined_content = f"{state['task']}\n\nHere is the financial data:\n\n{financial_data_str}"
    messages = [
        SystemMessage(content=GATHER_FINANCIALS_PROMPT),
        HumanMessage(content=combined_content),
    ]
    response = model.invoke(messages)
    return {"financial_data": response.content}
```

**Why it's 🟢:** the CSV is already loaded into a pandas DataFrame and rendered to a string inside the function. The LLM is then asked to "gather the financial data" — data that's literally in its input prompt — and emit "detailed financial data". The output goes verbatim into `state["financial_data"]`, which `analyze_data_node` reads.

This is the LLM doing **nothing useful**. The downstream consumer doesn't need a rephrased version of the CSV; it can read the CSV string directly. The function name (`gather_financials`) suggests an original design where the LLM was meant to scrape financial data from the web, but the current implementation already does the reading. The LLM call is leftover scaffolding.

**Proposed replacement**

```python
def gather_financials_node(state: AgentState):
    csv_file = state["csv_file"]
    df = pd.read_csv(StringIO(csv_file))
    financial_data_str = df.to_string(index=False)
    return {"financial_data": financial_data_str}
```

Two lines saved, one LLM call per run eliminated, zero quality loss because the downstream prompt builds on the CSV string anyway.

**Risks**

- If the design intent was "LLM normalizes vendor-specific CSV formats", the swap loses that. The current prompt is too vague to absorb that work — if it ever happens, the prompt would say so. Read commit history if uncertain.

**Per-call cost (current):** input ~1-3k tokens (CSV + task) × $0.50/M + output ~500-1500 tokens × $1.50/M = **~$0.001-$0.004/call**. Per pipeline run: 1 call. Saved ~1s of latency and removes an unnecessary failure mode.

---

### 🟢 `finance_agent.py:97` — `research_competitors_node` (template-able query generation)

**Category:** sub-task extraction — query generation from a closed input (competitor name)
**Confidence:** high
**Model:** `gpt-3.5-turbo` with `with_structured_output(Queries)`
**Promoter:** P1 (strict pydantic schema on output)

**Current call**

```python
class Queries(BaseModel):
    queries: List[str]

RESEARCH_COMPETITORS_PROMPT = """You are a researcher tasked with providing information about similar companies for performance comparison. Generate a list of search queries to gather relevant information. Only generate 3 queries max."""

def research_competitors_node(state: AgentState):
    content = state["content"] or []
    for competitor in state["competitors"]:
        queries = model.with_structured_output(Queries).invoke(
            [
                SystemMessage(content=RESEARCH_COMPETITORS_PROMPT),
                HumanMessage(content=competitor),
            ]
        )
        for q in queries.queries:
            response = tavily.search(query=q, max_results=2)
            ...
```

**Why it's 🟢:** input is a single string (competitor name like `"Microsoft"`), output is 3 search queries. The 3 queries the LLM produces are entirely predictable (variants of "{competitor} financial performance", "{competitor} revenue", "{competitor} earnings"). Pure template territory. Promoter P1 fires (`Queries` is a strict pydantic schema). No disqualifiers — input isn't free-form natural language, it's a known company name.

**Proposed replacement**

```python
SEARCH_QUERY_TEMPLATES = [
    "{competitor} financial performance 2024",
    "{competitor} revenue and earnings",
    "{competitor} annual report key metrics",
]

def research_competitors_node(state: AgentState):
    content = state["content"] or []
    for competitor in state["competitors"]:
        queries = [t.format(competitor=competitor) for t in SEARCH_QUERY_TEMPLATES]
        for q in queries:
            response = tavily.search(query=q, max_results=2)
            for r in response["results"]:
                content.append(r["content"])
    return {"content": content}
```

**Risks**

- If competitors are exotic (private companies, recent IPOs, non-English names), the LLM's queries might be more nuanced than the template. Mitigation: keep the LLM path behind a feature flag for the first 50 runs; A/B compare Tavily result quality.
- Template needs occasional refresh as Tavily's results shift over time.

**Per-call cost (current):** input ~120 tokens × $0.50/M + output ~80 tokens × $1.50/M = **~$0.00018/call** × N competitors per run. Saving is small per-call but removes a structured-output failure mode (pydantic validation can throw on weak gpt-3.5-turbo output).

---

## 🟡 Worth trying

### 🟡 `finance_agent.py:120` — `research_critique_node` (query generation from feedback prose)

**Category:** sub-task extraction — but with free-form input
**Confidence:** medium
**Model:** `gpt-3.5-turbo` with `with_structured_output(Queries)`

**Current call**

```python
RESEARCH_CRITIQUE_PROMPT = """You are a researcher tasked with providing information to address the provided critique. Generate a list of search queries to gather relevant information. Only generate 3 queries max."""

def research_critique_node(state: AgentState):
    queries = model.with_structured_output(Queries).invoke(
        [
            SystemMessage(content=RESEARCH_CRITIQUE_PROMPT),
            HumanMessage(content=state["feedback"]),  # <-- feedback is free-form prose
        ]
    )
    ...
```

**Why it's 🟡 not 🟢:** same shape as `research_competitors_node` (P1 fires, output is `Queries`), but **disqualifier D1 hits** — `state["feedback"]` is free-form prose from `collect_feedback_node` (another LLM). Templating queries from arbitrary critique text doesn't work; you'd need to extract topics first.

**Proposed approach (two-stage)**

```python
# Stage 1 — extract key topics from feedback with KeyBERT or YAKE (both run locally, no model API call)
from keybert import KeyBERT
kw = KeyBERT()
topics = [t for t, _ in kw.extract_keywords(feedback, keyphrase_ngram_range=(2, 4), top_n=3)]

# Stage 2 — template queries from topics
queries = [f"{topic} financial analysis" for topic in topics]
```

Shadow-mode this for 30 runs — compare Tavily result overlap with the LLM path. If overlap ≥ 80%, swap.

**Risks**

- KeyBERT's defaults may pick generic topics ("financial performance", "revenue growth") that don't differentiate the critique. Tune `keyphrase_ngram_range` and `top_n` on real critiques before shipping.

**Per-call cost (current):** similar to `research_competitors_node` — **~$0.0002/call**. Replacing also removes the dependency on `gpt-3.5-turbo` returning valid pydantic.

---

## 🔴 Keep

| Location | Function | Why kept |
|---|---|---|
| `finance_agent.py:85` | `analyze_data_node` | "Analyze the provided financial data and provide detailed insights and analysis" — open-ended analyst prose for downstream consumption |
| `finance_agent.py:107` | `compare_performance_node` | "Compare the financial performance...like a world-class analyst would" — generation |
| `finance_agent.py:128` | `collect_feedback_node` | "Provide detailed feedback and critique" — open-ended reviewer prose |
| `finance_agent.py:136` | `write_report_node` | "Write a comprehensive financial report" — long-form generation, shown verbatim to user via Streamlit |

All four are generation (category 8) / reasoning (category 9), no promoters strong enough to lift them, output flows to the human reader via Streamlit. Genuine LLM territory.

---

## ❌ Out of scope

The four `simple_agent*.py` files are tutorial / scaffolding code, not part of the deployed agent:

| File | Why ❌ |
|---|---|
| `simple_agent.py` | Top-level smoke test (`"who is Nelson Mandela?"`) + `Agent` class with the canonical ReAct `planet_mass`/`calculate` tutorial example. Half the file is commented-out demo invocations. Not imported by `finance_agent.py`. |
| `simple_agent_hum_in_loop.py` | LangGraph "human-in-the-loop with interrupt" tutorial. Hardcoded `user_input = "I'm learning about astrology..."`. Not imported. |
| `simple_agent_lngraph.py` | LangGraph hello-world chatbot with `while True: input()` REPL. Not imported. |
| `simple_agent_lngraph_tools.py` | LangGraph `bind_tools` + Tavily tutorial. Not imported. |

These are the developer's "I was following a LangChain tutorial" scratch files. Applying the colour rubric to them would emit noise — the LLM call doesn't serve a product goal, it serves a learning goal. **❌ verdict skips them cleanly.**

**Cleanup recommendation:** move under `examples/` or `tutorials/` and add a one-line README pointer; future audits + future contributors then don't have to re-classify them.

---

## Aggregate impact estimate

Per Streamlit run (one company, ~3 competitors, `max_revisions=2`):

- `gather_financials_node` × 1 → **🟢 drop**
- `analyze_data_node` × 1 → 🔴
- `research_competitors_node` × 3 → **🟢 drop**
- `compare_performance_node` × 1-3 (revision loop) → 🔴
- `collect_feedback_node` × 1-2 → 🔴
- `research_critique_node` × 1-2 → 🟡
- `write_report_node` × 1 → 🔴

**Replaceable per run:** 4 LLM calls (1 gather + 3 competitor query gen).

**Per-call cost:** ~$0.0005 average → savings ~$0.002/run. Negligible in dollars (~10k runs to save $20). The wins are:

1. **Latency** — `gather_financials_node` removed entirely (~1s saved per run); `research_competitors_node` LLM call replaced with formatting (~3 × 1s = ~3s saved per run).
2. **Failure modes removed** — both 🟢 cases use `with_structured_output(Queries)` which can throw on weak `gpt-3.5-turbo` JSON adherence. Templates can't throw.
3. **Reproducibility** — same competitor input always yields the same Tavily query, which makes the rest of the pipeline more reproducible for testing.

**Token counting method:** estimates by character length × 0.25 (cl100k_base ratio for gpt-3.5-turbo). Run `tiktoken` on a real CSV input for precise numbers.

## Next steps

1. **Apply the two 🟢 replacements in `finance_agent.py`** — both are < 10 lines, no behavioural risk for the `gather_financials_node` case, low-risk template for `research_competitors_node`.
2. **Move `simple_agent*.py` under `examples/`** — pure hygiene; stops future audits and contributors from wasting time on tutorial code.
3. **(Optional) Wire KeyBERT for `research_critique_node`** — shadow-mode for 30 runs, compare Tavily result overlap.

---

## Skill self-test — how v0.2.0 performed

This audit exercised three of the v0.2.0 additions; reporting on each:

- **❌ Out-of-scope verdict.** Fired cleanly on the 4 tutorial files. Without it, the audit would have produced four near-identical 🟡/🔴 verdicts on the same canned ReAct / LangGraph hello-world prompts and buried the one real finding. The disqualifier "files not imported by the production entry point" wasn't formalized in `rubric.md` § ❌ but worked here by inspection — worth adding to the heuristics list as a 6th trigger ("file is unreferenced by the main app").
- **🟢 sub-task extraction.** Used implicitly on `research_competitors_node` — the call is structurally a "🟡 sub-task" but the input is so closed (a competitor name) that the verdict promotes to 🟢 outright. The new mode handled this without friction.
- **Duplication pass.** Did not fire — the 5 ❌ files use different prompts, not the same call. As expected, the pass only triggers on cross-file same-task patterns.
- **Per-file pivot.** Used the lightweight version (5 files, not 3 subprojects) — the table-then-detail format scaled down cleanly.

One small gap surfaced: the **"no-op LLM call"** pattern (`gather_financials_node` rephrases data already in its prompt) doesn't have a named place in `taxonomy.md`. It maps loosely to "rephrasing (7)" but the verdict bias for rephrasing is 🔴, while this case is clearly 🟢. Worth one sentence in `taxonomy.md` § 7 noting that "rephrasing where the output is consumed by downstream code, not a human" → 🟢.

> _Audit produced by `llm-buster` skill v0.2.0 (as `de-llmer`, pre-rebrand)._
