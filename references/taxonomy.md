# Task taxonomy — what is this LLM call actually doing?

Classify every LLM call into exactly one of these 10 categories. Pick the **dominant** task; note a secondary if it co-applies. The category determines which replacement recipe in [replacements.md](replacements.md) to reach for and biases the rubric in [rubric.md](rubric.md).

## 1. Structured extraction

**Shape:** "Given this {document/email/log/page}, return a JSON with fields X, Y, Z."

**Signals:**
- `response_format={"type":"json_object"}` or `json_schema`;
- `tools=[{...}]` with a forced `tool_choice`;
- prompt names a fixed list of fields;
- downstream `json.loads(...)`.

**Default verdict bias:** 🟢 if input is structured/semi-structured (HTML, email, CSV, log lines, known templates); 🟡 if input is free-form natural language or OCR.

**Common replacements:** regex, `pydantic` + handwritten parser, BeautifulSoup, `email`, `dateutil`, `pypdf`/`pdfplumber`, JSON Schema-driven parsers.

## 2. Classification

**Shape:** "Is this {spam/positive/category X}?" — output is one of a known finite set of labels.

**Signals:**
- prompt enumerates labels (`"reply with one of: refund, complaint, question"`);
- temperature low, output very short;
- downstream `if response == "refund": ...`.

**Default verdict bias:** 🟡 — replaceable with a classifier when ≥ ~100 labeled examples exist and label set is stable. 🟢 if a small keyword-list / rule set covers it (binary or very few labels with strong lexical signals).

**Common replacements:** keyword rules, sklearn `TfidfVectorizer + LogisticRegression`, fastText, a fine-tuned embedding classifier (sentence-transformers + LogReg on top).

## 3. Routing / intent detection

**Shape:** "Pick which of these N branches/tools/agents handles this request." Special case of classification where the labels are downstream code paths.

**Signals:**
- output is a key into a dispatch dict;
- N ≤ ~10;
- prompt describes each branch's purpose.

**Default verdict bias:** 🟢 if N is small and inputs have stable lexical cues; 🟡 if intents are subtle (e.g. "complaint" vs "feedback"). The cost of misrouting decides — silent-misroute = harder rubric.

**Common replacements:** keyword/regex router, intent classifier (same stack as 2), embedding similarity to N anchor sentences.

## 4. Validation

**Shape:** "Is this a valid {email/URL/phone/SSN/credit-card/SKU}?" or "does this text satisfy policy X?".

**Signals:**
- yes/no output;
- prompt enumerates rules ("must contain @, ...").

**Default verdict bias:** 🟢. Format validation is a solved problem. Policy validation may be 🟡 if rules are nuanced.

**Common replacements:** `email-validator`, `phonenumbers`, `validators` (Python), `card-validator`, RFC-compliant regex, JSON Schema, a rule engine (`durable_rules`).

## 5. Normalization / format conversion

**Shape:** "Convert this {date string / address / phone / currency / unit / case} into canonical form."

**Signals:**
- input is a single field;
- prompt describes target format ("ISO 8601", "E.164");
- output is a single field of identical type.

**Default verdict bias:** 🟢. Every format has a library.

**Common replacements:** `dateutil.parser`, `arrow`, `pendulum`, `phonenumbers`, `babel`, `pint` (units), `pycountry`, custom `str.translate`/`str.casefold` chains.

## 6. Summarization

**Shape:** "Summarize this {document/transcript/PR}."

**Signals:**
- prompt has "summarize", "TL;DR", "in N words";
- output is freeform prose;
- length-bound output (`max_tokens=200`).

**Default verdict bias:** 🔴 for human-readable summaries of varied content. 🟡 for **extractive** "summaries" where the prompt actually means "pull the first/last/N sentences" or "list the bullets" — then it's extraction in disguise.

**Common replacements (for the 🟡 cases):** sentence tokenizer + `textrank` (`summa`), TF-IDF top-K sentences, header/bullet extraction with `markdown`/`bs4`. For the 🔴 cases — leave alone.

## 7. Rephrasing / tone change

**Shape:** "Rewrite this in a more {formal/friendly/concise} tone." / "Translate to plain English."

**Signals:**
- input and output are the same content, different surface;
- prompt has "rewrite", "rephrase", "tone", "translate".

**Default verdict bias:** 🔴. Tone is exactly the kind of natural-language judgment LLMs are made for. Possible 🟡 only for **template-based** rephrasing (e.g. "replace pronoun X with Y").

**Common replacements (🟡 only):** templated string substitution, controlled-vocabulary glossary lookup.

## 8. Generation (open-ended)

**Shape:** "Write a {product description / email reply / blog post / SQL query / unit test}."

**Signals:**
- output streamed to UI or saved as final artifact;
- prompt asks for novel content;
- output length > ~100 tokens of prose.

**Default verdict bias:** 🔴. Don't touch.

**Common replacements:** none. If the user insists on deterministic, the only options are template engines (Jinja, Handlebars), and that's a product decision, not an optimization.

## 9. Reasoning / multi-step inference

**Shape:** "Given these facts, decide X." Often uses chain-of-thought, often goes back to the model multiple times.

**Signals:**
- `system` prompt instructs step-by-step thinking;
- multiple model calls in sequence to refine an answer;
- `messages=[...]` includes prior reasoning turns;
- thinking blocks / `extended_thinking`.

**Default verdict bias:** 🔴. This is the model's job.

**Common replacements:** none, unless the "reasoning" reduces to a finite decision table — in which case re-classify as routing (3) or validation (4).

## 10. Agentic / tool-loop

**Shape:** Model is given a set of tools and decides which to call in what order. Often inside a `while not done:` loop.

**Signals:**
- `tools=[...]` with multiple options;
- response inspected for `tool_calls`;
- loop continues until model emits final answer or stop condition;
- subagents, planning, scratchpads.

**Default verdict bias:** 🔴. Agentic flow with state and tool dispatch is the LLM's natural habitat.

**Common replacements:** none, unless the agent's tool sequence is actually fixed in practice — then it's a hardcoded pipeline (and the LLM was overkill). Check the event log / production traces; if 95% of runs follow the same tool order, propose unrolling into a non-agentic pipeline.

---

## How to pick when two fit

Order of precedence (higher beats lower):

1. **Agentic (10)** — if there's a tool loop, that's the category, regardless of what the tools do.
2. **Reasoning (9)** — if there's chain-of-thought or sequential calls with carried state.
3. **Generation (8)** — if the output is novel prose meant for a human.
4. **Rephrasing (7)** — if output preserves meaning but changes surface.
5. **Summarization (6)** — if output is a shorter version of the input.
6. **Routing (3)** vs **Classification (2)** — routing wins if the output becomes a code-path key.
7. **Extraction (1)** wins over Normalization (5) if multiple fields are pulled out.
8. **Validation (4)** wins over Classification (2) for binary yes/no on a fixed rule.

When unsure, write the dominant category and parenthesize the secondary (e.g. "extraction (with light classification)").

---

## Quick-reference matrix

| # | Category | Default verdict bias | Replacement difficulty |
|---|---|---|---|
| 1 | Extraction | 🟢 / 🟡 | low–medium |
| 2 | Classification | 🟡 | medium |
| 3 | Routing | 🟢 / 🟡 | low–medium |
| 4 | Validation | 🟢 | low |
| 5 | Normalization | 🟢 | low |
| 6 | Summarization | 🔴 (🟡 if extractive) | high if extractive |
| 7 | Rephrasing | 🔴 | n/a |
| 8 | Generation | 🔴 | n/a |
| 9 | Reasoning | 🔴 | n/a |
| 10 | Agentic | 🔴 | n/a |

This matrix is a _bias_, not a verdict. Apply [rubric.md](rubric.md)'s disqualifiers before committing to a colour.
