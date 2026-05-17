# Verdict rubric — 🟢 / 🟡 / 🔴 / ❌ / 🔵

Apply this after classifying the task via [taxonomy.md](taxonomy.md). The category gives a default bias; this rubric refines it with the actual call's properties and disqualifiers.

## Verdict modes

Three colours for the standard case, plus two structural modes that short-circuit the colour rubric entirely:

- 🟢 / 🟡 / 🔴 — the standard verdicts for in-scope, locally-authored LLM calls. See "The three colours" below.
- **❌ Out of scope** — fires when the LLM call exists for benchmarking, training-data generation, or as a baseline being evaluated. Skips colour assignment. See "❌ Out of scope" below.
- **🔵 Delegated** — fires when the codebase is a thin client to a remote service whose LLM internals are not in this repo. Skips per-call rubric entirely. See "🔵 Delegated" below.

When ❌ or 🔵 fires, do **not** also assign 🟢/🟡/🔴. The structural verdicts replace the colour, not extend it.

## The three colours

### 🟢 Replace now

Replace with deterministic code. Effort low, risk low, savings real.

**All of these must hold:**

- Task category is **extraction (1)**, **routing (3)** with ≤ ~10 stable branches, **validation (4)**, or **normalization (5)** — _or_ classification (2) reducible to keyword/regex rules.
- Input has a recognizable structure: HTML, email, JSON, CSV, log lines, known templates, fixed-format identifiers.
- Output schema is fixed and named in the prompt (or via `tools=` / `response_model=` / `json_schema`).
- The deterministic replacement is < ~50 lines of code using a stdlib or well-known library.
- Failure mode of misparse is bounded: a missing field, a `None`, a raised exception — not silent semantic drift.

**Examples:**

- Pull `sender`, `subject`, `date` from an RFC 5322 email → `email.parser`.
- Decide whether a Stripe webhook is `invoice.paid` vs `invoice.failed` → look at `event["type"]`.
- Convert "tomorrow at 3pm Pacific" to ISO 8601 → `dateutil` + `pytz`.
- Validate that a string is a UUID → `uuid.UUID(s)` in try/except.
- Route GitHub webhook events to handlers by `X-GitHub-Event` header → dict dispatch.

### 🟡 Worth trying

Replaceable with classical ML / heuristics, but needs validation against real data before swap. Effort medium, risk medium, savings real.

**Any of these hold:**

- Task is **classification (2)** with ≥ ~100 labeled examples in hand (or harvestable from existing LLM logs).
- Task is **extraction (1)** from noisy input (OCR, free-form user-generated text) where regex will catch 80% but not 100%.
- Task is **routing (3)** with subtle intent boundaries (e.g. "complaint" vs "feedback").
- Task is **summarization (6)** that's actually extractive ("first 3 bullets", "TL;DR is the first sentence under the heading").

**Pattern:** propose a shadow-mode rollout: run deterministic + LLM in parallel for a sample, log disagreements, only swap after the agreement rate clears the user's threshold (typically ≥ 95%).

**Examples:**

- Classify support tickets into 8 categories → TF-IDF + LogReg trained on 6 months of historical LLM outputs.
- Extract product attributes from messy supplier feeds → regex per attribute + fuzzy match for known brand names.
- Detect customer intent (one of 12) on inbound chat → embedding similarity against 12 anchor messages with a confidence threshold; fall back to LLM below threshold.

### 🔴 Keep the LLM

Don't replace. Either the task genuinely needs the LLM, or replacement risk dwarfs savings.

**Any of these hold (any one is enough):**

- Task category is **generation (8)**, **reasoning (9)**, **agentic (10)**, or non-extractive **summarization (6)** / **rephrasing (7)**.
- Output is shown verbatim to a human end-user as natural language.
- Streaming response (`stream=True`) to a UI.
- Stateful multi-turn conversation (the `messages=[...]` array grows across calls).
- Prompt contains "explain", "answer", "respond to the user", "compose", "rewrite", "write", "create" — and the surrounding code feeds the output to a human.
- Category space is unbounded ("classify this article by topic" with no fixed taxonomy).
- The team cannot afford a 1–2% misclassification rate on the task (e.g. medical, legal, financial).

---

## Hard disqualifiers — any one forces 🔴

Some signals make replacement irresponsible regardless of how clean the task looks on paper. If **any** of these matches, the verdict is 🔴 and the decision algorithm short-circuits — do not count it against soft disqualifiers below.

| # | Hard disqualifier | Why it forces 🔴 |
|---|---|---|
| H1 | Output drives an irreversible action (send money, send email, delete record, file a ticket, post to social) | Misparse cost is unbounded. A regex bug here is a postmortem, not a regression. Recommend approval gating, not replacement. |
| H2 | The model is a fine-tuned variant (`gpt-4o-mini:ft:...`, `claude-*` via custom fine-tune) | The team has already curated a dataset and trained — deterministic code cannot reproduce the encoded behavior and there is no rollback path if it diverges. |
| H3 | The structured output is being reasoned over by a downstream LLM that consumed free-form prose upstream | A schema on the response doesn't change the fact that the model just reasoned over unstructured input. Replacing this with rules degrades the upstream reasoning silently. |

## Soft disqualifiers — apply to downgrade a verdict

After picking an initial colour from the category bias, walk this list. **Each match drops the verdict one level** (🟢 → 🟡, 🟡 → 🔴, 🔴 stays 🔴):

| # | Soft disqualifier | Why it drops the verdict |
|---|---|---|
| D1 | Input is free-form natural language (user chat, tweet, support text) | Regex/parsers handle the easy 80%; the LLM was absorbing the messy 20%. |
| D2 | Output is shown verbatim to a human end-user | Reduces tolerance for parser bugs to near zero. |
| D3 | There's no test suite around the LLM call | No way to verify the replacement matches; swap is high risk. |
| D4 | Token cost × traffic is small (per-call < $0.001 **and** < 1k/day) **and** the deterministic replacement needs > ~50 LOC or a non-stdlib dep | Savings don't justify engineering time. Cheap-and-trivial replacements (≤ ~20 LOC of stdlib) are still worth doing — this fires only when both the win and the lift are small. |
| D5 | The call is inside a `try/except` that already handles malformed responses | The team knows the LLM occasionally returns garbage and they catch it; the same handling needs to wrap a stricter parser too — adds friction. |
| D6 | The downstream code does a fuzzy `if "yes" in response.lower()` style check | The team has already conceded that strict parsing is hard — adding stricter regex won't help; the use case tolerates ambiguity. |

> Note: a previous version of this rubric included "the prompt has been edited 3+ times in git history" as a disqualifier. Removed — verifying it requires per-call `git log -p`, which isn't part of the analysis loop and was always skipped in practice.

## Promoters — apply to upgrade

Promoters lift the verdict toward 🟢 (replace). Each match cancels one soft disqualifier (or, if none, moves the bias up one level). **Promoters are additive up to a cap of 2** — `P1 + P2` together can cancel two disqualifiers, but a third promoter does nothing further.

| # | Promoter | Why it lifts the verdict |
|---|---|---|
| P1 | `response_format` uses a strict `json_schema` (not just `json_object`) | The team has already specified the contract — half the work is done. |
| P2 | `tools=[{...}]` with `tool_choice` forcing one function | Same — the schema is the function signature. |
| P3 | The prompt contains 3+ hardcoded few-shot examples of the same input → output mapping | Those examples _are_ the lookup table. Build a dict. |
| P4 | The model is `gpt-4o-mini` or `claude-haiku-*` and `temperature=0` | Team has already optimized for cost; this is a cheap-but-frequent call worth removing entirely. |
| P5 | The call is the bottleneck in a hot path (p95 > 500ms) | Latency win compounds savings. |

`json_object` alone is **not** P1 — without a schema the model can return any JSON shape. P1 requires `response_format={"type":"json_schema",...}` or equivalent.

---

## Decision algorithm

For each call:

1. **Hard check.** If any of H1–H3 matches → `Final = 🔴`. Stop. Note the hard disqualifier in the report so the reader sees _why_ the call survives.
2. **Bias** = default verdict from `taxonomy.md` matrix.
3. **Soft down** = count of matching soft disqualifiers (D1–D6).
4. **Up** = count of matching promoters (P1–P5), capped at 2.
5. **Net** = `max(Soft down − Up, 0)`.
6. **Final** = step the bias down by `Net`, clamped to {🟢, 🟡, 🔴}.

Worked examples:

> **Call A:** Extract 4 fields from an RFC 5322 email. `tools=` schema enforced. Tests exist. Volume ~10k/day on `gpt-4o-mini`.
> No hard hits. Bias 🟢 (extraction, structured input). Soft down 0. Up 2 (P2, P4). Net 0. **Final: 🟢.**

> **Call B:** Classify free-form user feedback into one of 6 sentiments. `temperature=0`, `gpt-4o-mini`. ~50k calls/day. No tests on this code path.
> No hard hits. Bias 🟡 (classification). Soft down 2 (D1 free-form, D3 no tests). Up 1 (P4). Net 1. **Final: 🔴.** If the user can produce 1k labeled examples and accepts shadow rollout, treat as 🟡 with explicit caveat.

> **Call C:** Route GitHub webhook to handler based on inferred intent. Output triggers a `POST` to a payments processor.
> H1 matches (irreversible action: payments). **Final: 🔴**, regardless of how clean the routing surface looks. Recommend approval gating, not replacement.

> **Call D:** Extract invoice fields from PDF text. Schema enforced via `tools=`. Hardcoded 5 few-shot examples in the prompt. `gpt-4o-mini`, `temperature=0`. Tests exist.
> No hard hits. Bias 🟢. Soft down 0. Up would be 3 (P2, P3, P4) but capped at 2. Net 0. **Final: 🟢.** The cap matters: it prevents stacking promoters into "negative" disqualifier territory.

---

## Confidence labels

In the report, after the colour, add a confidence in `{high, medium, low}`:

- **high** — bias clear, no disqualifiers, prompt + downstream parse + tests all agree on the spec.
- **medium** — bias clear but one disqualifier matched, or the prompt is dynamically assembled.
- **low** — couldn't read the full prompt, or couldn't determine downstream use, or the call is buried in an abstraction. **Recommend asking the user for a rendered prompt sample rather than committing to a verdict.**

Low-confidence 🟢 should be downgraded to 🟡 in the final report; low-confidence 🔴 stays 🔴 (safe default).

---

## ❌ Out of scope

Fires when the LLM call's existence is the experimental artifact — replacing it defeats the purpose of the code. Skips colour assignment entirely; the audit reports the site as ❌ with a one-line reason.

**Fires when any of these hold:**

- **Training-data generation.** GPT-4 (or similar strong model) produces labels, rationales, or prompts that are then used to fine-tune a smaller model. Path hints: `data_gen/`, `prepare_data/`, `formulate_training_data/`, `synthesize/`, `generate_dataset/`, files producing `.jsonl` / `.parquet` output. The LLM output **is** the training label — replacing with rules trains the model on rules.
- **Benchmark / evaluation harness.** OpenAI API used as a baseline being benchmarked (MMLU, HaluEval, TruthfulQA, custom domain benchmarks). Path hints: `Evaluation_methods/`, `eval/`, `benchmark/`, `benchmarks/`, `compare_*.py`. The LLM is what's being measured.
- **LLM-as-judge.** The LLM scores another system's outputs (often the user's own model). Detect by: prompt contains "rate", "score", "compare", "which of these is better", followed by a 0-N scale or A/B choice; output parsed into a numeric quality score saved to a results file.
- **Research/notebook artifact published alongside a paper.** Calls inside notebooks named after a paper section, an arXiv ID, or a benchmark dataset. Usually paired with a `cite this work` block in the README. The notebook _is_ the reproducibility evidence; replacing the LLM call changes the reproduced result.

**Report shape for ❌ sites:** list the location + one-line reason in a single table (don't open a per-call block). Group all ❌ sites at the end of the report so 🟢/🟡/🔴 stays visually clean.

**Do not propose a deterministic replacement for ❌.** It's not a "🔴 with caveats" — the underlying task is "use this LLM"; the audit has nothing to say.

---

## 🔵 Delegated

Fires when the audited codebase is a **thin client** to a remote service whose LLM internals are not in this repo. The standard rubric can't drill into prompts because the prompts live on the server. Skips per-call colour assignment; the audit pivots to architecture-level recommendations.

**Fires when all of these hold:**

- The grep for SDK / HTTP patterns returns **zero direct LLM calls** in the user's source tree (excluding tests and vendored code).
- The package's HTTP transport (`requests`, `httpx`, `aiohttp`) targets a **non-LLM-provider endpoint** — i.e. not `api.openai.com`, `api.anthropic.com`, etc. — but the project's README / pyproject describes an LLM-driven product.
- The package exposes an `LLMConfig`-shaped schema (provider name + API key + model name) that gets forwarded to the remote service — i.e. the **user supplies their own LLM credentials** to the third party.
- A typed `AgentType` / `agents` enumeration or equivalent contract documents what runs server-side.

**Report shape for 🔵:**

1. Architecture description — client/server split, what runs locally (preprocessing, payload assembly) vs remotely (the agents).
2. Enumerate the agent surface from public schemas / docs.
3. Inferred per-agent verdicts (low-confidence by definition; mark them as such).
4. **Architecture-level recommendations** instead of per-call replacements:
   - Tier-route cheap agents (summarization, chat) to cheaper providers if the API supports per-agent routing.
   - Skip agents whose output the workflow doesn't use.
   - Consider whether the LLM-driven approach is necessary at all vs domain-specific deterministic alternatives (cite them by name from the relevant ecosystem).
   - Cache results client-side if the API doesn't.
   - Flag credential-hygiene concerns (long-lived API keys sent over wire, prefer short-TTL alternatives like AWS STS when available).

**Do not** assign 🟢/🟡/🔴 alongside 🔵. The verdict is "the audit's standard frame doesn't apply here; here's what it can usefully say instead."

---

## Sub-task extraction — a 🟡 sub-mode

A common 🟡 pattern doesn't fit the "replace the whole call" framing. Sometimes the LLM call genuinely needs to stay (open-ended report writing, agentic synthesis), but a **sub-task inside the prompt** is deterministic and worth extracting into code that runs _before_ the LLM call:

- A 3000-token system prompt enumerates 11 options and asks the model to "select up to 8" → split out a deterministic picker, hand the LLM a pre-selected list.
- A prompt contains "compute the ratio of A to B and bucket as X/Y/Z" hardcoded heuristic → compute the bucket in code, feed the LLM the label.
- A prompt computes a verdict that downstream code parses out → compute the verdict in code from primary signals, feed the LLM only the prose-writing job.

This is **🟡 (sub-task extraction)** — the wrapping LLM call stays 🔴, but the audit notes the extractable sub-task as a distinct improvement. The wins are usually:

- **Reliability** — bucket labels / picks are now stable across re-runs of the same data.
- **Token cost** — the long instruction block in the prompt shrinks.
- **Reproducibility** — same input now produces same intermediate signals, which makes downstream debugging tractable.

When you mark a call 🟡 (sub-task extraction), the report block must say so explicitly — otherwise it reads as "the whole call is replaceable", which would be wrong.
