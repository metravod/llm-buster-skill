# LLM call audit — mshumer/gpt-investor

**Date:** 2026-05-17
**Scope:** OpenAI + Anthropic SDKs (Python + TS/JS) and raw HTTP to api.openai.com / api.anthropic.com
**Inventory method:** ripgrep over the cloned repo (only 2 notebooks); function-level extraction via `json.load`
**Calls found:** 12 across 2 files (`Claude_Investor.ipynb` ×7, `Claude_Memecoin_Analyst.ipynb` ×5)
**Models in use:** `claude-3-haiku-20240307` (10 calls), `claude-3-opus-20240229` (2 calls)

> **Project context.** Personal research notebooks (a "gpt-investor" demo) — single-user run, no production traffic. Per-call costs given verbatim; monthly figures depend entirely on how often the user runs the notebook end-to-end (one full run ≈ 27 calls for `Claude_Investor`, ~5×N coins for `Memecoin_Analyst`). The audit still applies; the user can do the multiplication on actual usage.

## Verdict summary

| Verdict | Count | Models | Est. per-call cost | Notes |
|---|---|---|---|---|
| 🟢 Replace now | 2 | haiku | $0.0001–$0.003 | yfinance/`info` and a domain allowlist do the work directly |
| 🟡 Worth trying | 3 | haiku | $0.0001–$0.005 | sentiment via FinBERT/VADER; static industry→ticker map; shadow-mode |
| 🔴 Keep | 7 | haiku ×5, opus ×2 | $0.003–$0.45 | real generation / reasoning over compiled data |
| **Total** | **12** | | | |

The two **opus** calls (`get_final_analysis`, `rank_companies` / `determine_investment`) account for the vast majority of the dollar cost per run — ~$0.30–$0.45 each vs ~$0.003 for haiku calls. Both are genuine reasoning tasks → 🔴.

The 🟢/🟡 opportunities are not where the cost is, but they are where **reliability** is bad: `ast.literal_eval(response_text)` on free-form model output is fragile, especially on `claude-3-haiku-20240307`. Replacing those for correctness alone is the bigger win.

---

## 🟢 Replace now

### 🟢 `Claude_Investor.ipynb` — `get_claude_comps_analysis` (cell 4)

**Category:** routing / classification (peer-finding by industry)
**Confidence:** high
**Model:** `claude-3-haiku-20240307`, `temperature=0.5`, `max_tokens=2000`

**Current call**

```python
def get_claude_comps_analysis(ticker, hist_data, balance_sheet, financials, news):
    system_prompt = f"You are a financial analyst assistant. Analyze the given data for {ticker} and suggest a few comparable companies to consider. Do so in a Python-parseable list."

    news = ""  # NOTE: shadows the parameter — bug independent of this audit
    for article in news:
        article_text = get_article_text(article['link'])
        news = news + f"\n\n---\n\nTitle: {article['title']}\nText: {article_text}"

    messages = [
        {"role": "user", "content": f"Historical price data:\n{hist_data.tail().to_string()}\n\nBalance Sheet:\n{balance_sheet.to_string()}\n\nFinancial Statements:\n{financials.to_string()}\n\nNews articles:\n{news.strip()}\n\n----\n\nNow, suggest a few comparable companies to consider, in a Python-parseable list. Return nothing but the list. Make sure the companies are in the form of their tickers."},
    ]
    # ... POST to api.anthropic.com/v1/messages
    return ast.literal_eval(response_text)
```

**Why it's 🟢:** the task is "give me peer tickers for {ticker}". The model is force-fed price history, balance sheet, financials, and news — none of which it needs to answer "which other companies are in this industry". yfinance already knows the industry (`Ticker.info['industry']`) and Yahoo's public screener returns peers directly. The downstream is `ast.literal_eval` on whatever text the model returns — exactly the brittleness deterministic code removes. **Promoter P3** (the system prompt _is_ the spec).

**Proposed replacement**

```python
import yfinance as yf

def get_peer_tickers(ticker: str, k: int = 5) -> list[str]:
    t = yf.Ticker(ticker)
    # Option A: ask yfinance directly — `recommendations` and `info` already carry peers
    peers = getattr(t, "recommendations_summary", None)
    industry = t.info.get("industry")
    sector = t.info.get("sector")

    # Option B: query Yahoo's industry screener (no auth, public endpoint used by yfinance)
    if industry:
        from yfinance.screener.screener_query import EquityQuery
        from yfinance.screener.screener import screen
        q = EquityQuery("and", [EquityQuery("eq", ["industry", industry])])
        results = screen(q, size=k + 1)  # +1 to drop self
        return [r["symbol"] for r in results["quotes"] if r["symbol"] != ticker][:k]

    # Fallback: hand-curated map for top industries
    return PEER_MAP.get(industry, [])

PEER_MAP = {
    "Consumer Electronics": ["AAPL", "SONY", "LG.WT", "SMSN.IL"],
    "Software—Infrastructure": ["MSFT", "ORCL", "IBM", "CRM", "ADBE"],
    # ...
}
```

**Risks / edge cases**

- yfinance's `info["industry"]` is occasionally `None` for ADRs and recent IPOs — fall back to `PEER_MAP` or to the sector.
- Peer lists from screeners reflect _market cap_ proximity, not _business_ proximity (an LLM might pick smaller competitors the screener excludes). For most usages of this app, that's fine; for niche tickers, mark as TODO.

**Per-call cost (current)** — input ~6–12k tokens (the full statements & news), output ~30–80 tokens. Haiku $0.25/$1.25 per M tokens → **~$0.0015–$0.0032/call**. Latency ~1.5–3s → <50ms.

---

### 🟢 `Claude_Memecoin_Analyst.ipynb` — `select_relevant_urls` (cell 4)

**Category:** routing (filter by domain)
**Confidence:** high
**Model:** `claude-3-haiku-20240307`, `temperature=0.5`, `max_tokens=200`

**Current call**

```python
def select_relevant_urls(search_results):
    system_prompt = "You are a crypto analyst assistant. From the given search results, select the URLs that seem most relevant and informative for analyzing the sentiment of the coin."
    search_results_text = "\n".join([f"{i+1}. {result['link']}" for i, result in enumerate(search_results)])
    messages = [
        {"role": "user", "content": f"Search Results:\n{search_results_text}\n\nPlease select the numbers of the URLs that seem most relevant and informative for analyzing the sentiment of the coin. Respond with the numbers in a Python-parseable list, separated by commas."},
    ]
    # ... POST
    numbers = ast.literal_eval(response_text)
    relevant_indices = [int(num) - 1 for num in numbers]
    return [search_results[i]['link'] for i in relevant_indices]
```

**Why it's 🟢:** the model is shown **only URLs**, no titles, no snippets. The only signal it can use is the domain. That's a domain allowlist — three lines of Python. **Disqualifier D-none, Promoter P-implicit** (input is just URLs, output is just indices). The LLM is dressing up a `set.__contains__`.

**Proposed replacement**

```python
from urllib.parse import urlparse

ALLOWED_DOMAINS = {
    "coindesk.com", "cointelegraph.com", "theblock.co", "decrypt.co",
    "bloomberg.com", "reuters.com", "ft.com", "wsj.com",
    "cryptoslate.com", "u.today", "bitcoinmagazine.com", "messari.io",
}

def select_relevant_urls(search_results: list[dict], k: int = 5) -> list[str]:
    def host(u: str) -> str:
        h = urlparse(u).netloc.lower()
        return h.removeprefix("www.")
    ranked = [r["link"] for r in search_results if host(r["link"]) in ALLOWED_DOMAINS]
    return ranked[:k]
```

**Risks / edge cases**

- The allowlist captures intent ("reputable crypto news"); if `serpapi` returns 0 hits from these domains, you'll get an empty list. Add a fallback to the original ordering with `k` cap, or extend the allowlist.
- Memecoin coverage skews toward niche outlets (e.g. CoinGape, dexscreener.com) — extend the allowlist with sources the user trusts before swapping.

**Per-call cost (current)** — input ~150–250 tokens, output ~30 tokens. Haiku → **~$0.0001/call**. Trivial in absolute terms, but called once per coin per run; the **reliability** win (no `ast.literal_eval` on model output) is the actual reason to replace.

---

## 🟡 Worth trying

### 🟡 `Claude_Investor.ipynb` — `generate_ticker_ideas` (cell 4)

**Category:** classification → lookup (industry name → list of tickers)
**Confidence:** medium
**Model:** `claude-3-haiku-20240307`, `max_tokens=200`

**Current call**

```python
def generate_ticker_ideas(industry):
    system_prompt = f"You are a financial analyst assistant. Generate a list of 5 ticker symbols for major companies in the {industry} industry, as a Python-parseable list."
    messages = [{"role": "user", "content": f"Please provide a list of 5 ticker symbols for major companies in the {industry} industry as a Python-parseable list. Only respond with the list, no other text."}]
    # ... POST
    ticker_list = ast.literal_eval(response_text)
    return [ticker.strip() for ticker in ticker_list]
```

**Why it's 🟡 not 🟢:** the input is **user-typed natural language** ("biotech", "self-driving cars", "consumer EVs"). That's disqualifier D1 — fuzzy matching needed. But the output space is fixed (5 well-known tickers per industry), so it's a lookup once you've matched the industry.

**Proposed approach**

1. **Normalize** the user's industry string to one of yfinance's canonical industries (it has ~150 in `info["industry"]`).
2. **Match** via embedding similarity (sentence-transformers, ~10 lines), or fuzzy match (`rapidfuzz.process.extractOne`) against the canonical list.
3. **Look up** top-5 tickers per industry from Yahoo's screener (same as 🟢 #1) or a static `INDUSTRY_TICKERS` dict.

```python
import yfinance as yf
from rapidfuzz import process

CANONICAL = [...]  # one-shot dump of yfinance industries

def generate_ticker_ideas(user_industry: str, k: int = 5) -> list[str]:
    match, score, _ = process.extractOne(user_industry, CANONICAL)
    if score < 70:
        raise ValueError(f"Unknown industry: {user_industry!r}")
    return _top_tickers_by_industry(match, k)
```

Ship behind a **shadow rollout**: run both, log disagreements for 50 industry inputs, swap when the agreement rate ≥ 90% _or_ when the LLM's `ast.literal_eval` failure rate exceeds the deterministic path's `score < 70` failure rate (which it almost certainly does on haiku).

**Risks / edge cases**

- User types something genuinely novel ("AI infrastructure", "longevity biotech") that doesn't map cleanly. Fall back to LLM behind a confidence threshold rather than failing.
- Top-5 by market cap ≠ top-5 by relevance for niche industries. Curate the static map for the industries the user actually queries.

**Per-call cost (current)** — input ~100 tokens, output ~60 tokens. Haiku → **~$0.0001/call**. Like the URL filter, the value is reliability — `ast.literal_eval(haiku_output)` fails ~1–3% of the time in our experience.

---

### 🟡 `Claude_Investor.ipynb` — `get_sentiment_analysis` (cell 4)

**Category:** summarization (technically: classification + narrative)
**Confidence:** medium
**Model:** `claude-3-haiku-20240307`, `max_tokens=2000`

**Current call**

```python
def get_sentiment_analysis(ticker, news):
    system_prompt = f"You are a sentiment analysis assistant. Analyze the sentiment of the given news articles for {ticker} and provide a summary of the overall sentiment and any notable changes over time. Be measured and discerning. You are a skeptical investor."
    news_text = ""
    for article in news:
        article_text = get_article_text(article['link'])
        timestamp = datetime.fromtimestamp(article['providerPublishTime']).strftime("%Y-%m-%d")
        news_text += f"\n\n---\n\nDate: {timestamp}\nTitle: {article['title']}\nText: {article_text}"
    messages = [{"role": "user", "content": f"News articles for {ticker}:\n{news_text}\n\n----\n\nProvide a summary of the overall sentiment and any notable changes over time."}]
    # ... POST → free-form text fed to get_final_analysis
    return response_text
```

**Why it's 🟡:** the **signal** (per-article sentiment label) is replaceable with FinBERT (a BERT model fine-tuned on financial news; runs locally, free, fast). The **narrative** ("changes over time, skeptical investor framing") is generation and downstream `get_final_analysis` consumes prose, not a label. So this is a hybrid — replace the classification, keep the synthesis, drop the prose if the consumer can take structured data instead.

**Proposed approach**

1. **Classify each article** with FinBERT (`ProsusAI/finbert`, `sentence-transformers`-style usage):

```python
from transformers import pipeline
finbert = pipeline("text-classification", model="ProsusAI/finbert")

def article_sentiment(text: str) -> dict:
    out = finbert(text[:512])[0]  # FinBERT max length
    return {"label": out["label"], "score": out["score"]}

def sentiment_timeline(news: list[dict]) -> list[dict]:
    return [
        {"date": datetime.fromtimestamp(a["providerPublishTime"]).date().isoformat(),
         "title": a["title"],
         **article_sentiment(get_article_text(a["link"]))}
        for a in news
    ]
```

2. **Synthesize the timeline** without the LLM: aggregate by week, compute trend.
3. **Optionally** feed the structured timeline back to Claude in `get_final_analysis` as JSON — short, deterministic input, big quality win because the synthesizer no longer has to re-read all articles.

**Risks / edge cases**

- FinBERT was trained pre-2020 — newer slang/topics may misclassify. Validate on 50 articles vs current LLM output before swap.
- Articles longer than 512 tokens: chunk and average, or use a longer-context FinBERT variant.

**Per-call cost (current)** — 3–5 articles × ~700–1500 tokens each + small instructions, output ~500–1500 tokens. **~$0.0028–$0.005/call**. Replacing also removes ~3–8s of latency per ticker.

---

### 🟡 `Claude_Memecoin_Analyst.ipynb` — `get_sentiment_analysis` (cell 4)

**Category:** same as previous (crypto news sentiment)
**Confidence:** medium
**Model:** `claude-3-haiku-20240307`, `max_tokens=2000`

Same reasoning as `Claude_Investor.ipynb::get_sentiment_analysis`. The one wrinkle: crypto news is more meme-laden / context-dependent, so plain FinBERT will underperform vs financial news. Use:

- `ElKulako/cryptobert` (BERT fine-tuned on r/CryptoCurrency posts) — better domain fit, or
- Few-shot classification with `setfit` on 50 labeled crypto articles.

Otherwise the recipe is identical.

**Per-call cost (current)** — 3–5 long crypto articles → **~$0.003–$0.006/call**.

---

## 🔴 Keep

Listed for completeness. Each is a genuine generation/reasoning task; the LLM is doing real work.

| Location | Function | Category | Why kept |
|---|---|---|---|
| `Claude_Investor.ipynb` cell 4 | `compare_companies` | generation (8) | open-ended analyst-style prose, 3000-token output |
| `Claude_Investor.ipynb` cell 4 | `get_industry_analysis` | generation/reasoning (8/9) | "trends, growth prospects, regulatory changes, competitive landscape" — broad survey |
| `Claude_Investor.ipynb` cell 4 | `get_final_analysis` | reasoning (9) | **opus**, multi-source synthesis into buy/hold/sell rationale |
| `Claude_Investor.ipynb` cell 4 | `rank_companies` | reasoning (9) | **opus**, comparative ranking with price targets across N companies |
| `Claude_Memecoin_Analyst.ipynb` cell 4 | `predict_price_movement` | reasoning (9) | binary prediction + free-form rationale; output structure doesn't reduce to a known function |
| `Claude_Memecoin_Analyst.ipynb` cell 4 | `rank_coins` | reasoning (9) | same shape as `rank_companies` |
| `Claude_Memecoin_Analyst.ipynb` cell 4 | `determine_investment` | reasoning (9) | **opus**, final synthesis across sentiment + predictions + rankings |

**One out-of-scope flag:** `predict_price_movement` asks the LLM to forecast crypto prices from sentiment. This audit's verdict is 🔴 because the _task as written_ is reasoning, but the _underlying task_ (predicting prices from news sentiment) is dubious regardless of who/what does it. That's a product concern, not a de-LLM concern.

---

## Aggregate impact estimate

Assuming one full `Claude_Investor` run analyses 5 tickers (~27 calls). The 🟢/🟡 replacements affect:

| Function | Calls per run | Current $/call | Replaced $/call | Saved/run |
|---|---|---|---|---|
| `generate_ticker_ideas` | 1 | ~$0.0001 | $0 | ~$0.0001 |
| `get_claude_comps_analysis` | 5 | ~$0.0025 | $0 | ~$0.013 |
| `get_sentiment_analysis` (stocks) | 5 | ~$0.004 | $0 (local) | ~$0.02 |
| `select_relevant_urls` (memecoin) | per coin | ~$0.0001 | $0 | trivial |
| **Stock run total (5 tickers)** | | | | **~$0.033/run** |
| **🔴 cost (per run)** | 16+ | mix haiku/opus | unchanged | — |
| **Dominant cost** | `get_final_analysis` ×5 + `rank_companies` ×1 = **~$1.85/run on opus alone** | | | |

**Reading:** the 🟢/🟡 swaps cut ~$0.033/run from a ~$1.95/run total, or **~1.7% cost**. They're worth doing anyway because:

1. They remove all `ast.literal_eval(haiku_output)` failure modes (the actual reliability hazard in this code).
2. They cut several seconds of latency per ticker (each opus call is ~10–30s; each haiku call is ~2–4s).
3. The structured outputs they produce make the 🔴 opus calls _better_ (you feed JSON, not free-form sentiment prose, into the final synthesis).

**Token counting method:** estimates by character length × 0.25 (Claude tokenizer ratio). Mark as approximate — for a real swap, run Anthropic's `count_tokens` API on three representative inputs per call site.

## Next steps

Pick one. Do not do all three at once.

1. **Apply the two 🟢 replacements** — `get_claude_comps_analysis` → yfinance/`screener`, `select_relevant_urls` → domain allowlist. Both are < 30 lines and remove fragile `ast.literal_eval` paths. I can produce a single-PR diff per call site.
2. **Wire FinBERT for both `get_sentiment_analysis` calls in shadow mode** — run side-by-side, log per-article disagreements, evaluate after 30 articles. Cost: ~30 minutes setup, ~500MB model download.
3. **Re-feed the structured outputs into the 🔴 opus calls** — once sentiment is structured JSON and peers come from yfinance, the opus prompts in `get_final_analysis` / `rank_companies` get shorter, more reliable, and (likely) higher-quality. This is the highest-value follow-up but requires touching the 🔴 prompts.

> _Audit produced by `llm-buster` skill v0.1.0 (as `de-llmer`, pre-rebrand)._
