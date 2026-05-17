# Report format

The audit's deliverable. Emit as Markdown, paste into a PR description or save under `audits/llm-buster-<date>.md`. The structure is fixed so the user can diff successive audits as the codebase changes.

## Sections — in order

1. Header
2. Verdict summary table (include ❌ / 🔵 rows if any fired — see rubric.md)
3. **Per-subproject pivot** (only if repo has ≥ 3 subprojects — see "Umbrella-repo mode" below)
4. Per-call findings (grouped 🟢 → 🟡 → 🔴, then ❌ as a single table at the end)
5. **Duplication pass** (if ≥ 2 sites share the same task — see "Duplication pass" below)
6. Aggregate impact estimate
7. Next steps

If the project has > 15 call sites, replace section 4 with a per-file summary and offer to deep-dive a subset on request.

If section 4 is empty because the entire repo is 🔵 Delegated, replace it with the delegated-mode shape from `rubric.md` § 🔵.

### Umbrella-repo mode

When the repo has ≥ 3 subprojects (umbrella research repo, monorepo with N services, plugin packages), lead with a **per-subproject table** before any per-file detail:

| Subproject | Calls | 🟢 | 🟡 | 🔴 | ❌ | Verdict-mix summary |
|---|---|---|---|---|---|---|
| FinGPT_Sentiment_Analysis_v3 | 4 | 0 | 3 | 1 | 0 | 3-label sentiment ×3 (see Duplication pass), 1 RAG |
| FinGPT_Forecaster | 3 | 0 | 0 | 0 | 3 | all ❌ (training-data generation) |
| ... | | | | | | |

Then drill into only the top 2–3 subprojects with actionable 🟢/🟡 findings. Subprojects with all-❌ or all-🔴 get the table row only — no per-file blocks.

This stops a 30-call audit from producing 30 per-call blocks that no human will read.

### Duplication pass

After classifying every call, scan for sites that perform the **same task** (same prompt shape, same downstream parse). When ≥ 2 sites match, emit one consolidated finding with a cross-reference table instead of N near-identical blocks:

> **🟡 ×5 — sentiment classification (positive/negative/neutral)**
>
> | File | Provider | Model | Notes |
> |---|---|---|---|
> | `finogrid/.../openai_fallback.py` | OpenAI | gpt-3.5-turbo | canonical |
> | `finogrid/.../minimax_provider.py` | MiniMax | MiniMax-M2.7 | same prompt verbatim |
> | `fingpt/.../get_gpt_res.py` | OpenAI | text-curie-001 | same task, batched |
> | `fingpt/.../shares_news_sentiment_classify.py` | OpenAI | text-davinci-003 | variant: + NER |
> | `fingpt/.../external_LLMs.py::extract_classification` | OpenAI | gpt-3.5-turbo | generic, likely same |
>
> One FinBERT-backed `SentimentClassifier` collapses all five. See proposed replacement below.

Same-task detection heuristics: prompt body ≥ 60% string-overlap; same `response_format`; same downstream parse pattern (`int()`, dict lookup, label-in-set). If unsure, treat as distinct — wrong dedup is worse than wrong split.

---

## Template

```markdown
# LLM call audit — <project name>

**Date:** YYYY-MM-DD
**Scope:** OpenAI + Anthropic SDKs (Python + TS/JS) and raw HTTP to api.openai.com / api.anthropic.com
**Inventory method:** ripgrep over `<root>`, excluding `node_modules`, `.venv`, `dist`, `build`
**Calls found:** N across M files

## Verdict summary

| Verdict | Count | Models involved | Est. per-call cost | Notes |
|---|---|---|---|---|
| 🟢 Replace now | A | gpt-4o-mini, claude-haiku-4-5 | $0.0003–$0.002 | drop-in deterministic |
| 🟡 Worth trying | B | gpt-4o-mini | $0.001–$0.004 | needs shadow rollout |
| 🔴 Keep | C | claude-sonnet-4-6, gpt-4o | $0.01–$0.05 | genuine LLM territory |
| **Total** | **N** | | | |

## 🟢 Replace now

### 🟢 <file>:<line> — <one-line summary>

**Category:** <one of taxonomy.md categories>
**Confidence:** high / medium / low
**Model:** `gpt-4o-mini` (or whatever)

**Current call**

```python
# verbatim snippet — SDK call + the prompt template
resp = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "Extract sender_email and amount as JSON."},
        {"role": "user", "content": email_body},
    ],
    response_format={"type": "json_object"},
)
data = json.loads(resp.choices[0].message.content)
```

**Why it's 🟢:** the prompt names two fixed fields, the input is an RFC 5322 email, and the downstream is a strict `json.loads`. No promoters / disqualifiers applied (or: P1 strict JSON schema applied, no disqualifiers).

**Proposed replacement**

```python
import re
from email import message_from_string
from email.utils import parseaddr

AMOUNT_RE = re.compile(r"\$\s?([\d,]+\.?\d*)")

def parse_email(raw: str) -> dict | None:
    msg = message_from_string(raw)
    _, sender_email = parseaddr(msg["From"] or "")
    body = msg.get_payload(decode=True)
    if isinstance(body, bytes):
        body = body.decode(msg.get_content_charset() or "utf-8", errors="replace")
    m = AMOUNT_RE.search(body or "")
    if not (sender_email and m):
        return None
    return {"sender_email": sender_email, "amount": float(m.group(1).replace(",", ""))}
```

**Risks / edge cases**

- HTML-only emails (no plaintext part) — handle by walking `msg.walk()`.
- Non-USD currencies — current regex matches `$` only. Add per-currency patterns if relevant.
- Multiple amounts in body — current pattern returns first match; LLM may have been picking "the right one" by context. Confirm by sampling 50 historical inputs.

**Per-call cost (current)**

- Input ~350 tokens × $0.15/1M = $0.0000525
- Output ~40 tokens × $0.60/1M = $0.000024
- **Total: ~$0.00008/call** × <requests/month, TBD> = $TBD/month

(Multiply by actual traffic to get the monthly figure. Latency: ~600–1000ms → <1ms.)

---

(repeat per 🟢 finding)

## 🟡 Worth trying

### 🟡 <file>:<line> — <one-line summary>

**Category:** classification
**Confidence:** medium
**Model:** `gpt-4o-mini`

**Current call** — (snippet)

**Why it's 🟡:** classification bias 🟡, no disqualifier (input is semi-structured ticket subject + first 200 chars of body). Labeled examples likely available from historical logs.

**Proposed approach**

1. **Pull labeled data**: query the last 3 months of LLM output to build `(text, label)` pairs.
2. **Train baseline**: `TfidfVectorizer(ngram_range=(1,2))` + `LogisticRegression`. Expect 85–92% agreement with LLM on a held-out 20% split.
3. **Ship shadow-mode**: run both, log disagreements (snippet in [replacements.md](replacements.md) "shadow rollout"), still return the LLM verdict to callers.
4. **Cutover threshold**: agreement ≥ 97% on rolling 7-day window. Add a confidence threshold (e.g. 0.7); fall back to LLM below threshold permanently.

**Risks / edge cases**

- New label introduced after model train → silent misclassification. Mitigate with periodic retrains.
- Drift in input style (new product launched, new support channel) → monitor agreement rate.

**Per-call cost (current)** — (same format as above)

---

(repeat per 🟡 finding)

## 🔴 Keep

Listed for completeness — these calls stay as-is.

| Location | Category | Why kept |
|---|---|---|
| `chat/handler.py:88` | generation (8) | streamed reply to end-user |
| `agents/orchestrator.py:42` | agentic (10) | tool loop over 6 tools |
| `summaries/digest.py:17` | summarization (6) | abstractive, varied input |

(One row per 🔴 call. No deep dive — full justification is the row's category + reason.)

## Aggregate impact estimate

Assuming the user provided monthly call volume (otherwise mark `TBD`):

| Verdict | Calls/month | Avg cost/call | Monthly spend | Avg latency | Latency removed |
|---|---|---|---|---|---|
| 🟢 (drop) | … | … | $… | …ms | …ms |
| 🟡 (after shadow) | … | … | $… | …ms | …ms |
| 🔴 (keep) | … | … | $… | — | — |
| **Replaceable total** | | | **$…/mo** | | |

**Token counting method:** `tiktoken` for OpenAI (`o200k_base` for 4o family, `cl100k_base` for 3.5/4), Anthropic's `count_tokens` API for Claude. If unavailable, mark estimates as approximate.

## Next steps

Pick one. Do not do all three at once.

1. **Apply 🟢 replacements** — I can open a PR with the deterministic code, tests, and a shim that flags any production behaviour delta. (Recommend doing this in one PR per 🟢 call site so you can roll back individually.)
2. **Set up 🟡 shadow rollout** — wire the side-by-side comparison harness, ship to staging, schedule a 1–2 week review.
3. **Sample real inputs first** — for low-confidence rows, pull 20–50 real inputs from prod logs and re-run the audit; some 🟡 will turn 🟢, some will turn 🔴.

> _Audit produced by `llm-buster` skill v1.0.0._
```

---

## Style rules

- **One block per finding.** Don't merge similar calls. Each call site is a separate decision unit.
- **Quote the actual code.** No paraphrased SDK calls. Copy the lines verbatim, including the prompt template; preserve f-strings and template literals.
- **Show the prompt.** The verdict can't be audited without seeing what the prompt asked for. If the prompt is > 30 lines, fold the middle with `# ... <K lines omitted> ...` but keep the first 10 and last 10.
- **Include risks even for 🟢.** Every replacement has edge cases. Risks-free is a lie and reviewers will catch it.
- **No invented numbers.** If you don't have the traffic, write `× $TBD requests/month`. If you don't have token counts, write "token count: estimate pending — run `tiktoken` on a real input".
- **Confidence label is mandatory.** `high` / `medium` / `low` after the colour. Drives the user's review depth.

## What to omit

- Don't repeat the rubric for each finding. One justification line is enough.
- Don't list all 8 disqualifiers; name only the ones that fired.
- Don't paste 200-line tool definitions; abbreviate with `# ... 6 more tools ...`.
- Don't include marketing-style impact framing ("save thousands per year!"). Plain numbers, no exclamation marks.
