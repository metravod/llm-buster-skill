# Replacement recipes

Code recipes for each 🟢 / 🟡 task category. Use these as starting points — adapt the field names, regexes, and thresholds to the actual call being replaced. Every recipe includes the original LLM shape and the deterministic equivalent, side by side.

---

## 1. Structured extraction

### 1a. JSON from a known-template log line

**LLM version**

```python
resp = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "Extract user_id and event_name from the log line as JSON."},
        {"role": "user", "content": line},
    ],
    response_format={"type": "json_object"},
)
data = json.loads(resp.choices[0].message.content)
```

**Deterministic**

```python
import re

LOG_RE = re.compile(r"user=(?P<user_id>\d+)\s+event=(?P<event_name>[a-z_]+)")

def parse_log(line: str) -> dict[str, str] | None:
    m = LOG_RE.search(line)
    return m.groupdict() if m else None
```

Edge cases to cover with tests: missing field, trailing whitespace, multi-line input, malformed integer in `user_id`.

### 1b. Fields from an RFC 5322 email

**Deterministic**

```python
from email import message_from_string
from email.utils import parsedate_to_datetime, parseaddr

def parse_email(raw: str) -> dict:
    msg = message_from_string(raw)
    name, addr = parseaddr(msg["From"] or "")
    body = msg.get_payload(decode=True)
    if isinstance(body, bytes):
        body = body.decode(msg.get_content_charset() or "utf-8", errors="replace")
    return {
        "sender_name": name,
        "sender_email": addr,
        "subject": msg["Subject"],
        "date": parsedate_to_datetime(msg["Date"]) if msg["Date"] else None,
        "body": body,
    }
```

### 1c. Schema-bound parsing with `pydantic`

If the LLM call was using `instructor` with a `response_model=`, you may already have the schema. Drop in handwritten parsing:

```python
from pydantic import BaseModel, ValidationError

class Invoice(BaseModel):
    number: str
    amount: float
    currency: str
    due_date: date

def parse_invoice(raw: dict) -> Invoice | None:
    try:
        return Invoice.model_validate(raw)
    except ValidationError:
        return None
```

### 1d. HTML scraping

**Deterministic** — `BeautifulSoup` + CSS selectors:

```python
from bs4 import BeautifulSoup

def parse_product(html: str) -> dict:
    s = BeautifulSoup(html, "lxml")
    return {
        "title": (s.select_one("h1.product-title") or {}).get_text(strip=True),
        "price": (s.select_one(".price") or {}).get_text(strip=True),
        "in_stock": "in stock" in (s.select_one(".stock") or BeautifulSoup("", "lxml")).get_text(strip=True).lower(),
    }
```

---

## 2. Classification

### 2a. Keyword rules (smallest hammer, try first)

```python
RULES: list[tuple[re.Pattern, str]] = [
    (re.compile(r"\b(refund|money\s+back|chargeback)\b", re.I), "refund"),
    (re.compile(r"\b(bug|broken|crash|error)\b", re.I), "bug_report"),
    (re.compile(r"\b(how\s+do\s+i|how\s+can\s+i|where\s+is)\b", re.I), "question"),
]

def classify(text: str) -> str:
    for pat, label in RULES:
        if pat.search(text):
            return label
    return "other"
```

**When to graduate from keyword rules:** when the LLM-vs-rules disagreement rate on a 200-sample audit exceeds the user's tolerance (typically > 5%), move to 2b.

### 2b. sklearn baseline (TF-IDF + LogisticRegression)

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import Pipeline
import joblib

# Train (one-shot, on labeled samples — historical LLM outputs are a fine starting set)
pipe = Pipeline([
    ("tfidf", TfidfVectorizer(ngram_range=(1, 2), min_df=2, max_df=0.95)),
    ("clf", LogisticRegression(max_iter=1000, class_weight="balanced")),
])
pipe.fit(X_train, y_train)
joblib.dump(pipe, "intent_model.joblib")

# Serve
pipe = joblib.load("intent_model.joblib")
def classify(text: str) -> tuple[str, float]:
    probs = pipe.predict_proba([text])[0]
    idx = probs.argmax()
    return pipe.classes_[idx], float(probs[idx])
```

Ship behind a confidence threshold (typically 0.7); below threshold, fall back to LLM. This is the "🟡 with safety net" pattern.

### 2c. Embedding similarity (when you have ~10 labeled per class)

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer("all-MiniLM-L6-v2")  # 80MB, CPU-fine
ANCHORS = {
    "refund":   ["I want my money back", "Please refund me", "Charge me back"],
    "bug":      ["The app crashed", "I get an error when I", "It's broken"],
    "question": ["How do I", "Where is the", "Can I change my"],
}
labels = list(ANCHORS)
anchor_vecs = {l: model.encode(ANCHORS[l]).mean(axis=0) for l in labels}

def classify(text: str, threshold: float = 0.55) -> str:
    v = model.encode(text)
    scores = {l: float(np.dot(v, a) / (np.linalg.norm(v) * np.linalg.norm(a))) for l, a in anchor_vecs.items()}
    label, score = max(scores.items(), key=lambda kv: kv[1])
    return label if score >= threshold else "uncertain"
```

---

## 3. Routing / intent detection

If routing is by header or known field, just read it:

```python
def route_webhook(headers: dict, payload: dict) -> str:
    event = headers.get("X-GitHub-Event") or payload.get("type")
    return HANDLERS.get(event, "unknown")
```

If routing is from natural language, treat as classification and reach for 2a/2b/2c.

---

## 4. Validation

```python
# Email
from email_validator import validate_email, EmailNotValidError
def is_email(s: str) -> bool:
    try:
        validate_email(s, check_deliverability=False); return True
    except EmailNotValidError:
        return False

# Phone (E.164)
import phonenumbers
def is_phone(s: str, region: str = "US") -> bool:
    try:
        return phonenumbers.is_valid_number(phonenumbers.parse(s, region))
    except phonenumbers.NumberParseException:
        return False

# UUID
import uuid
def is_uuid(s: str) -> bool:
    try: uuid.UUID(s); return True
    except ValueError: return False

# JSON schema
from jsonschema import Draft202012Validator
SCHEMA = {...}
def is_valid(payload: dict) -> tuple[bool, list[str]]:
    errs = [e.message for e in Draft202012Validator(SCHEMA).iter_errors(payload)]
    return (not errs, errs)
```

---

## 5. Normalization

```python
# Dates
from dateutil import parser as dateparser
def to_iso(s: str) -> str | None:
    try: return dateparser.parse(s).isoformat()
    except (ValueError, OverflowError): return None

# Phone -> E.164
def to_e164(s: str, region: str = "US") -> str | None:
    try:
        p = phonenumbers.parse(s, region)
        return phonenumbers.format_number(p, phonenumbers.PhoneNumberFormat.E164) if phonenumbers.is_valid_number(p) else None
    except phonenumbers.NumberParseException:
        return None

# Currency
from babel.numbers import parse_decimal
def to_amount(s: str, locale: str = "en_US") -> float | None:
    try: return float(parse_decimal(s, locale=locale))
    except Exception: return None

# Casefold / normalization
import unicodedata
def normalize_name(s: str) -> str:
    return unicodedata.normalize("NFKC", s).strip().casefold()
```

---

## 6. Extractive summarization (the 🟡 sliver)

Only when "summary" really means "the first N bullets / sentences / headers":

```python
# Top-K sentences by TF-IDF importance
from sklearn.feature_extraction.text import TfidfVectorizer
from nltk.tokenize import sent_tokenize  # or a regex if nltk is heavy

def extractive_summary(text: str, k: int = 3) -> list[str]:
    sents = sent_tokenize(text)
    if len(sents) <= k:
        return sents
    vec = TfidfVectorizer().fit_transform(sents)
    scores = vec.sum(axis=1).A1
    top = sorted(range(len(sents)), key=lambda i: -scores[i])[:k]
    return [sents[i] for i in sorted(top)]

# Markdown header / bullet pull
import re
def pull_bullets(md: str) -> list[str]:
    return [m.group(1).strip() for m in re.finditer(r"^[*\-+]\s+(.+)$", md, re.M)]
```

For abstractive summaries (rewriting in new words), do not replace — that's category 7/8, 🔴.

---

## TS / JS equivalents

The same recipes in TypeScript, abbreviated:

```ts
// Email validation
import validator from "validator";
const isEmail = (s: string) => validator.isEmail(s);

// Date normalization (Luxon)
import { DateTime } from "luxon";
const toIso = (s: string) => DateTime.fromRFC2822(s).isValid
  ? DateTime.fromRFC2822(s).toISO()
  : DateTime.fromISO(s).toISO();

// Phone (libphonenumber-js)
import { parsePhoneNumberFromString } from "libphonenumber-js";
const toE164 = (s: string, region = "US") =>
  parsePhoneNumberFromString(s, region as any)?.number ?? null;

// Schema-bound parsing (zod)
import { z } from "zod";
const Invoice = z.object({
  number: z.string(),
  amount: z.number(),
  currency: z.string().length(3),
  due_date: z.string().transform((s) => new Date(s)),
});
type Invoice = z.infer<typeof Invoice>;
const parsed = Invoice.safeParse(rawJson);

// Fuzzy match
import { ratio } from "fast-fuzzy";
const score = ratio(a, b);
```

---

## Library cheat sheet

| Task | Python | TS / JS |
|---|---|---|
| Regex | stdlib `re` | stdlib `RegExp` |
| HTML parsing | `beautifulsoup4` + `lxml` | `cheerio`, `parse5` |
| Email parsing | stdlib `email` | `mailparser` |
| PDF text | `pdfplumber`, `pypdf` | `pdf-parse` |
| Date parsing | `python-dateutil`, `pendulum` | `luxon`, `date-fns` |
| Phone | `phonenumbers` | `libphonenumber-js` |
| Schema validation | `pydantic`, `jsonschema` | `zod`, `valibot`, `ajv` |
| Email validation | `email-validator` | `validator` |
| Fuzzy match | `rapidfuzz` | `fast-fuzzy`, `fuse.js` |
| TF-IDF + classifier | `scikit-learn` | `natural`, or call a Python service |
| Embeddings (local) | `sentence-transformers` | `@xenova/transformers` |
| Token counting | `tiktoken`, `anthropic.count_tokens` | `gpt-tokenizer`, `@anthropic-ai/tokenizer` |

---

## Pattern: shadow rollout

For every 🟡 replacement, recommend this rollout in the report rather than a hard swap:

```python
def classify(text: str) -> str:
    det = deterministic_classify(text)
    llm = llm_classify(text)
    log.info("shadow", det=det, llm=llm, agree=det == llm)
    return llm  # still serve LLM until agreement rate clears threshold
```

After 1–4 weeks, query the logs:

```sql
SELECT COUNT(*) AS total,
       SUM(CASE WHEN agree THEN 1 ELSE 0 END) AS agree,
       SUM(CASE WHEN agree THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS rate
FROM logs WHERE event = 'shadow';
```

When `rate ≥ 0.97` (user's threshold), flip the `return` — drop the LLM call. This is the safe path for every 🟡 verdict.

---

## Pattern: post-swap drift detection

Shadow rollout proves the replacement matched the LLM at swap-time. It does not protect against the input distribution shifting next month. After flipping `return` to the deterministic path, **keep a sampled LLM shadow running** for 2–4 weeks at low rate (1–5% of traffic is enough). The same `agree` column gives you a drift signal at near-zero cost:

```python
def classify(text: str) -> str:
    result = deterministic_classify(text)
    if random.random() < 0.02:  # 2% sample
        llm_result = llm_classify(text)
        log.info("post_swap_shadow", det=result, llm=llm_result, agree=result == llm_result)
    return result
```

Watch for two failure modes the swap could introduce:

1. **Agreement rate decays** (e.g. from 98% at week 1 to 91% at week 6). Inputs have drifted into shapes the deterministic rules don't cover. Action: re-audit, extend the rules or revert to LLM for the divergent slice.
2. **Agreement rate stays high but a specific label collapses to zero or to a single category.** The deterministic path may be deterministically wrong on a thin slice (e.g. one rare intent that the LLM was catching). Action: check per-label confusion, not just aggregate.

Set up an alert on `weekly_agree_rate < threshold - 3 percentage points` rather than reviewing dashboards manually — the swap saved engineering time, don't give it back.

After the shadow has been quiet for one full season (or one full traffic cycle if the app is event-driven), drop the shadow entirely. Document the date in the code so the next maintainer doesn't think "why does this 2%-sampled API call exist".

---

## What NOT to write

- **No regex for free-form prose.** If the prompt input is "user's message" or "support ticket body" without a template, do not propose a regex.
- **No `try/except: pass`** to mask parse failures. Replacements must surface errors loudly — the LLM was hiding them silently and that's how regressions hide.
- **No dependency on a model service** for the replacement (defeats the point — embeddings via local model are fine; embeddings via OpenAI are still an LLM call).
- **No "AI-powered" libraries** as replacements (Cohere classify, Hugging Face Inference API). Replacements must be deterministic and local.
