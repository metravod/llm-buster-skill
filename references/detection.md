# Detection patterns — finding LLM calls

The basic inventory in `SKILL.md` covers ~80% of real projects. Load this file when the basic grep misses calls (heavily wrapped projects), or when you need to enumerate _every_ call before producing a final report.

## Python — OpenAI

### v1+ SDK (`openai>=1.0`)

```text
from openai import OpenAI
from openai import AsyncOpenAI
from openai import AzureOpenAI

client = OpenAI(...)            # or env-based: OpenAI()
client.chat.completions.create(...)
client.chat.completions.stream(...)
client.completions.create(...)              # legacy completions endpoint
client.responses.create(...)                # newer Responses API
client.beta.chat.completions.parse(...)     # structured outputs helper
client.embeddings.create(...)               # embeddings — usually a 🟢 candidate to cache
```

Grep:

```bash
rg -n -e 'from openai import' -e 'import openai' \
   -e '\bOpenAI\(' -e '\bAsyncOpenAI\(' -e '\bAzureOpenAI\(' \
   -e '\.chat\.completions\.(create|stream|parse)\b' \
   -e '\.completions\.create\b' \
   -e '\.responses\.create\b' \
   -e '\.embeddings\.create\b'
```

### v0.x SDK (legacy, still common)

```text
import openai
openai.ChatCompletion.create(...)
openai.Completion.create(...)
openai.Embedding.create(...)
```

Grep adds: `-e 'openai\.(ChatCompletion|Completion|Embedding)\.'`.

## Python — Anthropic

```text
from anthropic import Anthropic, AsyncAnthropic, AnthropicBedrock, AnthropicVertex

client = Anthropic(...)
client.messages.create(...)
client.messages.stream(...)
client.messages.count_tokens(...)
client.completions.create(...)              # legacy text completions
```

Grep:

```bash
rg -n -e 'from anthropic import' -e 'import anthropic' \
   -e '\bAnthropic\(' -e '\bAsyncAnthropic\(' \
   -e '\.messages\.(create|stream|count_tokens)\b' \
   -e 'anthropic\.completions\.create\b'
```

## TypeScript / JavaScript — OpenAI

```ts
import OpenAI from "openai";
import { OpenAI, AzureOpenAI } from "openai";
const client = new OpenAI({ apiKey: ... });
client.chat.completions.create({...})
client.chat.completions.stream({...})
client.completions.create({...})
client.responses.create({...})
client.embeddings.create({...})
client.beta.chat.completions.parse({...})
```

Grep:

```bash
rg -n -e "from ['\"]openai['\"]" -e 'new OpenAI\(' -e 'new AzureOpenAI\(' \
   -e '\.chat\.completions\.(create|stream|parse)\(' \
   -e '\.completions\.create\(' \
   -e '\.responses\.create\(' \
   -e '\.embeddings\.create\('
```

## TypeScript / JavaScript — Anthropic

```ts
import Anthropic from "@anthropic-ai/sdk";
import { Anthropic, AnthropicBedrock } from "@anthropic-ai/sdk";
const client = new Anthropic({ apiKey: ... });
client.messages.create({...})
client.messages.stream({...})
client.messages.countTokens({...})
```

Grep:

```bash
rg -n -e "from ['\"]@anthropic-ai/sdk['\"]" -e 'new Anthropic\(' \
   -e '\.messages\.(create|stream|countTokens)\('
```

## Raw HTTP — both providers

Some projects skip the SDK and `fetch`/`requests`/`httpx` the endpoint directly. Grep:

```bash
rg -n -e 'api\.openai\.com/v1/(chat/completions|completions|responses|embeddings)' \
   -e 'api\.anthropic\.com/v1/(messages|complete)' \
   -e 'OPENAI_API_KEY' -e 'ANTHROPIC_API_KEY' \
   -e 'Bearer sk-' -e 'x-api-key.*sk-ant'
```

Treat raw-HTTP call sites the same as SDK calls — the prompt is still in the `messages=[...]` JSON payload.

## Wrapper frameworks

If the user's project layers a wrapper on top, the SDK grep above will _also_ find the underlying call most of the time (LangChain, LiteLLM, instructor all import the provider SDKs). Use these patterns to find the **prompt definition**, which is what you actually need to classify the task.

### LangChain

```text
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain.prompts import PromptTemplate, ChatPromptTemplate
from langchain.chains import LLMChain
PromptTemplate.from_template("...")
ChatPromptTemplate.from_messages([...])
chain = prompt | llm | parser
chain.invoke({...})  /  chain.ainvoke({...})  /  chain.stream({...})
```

The classifiable spec is the `PromptTemplate` content, not the `.invoke()` call.

### LiteLLM

```text
from litellm import completion, acompletion
completion(model="gpt-4o", messages=[...])
```

Treat as an OpenAI call — same `messages=[...]` payload shape.

### instructor

```text
import instructor
client = instructor.from_openai(OpenAI())
client.chat.completions.create(response_model=MyPydanticModel, ...)
```

The `response_model=` argument **is the spec**. If `MyPydanticModel` has 4 plain fields with regex-validatable types — strong 🟢 candidate.

### LlamaIndex

```text
from llama_index.llms.openai import OpenAI as LIOpenAI
from llama_index.llms.anthropic import Anthropic as LIAnthropic
llm.complete(...) / llm.chat(...) / llm.predict(...)
```

### Haystack

```text
from haystack.components.generators import OpenAIGenerator, AnthropicGenerator
from haystack.components.generators.chat import OpenAIChatGenerator
generator.run(prompt=...)
```

## Detection-only providers

The skill's **replacement recipes** target OpenAI and Anthropic. But many real projects route some or all of their LLM calls through provider-neutral wrappers, Google Gemini, AWS Bedrock (non-Anthropic models), or local runtimes. The audit still needs to **find** these calls and include them in the inventory so the user understands the full LLM surface — even though the per-call analysis loop won't propose a regex for, say, a Gemini call against open-ended user input.

When a call site uses one of the providers below, mark it as `[detection-only]` in the inventory, classify the task category, give a verdict using the rubric, but **skip the replacement code block** — instead, write "Replacement recipe out of scope for this skill version; rubric applies." This keeps the audit honest about LLM cost surface without claiming expertise the skill doesn't have for those SDKs.

### Multi-provider wrappers

#### LiteLLM (already covered above for OpenAI shape)

LiteLLM normalizes 100+ providers to the OpenAI `messages=[...]` shape. The `model=` string tells you the real provider — `model="gemini/gemini-1.5-pro"`, `model="bedrock/anthropic.claude-3-haiku"`, `model="ollama/llama3"`. Inspect the model string before assuming OpenAI.

#### LangChain provider modules

```text
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_aws import ChatBedrock, BedrockLLM
from langchain_ollama import ChatOllama
from langchain_mistralai import ChatMistralAI
from langchain_cohere import ChatCohere
```

Grep: `rg -n 'langchain_(google_genai|aws|ollama|mistralai|cohere)'`

### Google Gemini

```text
# Newer unified SDK
from google import genai
client = genai.Client(api_key=...)
client.models.generate_content(model="gemini-1.5-pro", contents=...)
client.models.generate_content_stream(...)

# Legacy SDK
import google.generativeai as genai
genai.configure(api_key=...)
model = genai.GenerativeModel("gemini-1.5-flash")
model.generate_content(...)

# Vertex AI
from vertexai.generative_models import GenerativeModel
GenerativeModel("gemini-1.5-pro").generate_content(...)
```

Grep:

```bash
rg -n -e 'from google import genai' -e 'google\.generativeai' \
   -e 'from vertexai' -e 'GenerativeModel\(' \
   -e 'generate_content\(' -e 'generate_content_stream\('
```

Raw HTTP endpoint: `generativelanguage.googleapis.com/v1*/models/.*:generateContent`.

### AWS Bedrock (non-Anthropic models)

Anthropic-via-Bedrock is in scope (covered by `AnthropicBedrock` above). Other Bedrock models — Titan, Llama, Mistral, Cohere, AI21, Stability — are detection-only.

```text
import boto3
client = boto3.client("bedrock-runtime")
client.invoke_model(modelId="amazon.titan-text-express-v1", body=...)
client.invoke_model_with_response_stream(modelId="meta.llama3-70b-instruct-v1:0", ...)
client.converse(modelId=..., messages=[...])
client.converse_stream(...)
```

Grep:

```bash
rg -n -e "boto3\.client\(['\"]bedrock-runtime" \
   -e 'invoke_model(_with_response_stream)?\(' \
   -e '\.converse(_stream)?\('
```

Inspect the `modelId=` string. `anthropic.claude-*` → use the Anthropic rubric. Anything else → detection-only.

### Local runtimes — Ollama, vLLM, llama.cpp

```text
# Ollama — module-level API
import ollama
ollama.chat(model="llama3", messages=[...])
ollama.generate(model=..., prompt=...)

# Ollama — explicit Client (common when pointing at a remote host)
from ollama import Client
llm = Client(host="http://10.0.0.5:11434", headers={"Authorization": "Bearer ..."})
llm.chat(model=..., messages=[...])

# Or via OpenAI-compatible endpoint: http://localhost:11434/v1/chat/completions

# vLLM (usually OpenAI-compatible)
# Endpoint: http://<host>:8000/v1/chat/completions, model="any string"

# llama-cpp-python
from llama_cpp import Llama
llm = Llama(model_path="...")
llm.create_chat_completion(messages=[...])
```

Grep:

```bash
rg -n -e 'import ollama' -e 'from ollama' -e 'ollama\.(chat|generate)\(' \
   -e 'Client\(\s*host=' \
   -e 'from llama_cpp' -e 'Llama\(model_path' \
   -e 'localhost:(11434|8000)/v1'
```

Local runtimes have zero per-call cost but real latency and ops cost. The audit should still flag whether the task is deterministic-replaceable on the same rubric — a regex is still cheaper than a 7B-parameter model invocation, even local.

### Cohere, Mistral, AI21, Together, Groq, Fireworks, Replicate

```text
import cohere; co = cohere.Client(...); co.chat(...)
from mistralai import Mistral; Mistral(...).chat.complete(...)
import ai21; ai21.Completion.execute(...)
from together import Together; Together().chat.completions.create(...)
from groq import Groq; Groq().chat.completions.create(...)
import replicate; replicate.run("...", input=...)
```

Grep: `rg -n -e 'import cohere' -e 'from mistralai' -e 'import ai21' -e 'from together' -e 'from groq' -e 'import replicate'`

Most of these mirror the OpenAI shape (Together, Groq, Fireworks are OpenAI-compatible). Use the same `messages=[...]` reading strategy; just skip the replacement recipe.

## AST-level detection (when grep is noisy)

For large monorepos where grep returns thousands of false positives (e.g. `openai` is a comment, or appears in a Markdown doc), narrow with a language-aware tool:

```bash
# ast-grep (sg) — pattern-match real call expressions, not strings
sg --pattern '$CLIENT.chat.completions.create($$$)' --lang python
sg --pattern '$CLIENT.messages.create($$$)' --lang python
sg --pattern '$CLIENT.chat.completions.create($$$)' --lang ts
sg --pattern '$CLIENT.messages.create($$$)' --lang ts
```

Or fall back to `ripgrep --type py --type ts --type js` to constrain by language.

## Jupyter notebooks (`.ipynb`)

Notebooks are JSON-wrapped Python. Naive `rg` over `.ipynb` files returns string-escaped fragments (`"    response = requests.post(...)\n"`) that are awkward to read and lose context. Two cleaner options:

```bash
# Option 1 — convert in place, grep on the result, clean up
jupyter nbconvert --to script path/to/notebook.ipynb --stdout | rg -n 'pattern'

# Option 2 — one-shot Python extraction that prints each code cell with index
python3 -c "
import json, sys
nb = json.load(open(sys.argv[1]))
for i, cell in enumerate(nb['cells']):
    if cell['cell_type'] == 'code':
        print(f'--- cell {i} ---'); print(''.join(cell['source']))
" path/to/notebook.ipynb | rg -n 'pattern'
```

Once you have the prompt template extracted from the cell, classify it the same way as a regular `.py` file. Cell numbers go in the report instead of line numbers (e.g. `notebook.ipynb cell 4` rather than `notebook.ipynb:42`).

If `jupyter` isn't installed, the inline Python snippet above only needs the stdlib.

## What to extract from each call site

For every hit, before classification, capture:

1. **File and line range** of the full call (not just the function name — the whole `messages=[...]` block).
2. **Model name** — `gpt-4o-mini`, `claude-haiku-4-5`, etc. Cheap models bias toward 🟢 candidacy because the user already optimized cost once.
3. **System prompt** — string literal or assembled string.
4. **User prompt template** — what dynamic data is interpolated, what's static.
5. **Response shape**:
   - `response_format=` argument (none / `json_object` / `json_schema`);
   - `tools=` + `tool_choice=` (forced function call = structured);
   - downstream parse (`json.loads`, regex, returned to user as string).
6. **Temperature / top_p** — informational, doesn't drive verdict but worth noting.
7. **Streaming?** — `stream=True` (Python) / `.stream(...)` — strong 🔴 signal if output goes to a UI.
8. **Call volume hint** — if the file has obvious traffic indicators (`@app.post`, cron schedule, bulk loop), note it for the savings estimate.

If the prompt is assembled from many fragments across files, stop and ask the user for a concrete example rendered prompt rather than guessing.
