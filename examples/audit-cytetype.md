# LLM call audit — NygenAnalytics/CyteType

**Date:** 2026-05-17
**Scope:** OpenAI + Anthropic SDKs, raw HTTP, LangChain wrappers (skill's standard surface)
**Inventory method:** ripgrep over `cytetype/` excluding `tests/` and `docs/`
**Calls found:** **0 direct LLM call sites in the client.**

## The audit is fundamentally different here

CyteType (cell-type annotation for single-cell RNA-seq) is a **thin Python client to a remote agentic service** owned by NygenAnalytics. The standard inventory grep returns zero hits because the entire LLM stack — prompts, agents, orchestration, model dispatch — lives behind an HTTP endpoint the client never sees. The package's job is:

1. **Preprocessing** (`cytetype/preprocessing/`) — pure numpy/scipy/scanpy statistics: marker-gene detection, group aggregation, subsampling, validation. No LLM, by design.
2. **Payload assembly** (`cytetype/core/payload.py`) — build a `pydantic` `AnnotateRequest` and validate.
3. **HTTP transport** (`cytetype/api/`) — POST to `/annotate`, upload large files via chunked uploads to `/upload`, poll `/status/{job_id}`, fetch `/results/{job_id}`.

What comes back is a cell-type annotation per cluster with an "evidence trail" (per the README). The LLM agents that produce that annotation run server-side.

**This is a verdict mode the current rubric doesn't have.** Proposing it explicitly below.

## Proposed verdict: 🔵 Delegated

**🔵 Delegated** — the call site is a thin HTTP transport to a service whose LLM internals the user does not own. The standard rubric (🟢/🟡/🔴/❌) doesn't apply because there are no prompts to read. The audit instead:

- Describes the client/server split and where work actually happens.
- Enumerates the documented agent surface (from public schemas / docs).
- Suggests architecture-level questions ("do you need agentic LLM annotation at all, or would deterministic marker-overlap tools serve?").
- Estimates the user's LLM bill _by proxy_ — by reading the `LLMModelConfig` surface and noting which provider list is supported.

CyteType is **🔵 in full**.

## What the schemas tell us

The package exposes a `LLMModelConfig` and a typed `AgentType` enum (in `cytetype/api/schemas.py:26-30`):

```python
AgentType: TypeAlias = Literal[
    "contextualizer", "annotator", "reviewer", "summarizer", "clinician", "chat"
]
LLMProvider: TypeAlias = Literal[
    "anthropic", "bedrock", "fireworks", "google", "groq", "huggingface",
    "mistral", "openai", "openrouter", "vertex", "xai",
]
```

The user can route different agents to different providers via `LLMModelConfig.targetAgents`. The user-supplied `apiKey` (or AWS credentials) means **the user pays for the LLM calls**, not NygenAnalytics. The server is the orchestrator; the user is the model-credit holder.

### Educated guesses on each agent (not verifiable from this repo)

| Agent | What the name suggests | Inferred verdict if it were in-repo |
|---|---|---|
| `contextualizer` | Reads `studyInfo` + `infoTags`, distils biological context for downstream agents | 🔴 (generation / reasoning) |
| `annotator` | Per cluster: marker gene list → cell-type label | **🟡 (the one to scrutinize)** — see below |
| `reviewer` | Second opinion on annotator's call, likely consensus or disagreement detection | 🔴 (reasoning over annotator prose) |
| `summarizer` | Human-readable narrative for the report | 🔴 (open-ended generation) |
| `clinician` | Clinical / disease context layer (the README mentions disease-context use cases) | 🔴 (reasoning + generation) |
| `chat` | Interactive Q&A with the user about results | 🔴 (conversational) |

The interesting one is **`annotator`**. Deterministic marker-overlap tooling already exists in the scRNA-seq ecosystem — **ScType**, **SCSA**, **CellID**, **Cellassign**, **decoupler-py**, **scCATCH** — all of which score cluster marker lists against curated cell-type signature databases (CellMarker, PanglaoDB) without an LLM. Many published benchmarks run them as baselines. CyteType's value is presumably the multi-agent consensus + Cell Ontology mapping + evidence trail, not the labelling step in isolation.

**The audit cannot verify how good NygenAnalytics' `annotator` is vs ScType-style baselines** without server access. But the question "do you actually need the LLM for the annotation step" is the right one to put in front of any CyteType user — especially since they're paying the bill.

## What you CAN audit from the client side

Five practical recommendations, none requiring server access:

### 1. Route the cheap agents to cheap providers

`LLMModelConfig.targetAgents` lets the user pin a specific agent to a specific provider/model. The `summarizer` and `chat` agents produce free-form prose for the human reader — both can be on the cheapest provider in the list (Groq + a small Llama, or OpenRouter routed to a budget model). Reserve the strong (and expensive) models for `annotator`, `reviewer`, and `clinician` where reasoning quality matters more.

This isn't "de-LLM" — it's tier-routing within the agent set, which the API explicitly supports. No code change in CyteType needed.

### 2. Skip the `chat` agent if you don't use the chat surface

If the workflow is "annotate → export → done" with no interactive follow-up, omit `chat` from `targetAgents` entirely (or use the absence of `LLMModelConfig` for it to be skipped server-side — check the docs for the exact dispatch rule). One agent's worth of latency and tokens, gone.

### 3. Consider whether you need agentic annotation at all

If your project's requirements are:
- "Get a cell-type label per cluster" → ScType / SCSA / CellID / decoupler-py do this deterministically. No LLM cost. No evidence trail.
- "Get a label + a reproducible evidence trail + Cell Ontology mapping" → CyteType is built for this. Stay.
- "Get a label with disease/clinical contextualization" → likely CyteType. Stay.

This is an **architecture-level decision**, not a code-level one. Worth raising before adopting the tool, not after.

### 4. Cache results aggressively

`cytetype/api/client.py` has `submit_annotation_job` + `wait_for_completion`. There's no client-side result cache — re-running annotation for an unchanged `AnnotateRequest` re-submits to the server and re-burns tokens. Hash the payload (input_data + llm_configs) and short-circuit on hit. Bug-fix-level change, not a de-LLM swap, but lives in the same engineering motion.

### 5. Validate `studyInfo` is not a free-form essay

`InputData.studyInfo` is a `str` with no length limit. Whatever the user types is presumably fed verbatim to the `contextualizer`. A 5000-character study description blows up every downstream agent's prompt. Add a client-side length warning + truncation suggestion. Small token discipline, big over time.

## Architectural observation worth flagging

The whole stack relies on the user **bringing their own LLM API key** to the server. That key is sent inside `LLMModelConfig.apiKey` to `/annotate`. The audit can't see what TLS / secret-handling NygenAnalytics applies — but from the client side, the user is shipping a long-lived production API key over the wire to a third-party service per request.

Recommend:
- **Use a provider key scoped to a budget/spend cap**, not the main production key.
- **Rotate the key on a schedule** rather than rely on the third-party's security posture.
- **Prefer AWS credentials (Bedrock) over `apiKey` when available** — `LLMModelConfig.check_aws_credentials` shows the AWS path is first-class, and STS-issued credentials with a short TTL are safer than long-lived OpenAI keys.

This isn't a de-LLM finding — it's a credential-hygiene finding the de-LLM grep happened to surface. Worth saying once.

## Verdict summary

| Verdict | Count | Notes |
|---|---|---|
| 🟢 Replace now | 0 | n/a — no client-side calls |
| 🟡 Worth trying | 0 | n/a |
| 🔴 Keep | 0 | n/a |
| ❌ Out of scope | 0 | n/a |
| **🔵 Delegated** | **1 (entire package)** | All LLM work runs server-side at NygenAnalytics. Audit pivots to architecture + tier-routing recommendations. |

## Methodology takeaway — patch the skill

This audit surfaces a **fifth verdict the rubric needs**: 🔵 Delegated. It fires when:

- The repo is a client to a remote service whose LLM internals are not in this codebase.
- The user supplies an LLM API key (or equivalent credentials) to the service, but doesn't author the prompts.
- The agentic surface is documented via a `LLMConfig` / `AgentType` schema or similar contract.

When 🔵 fires, the audit produces a **delegated-mode report**: architecture description, schema enumeration, inferred per-agent verdicts (low confidence by definition), tier-routing recommendations, credential-handling notes. The standard per-call template doesn't apply.

> _Audit produced by `llm-buster` skill v0.1.0 (as `de-llmer`, pre-rebrand; with proposed 🔵 verdict extension)._
