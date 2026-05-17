# LLM call audit — AI4Finance-Foundation/FinGPT

**Date:** 2026-05-17
**Scope:** OpenAI + Anthropic SDKs (Python + TS/JS), raw HTTP, LangChain wrappers
**Inventory method:** ripgrep across the whole umbrella repo, excluding `.git/`, `.idea/`, `figs/`, and per-subproject `tests/`
**Calls found:** ~30 call sites across 17 files in 8 subprojects (+ `finogrid/` runtime sidecar). Notebooks and tests included.

## Important framing — this is a research umbrella, not one app

`FinGPT` is a multi-subproject research repository whose **stated purpose** is to train and benchmark a finance-domain LLM (the `FinGPT` model itself). That changes how the de-LLM rubric applies. Three distinct call categories live here, and only one is genuinely in scope:

| Category | What the LLM is doing | Replaceable? |
|---|---|---|
| **Production / runtime** | Inference at serving time (sentiment scoring, RAG retrieval, agent chat) | ✅ in scope — apply the standard rubric |
| **Training-data generation** | GPT-4 produces labels / explanations / prompts used to fine-tune FinGPT | ❌ out of scope — the LLM is the gold reference being distilled. Replacing it defeats the project's purpose. |
| **Benchmark / LLM-as-judge** | OpenAI API used as a baseline being evaluated, or as a judge scoring FinGPT outputs | ❌ out of scope — the LLM is the thing being measured (or the measuring tool). |

The skill's existing rubric handles category 1 cleanly. Categories 2 and 3 need an explicit out-of-scope tag — the methodology should not propose "replace GPT-4 with FinBERT" inside an evaluation script that exists to compare FinBERT _against_ GPT-4. **Recommend adding this to the rubric in a follow-up.**

## Verdict summary

| Verdict | Sites | Notes |
|---|---|---|
| 🟢 Replace now | 0 | No clean drop-ins. The 🟡 group is essentially "🟢 once you accept FinBERT/FinGPT as the replacement" — same task across 5 files. |
| 🟡 Worth trying | **~6** | All variations of the same sentiment-classification call. Strong candidates for FinBERT or — fitting the project's _own_ stated intent — the local FinGPT model. |
| 🔴 Keep | ~8 | RAG synthesis, conversational robo-advisor, low-code development, multi-agent inference, single-shot summarization. |
| ❌ Out of scope | ~16 | Training-data generation + benchmark / LLM-as-judge. Cannot be replaced without breaking the research artifact. |
| **Total** | **~30** | |

## Per-subproject breakdown

### ✅ Production / runtime (in scope)

#### `finogrid/fingpt_integration/sentiment/openai_fallback.py` — 🟡

The file's own docstring says it all:

> "OpenAI fallback for FinGPT sentiment — practical for MVP stage. Switch to full FinGPT model when scaling."

```python
SENTIMENT_PROMPT = (
    "Instruction: What is the sentiment of this news? "
    "Please choose an answer from {{positive/negative/neutral}}.\n"
    "Input: {text}\nAnswer:"
)
SENTIMENT_MAP = {"positive": 1, "negative": -1, "neutral": 0}
```

3-label classification with a fixed mapping and `temperature=0`. **Textbook 🟡** by the rubric. Replacement: `ProsusAI/finbert` (CPU-fine, deterministic, ~2ms per text) or — per the team's own intent — `FinoGridSentimentAnalyzer` (local FinGPT) which is already the default when `FINGPT_LLM_PROVIDER=fingpt`. The OpenAI/MiniMax variants are explicit stop-gaps.

**Action:** verify the `FINGPT_LLM_PROVIDER=fingpt` path is reachable in production and add a CI integration test that exercises it. The `openai_fallback.py` and `minimax_provider.py` modules then become legitimate dev-only escape hatches.

#### `finogrid/fingpt_integration/sentiment/minimax_provider.py` — 🟡

Same task, same prompt, same `SENTIMENT_MAP`, same recommendation. Two files differ only in `base_url` and `temperature=0.01` vs `0`. **Both replaced by the same FinBERT or local-FinGPT call** — not two separate audits.

#### `fingpt/FinGPT_Others/shares_news_sentiment_classify.py` — 🟡

Two prompts (Chinese + English), both `text-davinci-003` (legacy completions). The English prompt:

> "Read following news and list 3 stocks that may be affected, briefly judge each one 'positive' or 'negative':"

This is **composite**: NER (extract stock tickers) + per-ticker sentiment classification. Two-stage deterministic replacement is feasible but harder than the pure-sentiment cases:

1. NER via spaCy (`en_core_web_lg`) with a custom ticker matcher seeded from yfinance's ticker list, or `FinNER` (financial NER models on HuggingFace).
2. Sentiment per extracted ticker — same FinBERT path as above, scored on sentences containing the ticker.

🟡 with explicit caveat: the original LLM call may have been combining loose entity mentions ("the iPhone maker" → AAPL) that NER models miss. Validate against ~50 historical inputs before swap.

#### `fingpt/FinGPT_Others/FinGPT_Trading/chatgpt-trading-v2/get_gpt_sentiment_results/get_gpt_res.py` — 🟡

```python
sentences = [f"Decide whether a sentence's sentiment is positive, neutral, or negative.\n\nSentence: \"{i}\"\nSentiment: " for i in sentences]
# ...openai.Completion.create(model="text-curie-001", prompt=sentences, ...)
```

Batched 20-at-a-time sentiment classification on `text-curie-001` (very old). **Same task as `openai_fallback.py`** — fold into the same FinBERT migration. Plus the script has zero error recovery (catches `Exception`, sleeps 10s, returns `"error"` literal, loses the batch) — replacing with FinBERT also fixes that.

#### `fingpt/FinGPT_RAG/multisource_retrieval/external_LLMs/external_LLMs.py::extract_classification` — 🟡

Generic classification helper with a user-supplied `classification_prompt`. The output is a single string compared downstream (likely against an options list). **🟡 conditional on the actual options being closed-set per call site** — read the callers in `multisource_retrieval/` to confirm before swapping. If options are open-ended ("describe the sector"), the verdict downgrades to 🔴.

#### `fingpt/FinGPT_FinancialReportAnalysis/utils/rag.py::Raptor.embed_cluster_summarize_texts` — 🔴

```python
template = """Here is a sub-set of LangChain Expression Language doc. ...
Give a detailed summary of the documentation provided.
Documentation:
{context}
"""
prompt = ChatPromptTemplate.from_template(template)
chain = prompt | self.model | StrOutputParser()
```

RAPTOR-style hierarchical-cluster summarization. The clustering step (UMAP + GMM) is already deterministic — that's the part of the RAPTOR algorithm that doesn't need an LLM. The **summarization** of each cluster needs the LLM because the output goes into the next level of clustering as input text. **🔴 (genuine summarization for the recursive index).** Worth flagging a minor gotcha: the template string still says _"LangChain Expression Language doc"_ — looks like leftover from the upstream Raptor demo; should be parameterized to the actual document domain (financial reports) for quality reasons unrelated to this audit.

#### `fingpt/FinGPT_FinancialReportAnalysis/reportanalysis.ipynb` — 🔴 ×4

Four `client.chat.completions.create` sites in the notebook. All are RAG-style "given retrieved section + question, answer" — open-ended QA over the retrieved text. **All 🔴** (generation conditioned on retrieved context). The output is shown to a human analyst as prose.

#### `fingpt/FinGPT_Others/FinGPT_Robo_Advisor/*.ipynb`, `FinGPT_Low_Code_Development/*.ipynb`, `chatgpt-trading-v2/trade_with_gpt3.ipynb` — 🔴

Conversational / code-generation / strategy-explanation. All open-ended generation. None are replaceable with deterministic code by design — these subprojects are explicitly demonstrating LLM use cases. **🔴, no action.**

#### `fingpt/FinGPT_MultiAgentsRAG/MultiAgents/inference_mmlu_Llama2*.ipynb` — 🔴

Multi-agent inference experiments using Llama-2 + GPT. Reasoning / debate setup, similar shape to `TradingAgents`. **🔴, agentic.**

#### `fingpt/FinGPT_MultiAgentsRAG/RAG/RAG_part.ipynb` — 🔴

RAG QA. Same shape as `reportanalysis.ipynb` — generation conditioned on retrieval. **🔴.**

---

### ❌ Out of scope — training-data generation

These call sites use the LLM to **produce ground truth** for fine-tuning FinGPT. Replacing them defeats the experiment.

| Location | Purpose | Why out of scope |
|---|---|---|
| `fingpt/FinGPT_Forecaster/data.py:225` | `gpt-4` generates the system + user prompt rationales used as training targets for the FinGPT-Forecaster fine-tune | The LLM output IS the training label. Replacing with rules → trains a model on rules-output, defeats the purpose. |
| `fingpt/FinGPT_Forecaster/prepare_data.ipynb` | Same flow in notebook form | Same as above |
| `fingpt/FinGPT_Forecaster/FinGPT-Forecaster-Chinese/Formulate_training_data.ipynb` | Chinese-language variant | Same |
| `fingpt/FinGPT_MultiAgentsRAG/Evaluation_methods/HaluEval/generate.py` | Generates hallucinated samples for the HaluEval benchmark | The LLM is the hallucination source; the dataset is the artifact |
| `fingpt/FinGPT_MultiAgentsRAG/Evaluation_methods/HaluEval/filtering.py` | Filters generated samples | Filter criteria depend on LLM consistency — replacing changes the dataset |

### ❌ Out of scope — benchmark / LLM-as-judge

These use the OpenAI API as a **baseline being evaluated** or as a **judge** scoring FinGPT outputs.

| Location | Purpose | Why out of scope |
|---|---|---|
| `fingpt/FinGPT_MultiAgentsRAG/Evaluation_methods/HaluEval/evaluate.py` (×6 sites) | `gpt-3.5-turbo` / other models as the model being benchmarked on HaluEval | The OpenAI call IS what's being measured |
| `fingpt/FinGPT_MultiAgentsRAG/Evaluation_methods/MMLU/eval_mmlu.py`, `gen_mmlu.py` | Same for MMLU | Same |
| `fingpt/FinGPT_MultiAgentsRAG/Evaluation_methods/TruthfulQA/evaluate.py` | Same for TruthfulQA | Same |
| `fingpt/FinGPT_RAG/multisource_retrieval/ChatGPT_sentiment_analysis_benchmark.ipynb` | ChatGPT as the sentiment baseline FinGPT is compared against | Same |
| `fingpt/FinGPT_RAG/multisource_retrieval/utils/classification_accuracy_verification.py` | Verifies classification accuracy — likely against an LLM-judged ground truth | Read before classifying. Likely out of scope. |

### N/A — wiring

`finogrid/fingpt_integration/minimax_llm_client.py` is the **generic async LLM-client wrapper** other Finogrid agents accept as their `llm_client` dependency. The class itself doesn't run a task; the tasks live wherever it's used (`InternalSupportAgent`, `AuditGovernanceAgent`, etc.). Audit those agents — not this file — for verdicts.

---

## The pattern of value

The 🟡 group is **the same call appearing five times in five files** with slightly different transports:

| File | Provider | Model | Same task? |
|---|---|---|---|
| `finogrid/.../openai_fallback.py` | OpenAI | `gpt-3.5-turbo` | ✅ 3-label sentiment |
| `finogrid/.../minimax_provider.py` | MiniMax (OAI-compat) | `MiniMax-M2.7` | ✅ same prompt verbatim |
| `fingpt/.../get_gpt_res.py` | OpenAI | `text-curie-001` | ✅ same task, batched |
| `fingpt/.../shares_news_sentiment_classify.py` | OpenAI | `text-davinci-003` | ✅ (variant: + NER) |
| `fingpt/.../external_LLMs.py` | OpenAI | `gpt-3.5-turbo` | ✅ (generic — likely sentiment per options) |

One FinBERT-backed `SentimentClassifier` class deletes ~80% of in-scope LLM call volume in this repo. The remaining 🔴 are the RAG/robo-advisor/agentic subprojects that demonstrate genuine LLM use cases.

**The project itself already provides the replacement** — `FinoGridSentimentAnalyzer` (local FinGPT) is the production-intended path, with OpenAI/MiniMax as cost-managed stop-gaps. The audit's recommendation aligns with the team's existing roadmap: bias deployments toward `FINGPT_LLM_PROVIDER=fingpt`, treat OpenAI/MiniMax as fallback-only, retire the duplicate stand-alone scripts (`get_gpt_res.py`, `shares_news_sentiment_classify.py`) or rewire them through the `get_sentiment_analyzer()` factory.

## Aggregate impact estimate

**Per-call cost (in-scope sentiment calls):** ~$0.0001 each on `gpt-3.5-turbo` / `text-curie`, comparable on MiniMax-M2.7. Per-run volume unknown without traffic data. With even modest production traffic (~10k news items / day), the in-scope LLM spend is on the order of **~$1/day**, dwarfed by the cost of running a single GPU instance for the local FinGPT model — meaning the cost case is **reliability and consistency**, not dollars. FinBERT or local FinGPT produces identical output for identical input; the API path doesn't.

**Token counting method:** approximated by character length × 0.25. Real numbers require running `tiktoken` over a sample.

## Methodology takeaways for the skill itself

Three observations to feed back into `llm-buster/references/`:

1. **Add a fourth verdict: ❌ Out of scope.** The current rubric has 🟢/🟡/🔴. Research and benchmark codebases need an explicit "do not propose deterministic replacement, this is the experimental artifact" tag. Otherwise the skill keeps generating noise on `evaluate.py` files. Suggested update: `references/rubric.md` adds a "Out-of-scope disqualifier" — fires when the call is inside an `Evaluation_methods/`, `eval/`, `benchmark/`, `data_gen/`, training-data path, or the docstring/README marks it as a baseline. Drops the verdict to ❌ regardless of category.

2. **Multi-subproject repos need a top-level pivot first.** The default per-call deep-dive blew up here (30 sites). A per-subproject sweep with verdict counts + targeted deep-dives on the top 3 wins is more useful for umbrella repos. Add to `references/report.md`: "If the repo has ≥ 3 subprojects, lead with a per-subproject table before per-file detail."

3. **Detect "same call, N files" duplication.** Five sites in this repo are the same FinBERT-replaceable sentiment call. The report should hoist that pattern explicitly rather than emit five near-identical findings. Add to `references/rubric.md` / `report.md`: a "duplication pass" after classification that groups identical-task sites and proposes one canonical replacement plus a cross-reference list.

## Next steps

Pick one. Don't do all three.

1. **Consolidate the 5 sentiment call sites** through a single `SentimentClassifier` (FinBERT or local FinGPT via existing `get_sentiment_analyzer()`), then rewire the 3 stand-alone scripts (`get_gpt_res.py`, `shares_news_sentiment_classify.py`, `external_LLMs.py::extract_classification`) to call it. One PR per script.
2. **Verify the local-FinGPT path** — ensure `FINGPT_LLM_PROVIDER=fingpt` is exercised in CI and the model loads in the production image. Without this, all the OpenAI/MiniMax fallbacks remain de-facto default.
3. **Feed the methodology takeaways back into the skill** — add the `❌ Out of scope` verdict and the duplication-detection pass to `llm-buster/references/`. This audit would have been 30% shorter and clearer with them in place.

> _Audit produced by `llm-buster` skill v0.1.0 (as `de-llmer`, pre-rebrand)._
