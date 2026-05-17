# llm-buster-skill

> An Agent Skill that audits a codebase for LLM API calls and reports which ones can be replaced with deterministic logic — without rewriting code by default.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Agent-Skill-7c3aed)](SKILL.md)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-8b5cf6)](SKILL.md)

## What this gives you

Drop this skill into your agent. When you ask it to "audit LLM usage", "find LLM calls", "do I really need a model for this", "cut LLM costs", or "de-LLM this project", the agent:

1. Inventories every OpenAI / Anthropic call site (Python SDK, TS/JS SDK, and raw HTTP).
2. Reads each prompt and downstream parse to figure out what the call is actually doing.
3. Classifies the task (extraction, classification, routing, validation, normalization, summarization, rephrasing, generation, reasoning, agentic).
4. Applies a verdict rubric with disqualifiers and produces 🟢 / 🟡 / 🔴 calls.
5. Emits a Markdown audit report with concrete deterministic replacements for 🟢 / 🟡, risks/edge-cases, and a per-call cost estimate.

**It does not rewrite code by default.** Replacement is a separate step the user explicitly opts into after reading the report.

## What a report looks like

Three findings from a typical small repo, one of each verdict. The skill emits Markdown; rendered inline here:

> **🟢 — `services/intent.py:42`**
>
> Task: classification (3 fixed labels: `refund` / `shipping` / `other`). Prompt is a system message listing the labels with 4 hardcoded few-shot examples; output is parsed with `label = resp.choices[0].message.content.strip().lower()`.
>
> **Replacement:** `rapidfuzz.process.extractOne(text, LABELS, scorer=fuzz.token_set_ratio)` with a 60-score floor, fallback to `"other"`. ~15 lines.
>
> **Risks:** loses paraphrases the few-shots didn't cover ("money back" → `refund`). Mitigate with an alias dict or a week of shadow-mode against the live LLM.
>
> **Per-call cost:** ~280 in + ~3 out tokens on `gpt-4o-mini` ≈ \$0.00005. `× requests/month = $TBD`.

> **🟡 — `pipelines/extract_invoice.py:88`**
>
> Task: extraction (8 named fields from supplier invoices, JSON output schema). Input is OCR-derived text — mostly clean for the 12 known supplier templates, messy for the long tail.
>
> **Recommendation:** rule-based extraction (regex + `pydantic`) for the known-template subset, LLM fallback for the rest. Shadow-mode the rule path for two weeks before flipping.
>
> **Risks:** rule paths are brittle to template revisions; per-supplier regex needs an owner. Don't deploy without a per-supplier coverage metric in the logs.

> **🔴 — `agent/summarize_thread.py:17`**
>
> Task: summarization of a Slack thread for a daily digest. Open-ended natural-language output, shown verbatim to a human reader, multi-turn context.
>
> **Verdict:** keep. Generation against free-form prose — no deterministic substitute. (`stream=True` also confirms UI-streamed output.)

Closing line of every audit: *"Want me to apply the 🟢 replacements?"*

For full-length real-world audits across 5 open-source projects (FinGPT, TradingAgents, FinSight, gpt-investor, CyteType), see [`examples/`](examples/).

## Scope

- **In scope:** OpenAI and Anthropic, Python and TS/JS, SDK and raw HTTP.
- **In scope (detection only):** wrappers that import the above — LangChain, LiteLLM, instructor, LlamaIndex, Haystack.
- **Out of scope (for now):** Google / Gemini, local LLMs (Ollama, vLLM), Cohere, Mistral, Bedrock-via-non-Anthropic, custom internal LLM gateways.

Future versions may widen the SDK list; the methodology generalizes.

## Install

### Claude Code, user-level (every project)

```bash
mkdir -p "$HOME/.claude/skills"
git clone https://github.com/metravod/llm-buster-skill.git \
  "$HOME/.claude/skills/llm-buster"
```

### Claude Code, project-level (one project)

```bash
mkdir -p .claude/skills
git clone https://github.com/metravod/llm-buster-skill.git \
  .claude/skills/llm-buster
```

### Codex

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
git clone https://github.com/metravod/llm-buster-skill.git \
  "${CODEX_HOME:-$HOME/.codex}/skills/llm-buster"
```

### Or paste this to your agent

```text
Install the llm-buster-skill skill for me:

1. Clone https://github.com/metravod/llm-buster-skill into the skill directory my
   agent reads on this machine (e.g. ~/.claude/skills/ for Claude Code, or
   ~/.codex/skills/ for Codex).
2. Verify that SKILL.md and the references/ directory are present.
3. Confirm the install path when done.
```

## Layout

```text
SKILL.md                            # activation rules + per-call analysis loop + non-negotiables
references/
  detection.md                      # SDK + HTTP + wrapper patterns (Python, TS/JS, AST)
  taxonomy.md                       # the 10 task categories with verdict biases
  rubric.md                         # 🟢/🟡/🔴 rules + disqualifiers + promoters + decision algorithm
  replacements.md                   # code recipes per category (regex, pydantic, sklearn, etc.)
  report.md                         # output template with style rules
```

Progressive disclosure: `SKILL.md` is loaded by the agent runtime when the skill activates; references are loaded only when relevant to the user's task.

## Usage

Once installed, ask the agent things like:

- "Audit LLM usage in this repo."
- "Do I actually need an LLM for `parser.py:42`?"
- "Cut LLM costs in this project."
- "Find every OpenAI call and tell me which are overkill."

The skill activates automatically on these phrases and on grep hits for the SDKs in the working directory.

After the report, the user can ask:

- "Apply the 🟢 replacements." → the agent opens a PR with one diff per call site.
- "Set up shadow rollout for the 🟡 calls." → the agent wires the side-by-side comparison harness.
- "Re-audit after I add real traffic numbers." → the agent updates only the impact section.

## What this is _not_

- **Not a refactoring tool.** Default is audit + report. Replacement is explicit, opt-in, one site at a time.
- **Not a cost-savings calculator.** Token estimates are honest; monthly dollars require user-supplied traffic.
- **Not a model-quality reviewer.** The skill doesn't critique prompts for clarity, jailbreak surface, or output quality — only whether the call should exist.
- **Not a benchmark.** The shadow-rollout pattern in `replacements.md` is the right way to validate a deterministic replacement empirically; this skill describes it but doesn't run it.

## Versioning

The skill version (in `SKILL.md` frontmatter) is independent of any external library. Bump on additions to the taxonomy, rubric, or supported SDKs.

## Contributing

Issues and PRs welcome. Two ground rules:

1. Every replacement recipe must run on its named library/version with no provider-side dependencies. No "AI-powered" libraries.
2. Don't grow the skill body for completeness — keep `SKILL.md` tight and push detail into `references/`.

## License

MIT — see [LICENSE](LICENSE).
