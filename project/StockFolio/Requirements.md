# StockFolio — Full Project Requirements

> US stock portfolio tracker with hybrid AI recommendations for a private beta group.  
> Last updated: April 2026

---

## Table of contents

1. [Project overview](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#1-project-overview)
2. [Goals & success criteria](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#2-goals--success-criteria)
3. [Users & access model](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#3-users--access-model)
4. [Functional requirements](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#4-functional-requirements)
5. [Non-functional requirements](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#5-non-functional-requirements)
6. [AI recommendation engine](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#6-ai-recommendation-engine)
7. [Third-party data sources](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#7-third-party-data-sources)
8. [System architecture](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#8-system-architecture)
9. [Microservices breakdown](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#9-microservices-breakdown)
10. [Databases & storage design](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#10-databases--storage-design)
11. [Message queue design (Kafka)](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#11-message-queue-design-kafka)
12. [ETL pipeline (Airflow)](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#12-etl-pipeline-airflow)
13. [Deployment & infrastructure](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#13-deployment--infrastructure)
14. [CI/CD pipeline](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#14-cicd-pipeline)
15. [Out of scope (v1)](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#15-out-of-scope-v1)

---
![[Screenshot 2569-04-20 at 10.10.17 AM.png]]
![[Screenshot 2569-04-20 at 10.10.34 AM.png]]
## 1. Project overview

StockFolio is a private stock portfolio tracker that helps a small group of friends (5–20 users) manage their US equity holdings and receive AI-powered buy/sell/hold recommendations. The system combines real-time market data, technical indicator analysis, and LLM-based news sentiment analysis to produce an explainable Signal Score per stock. All infrastructure is managed as code and deployed on GCP.

**Markets covered:** NYSE and NASDAQ equities only (no crypto, forex, or options in v1).

**Universe of stocks:** S&P 500 + NASDAQ 100 for the discovery/recommendation feed. Users may manually track any US-listed ticker in their personal portfolio.

---

## 2. Goals & success criteria

|Goal|Measurable outcome|
|---|---|
|Portfolio tracking is accurate|P&L matches brokerage statements within $0.01 rounding|
|Live prices are timely|Price updates reach the browser within 3 seconds during market hours|
|AI signals are useful|Users report the reasoning is clear and actionable (qualitative beta feedback)|
|System is stable|Uptime ≥ 99% during market hours (9:30am–4pm ET, Mon–Fri)|
|ETL is reliable|Nightly pipeline completes before 6am UTC with no data gaps|

---

## 3. Users & access model

StockFolio is invite-only. There is no public registration. An admin creates invite codes that new users redeem on signup.

**Roles:**

- **Admin** — can generate invite codes, view all users (not their portfolios), trigger manual ETL runs.
- **User** — can manage their own portfolio, watchlist, and alert preferences. Cannot see other users' data.

Each user's portfolio, positions, alerts, and analytics are strictly private. No social or sharing features in v1.

---

## 4. Functional requirements

### 4.1 Authentication

- Users register with email + password, or via Google OAuth.
- Passwords are hashed with bcrypt (cost factor ≥ 12).
- Auth uses short-lived JWT access tokens (15 min) and long-lived refresh tokens (30 days, stored in httpOnly cookies).
- Invite code is required at registration. Each code is single-use.
- Users can reset their password via email link (expires in 1 hour).

### 4.2 Portfolio management

- A user can add a position by specifying: ticker symbol, number of shares, purchase price per share, and purchase date.
- A user can edit or delete a position.
- The system displays for each position:
    - Current price (live during market hours, last close after hours)
    - Current market value (shares × current price)
    - Unrealized P&L in dollars and percentage
    - Daily change in dollars and percentage
    - Signal Score badge (BUY / HOLD / SELL) with score number
- The portfolio summary shows:
    - Total current value
    - Total unrealized P&L
    - Daily portfolio change
    - Sector allocation breakdown (pie chart)
    - Portfolio value history chart (daily, selectable range: 1W / 1M / 3M / 1Y)

### 4.3 Watchlist

- Users can add tickers to a watchlist (no purchase info required).
- Watchlist displays current price, daily change, and Signal Score for each ticker.
- Signal Scores are computed for watchlist tickers with the same frequency as portfolio tickers.

### 4.4 Live market data

- During market hours (9:30am–4:00pm ET, Mon–Fri), prices update every 15 seconds via WebSocket push to the frontend.
- The Market Data service is responsible for polling Polygon.io and publishing price updates to Kafka.
- Redis caches the latest quote per ticker so any service can read it without calling Polygon.io directly.
- Outside market hours, the system displays the last closing price with a clear "Market closed" label.
- Pre-market and after-hours quotes are displayed as informational only and do not update portfolio values.

### 4.5 Signal Score

See [Section 6](https://claude.ai/chat/06ada235-3d96-4f32-a6fd-fc6ac2c1e93c#6-ai-recommendation-engine) for full details.

- A Signal Score (0–100) is computed per ticker for every ticker tracked by at least one user.
- The score updates every 15 minutes during market hours and once after market close.
- The score maps to: 0–35 = SELL (red), 36–64 = HOLD (yellow), 65–100 = BUY (green).
- Each score is accompanied by a suggested entry price range and 3–5 bullet points of plain-English reasoning.
- Users can view the score breakdown (technical sub-score vs sentiment sub-score) on the stock detail page.

### 4.6 Stock discovery feed

- A "Interesting this week" feed surfaces up to 20 tickers from the S&P 500 + NASDAQ 100 that:
    - Are NOT in the user's current portfolio.
    - Have a Signal Score ≥ 70 as of the latest nightly run.
    - Have traded above average volume in the last 5 days.
- The feed is personalized: stocks in sectors the user already holds are ranked higher.
- The feed is refreshed nightly by the Airflow ETL pipeline and is the same for all users (no real-time personalization in v1).

### 4.7 Alerts & notifications

- Users can create price alerts: "Notify me when [TICKER] goes below/above [PRICE]."
- Users can create signal alerts: "Notify me when any watchlist ticker hits BUY."
- Alerts are evaluated by the AI Signal service consuming the `price-updates` Kafka topic.
- Triggered alerts are delivered as in-app notifications (visible in the notification bell in the UI).
- Users can mark notifications as read or clear all.
- Email notifications are out of scope for v1. Push notifications (mobile) are out of scope for v1.

### 4.8 Analytics dashboard

A personal analytics page powered by BigQuery showing:

- Portfolio value chart over time (daily snapshots).
- Win/loss ratio across all closed and open positions.
- Best and worst performing positions (all time).
- Sector exposure bar chart.
- Signal Score history for each tracked ticker (was the signal right?).

---

## 5. Non-functional requirements

|Category|Requirement|
|---|---|
|**Latency**|Portfolio dashboard loads in under 500ms (p95). Signal Score detail page under 800ms.|
|**Price feed**|Price update propagates from Kafka consumer to browser WebSocket within 3 seconds.|
|**Signal freshness**|Signal Score no older than 15 minutes during market hours.|
|**Availability**|99% uptime during market hours. Downtime outside market hours is acceptable for maintenance.|
|**Data consistency**|Portfolio P&L is always consistent with stored position data. No partial updates.|
|**Security**|All API endpoints require authentication. All inter-service calls use internal service tokens. HTTPS everywhere. No secrets in code — use GCP Secret Manager.|
|**Scalability**|System must handle 20–50 concurrent users and ~200 tracked tickers in v1 without autoscaling.|
|**Observability**|All services emit structured JSON logs. Key metrics (signal compute time, Kafka consumer lag, ETL duration) are visible in GCP Cloud Monitoring.|

---

## 6. AI recommendation engine

The Signal Score is computed by two parallel tracks that are merged by a weighted combiner.

### 6.1 Technical track (weight: 60%)

The technical engine runs on historical OHLCV data (at least 200 days of daily candles) and computes:

|Indicator|Signal logic|
|---|---|
|RSI (14-day)|< 30 → strong buy pressure; > 70 → overbought warning|
|MACD (12/26/9)|Bullish crossover → positive; bearish crossover → negative|
|Bollinger Bands (20-day, 2σ)|Price near lower band → oversold; near upper band → overbought|
|50-day / 200-day MA crossover|Golden cross → positive; death cross → negative|
|Volume trend|Above-average volume confirming a price move adds weight to that direction|

Each indicator contributes a sub-score. These are normalized and combined into a technical sub-score (0–100). Library: `pandas-ta` (Python).

### 6.2 Sentiment track (weight: 40%)

The sentiment engine fetches recent text data and passes it to an LLM (Claude claude-sonnet-4-20250514 via Anthropic API):

**Inputs:**

- Last 48 hours of news headlines for the ticker (NewsAPI / Benzinga).
- Most recent 8-K filing summary from SEC EDGAR (if available in last 7 days).
- Most recent earnings call transcript summary (if available).

**LLM prompt structure:**

```
You are a professional equity analyst. Analyze the following recent news and 
regulatory filings for the stock with ticker symbol {TICKER}.

Return ONLY a valid JSON object with this exact structure (no preamble, no markdown):
{
  "sentiment_score": <integer 0-100, where 0=extremely bearish, 100=extremely bullish>,
  "confidence": <"low" | "medium" | "high">,
  "key_positives": [<up to 3 short bullet strings>],
  "key_negatives": [<up to 3 short bullet strings>],
  "summary": "<one sentence plain-English summary>"
}

--- NEWS HEADLINES (last 48 hours) ---
{headlines}

--- RECENT SEC FILINGS ---
{filings_text}
```

**Output:** The LLM returns a structured JSON object. The `sentiment_score` field becomes the sentiment sub-score.

### 6.3 Signal combiner

```
signal_score = (technical_score × 0.60) + (sentiment_score × 0.40)
```

The weights are configurable per user in v2 (not in v1). The final score maps to:

|Score range|Signal|UI color|
|---|---|---|
|0 – 35|SELL|Red|
|36 – 64|HOLD|Yellow|
|65 – 100|BUY|Green|

**Price target logic:** The suggested entry price range is derived from the nearest support/resistance levels computed from Bollinger Bands and recent pivot points, not from the LLM.

**Re-computation schedule:**

- Every 15 minutes per tracked ticker during market hours.
- Once at 4:15pm ET after market close.
- On-demand if a user manually requests a refresh (rate limited to once per 5 minutes per ticker per user).

---

## 7. Third-party data sources

### 7.1 Polygon.io — Real-time & historical market data

**What it provides:** Real-time stock quotes via WebSocket, REST endpoints for historical OHLCV candles, ticker metadata, and corporate actions (splits, dividends).

**Pricing:** Free tier available. Starter plan (~$29/month) required for WebSocket access and more than 5 API calls/minute. For a private beta, Starter is sufficient.

**Sign up:** https://polygon.io → create account → API Keys → copy key.

**How to use in the project:**

```python
# Install
pip install polygon-api-client

# REST: Fetch historical daily OHLCV (last 200 days)
from polygon import RESTClient
from datetime import date, timedelta

client = RESTClient(api_key="YOUR_KEY")

aggs = client.get_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_=str(date.today() - timedelta(days=200)),
    to=str(date.today()),
    limit=200
)

for bar in aggs:
    print(bar.open, bar.high, bar.low, bar.close, bar.volume)
```

```python
# WebSocket: Subscribe to live quotes
from polygon import WebSocketClient
from polygon.websocket.models import WebSocketMessage

def handle_message(msgs: list[WebSocketMessage]):
    for msg in msgs:
        print(msg.symbol, msg.bid_price, msg.ask_price)

ws = WebSocketClient(api_key="YOUR_KEY", feed="delayed.polygon.io")
ws.subscribe("Q.*")  # subscribe to all quotes
ws.run(handle_message)
```

**Key endpoints used:**

|Endpoint|Purpose|Frequency|
|---|---|---|
|`GET /v2/aggs/ticker/{ticker}/range/1/day/{from}/{to}`|Historical OHLCV|Nightly ETL|
|`GET /v2/last/trade/{ticker}`|Latest trade price|Every 15 seconds|
|`GET /v3/reference/tickers/{ticker}`|Company info, sector|On first add|
|WebSocket `Q.*` channel|Live bid/ask quotes|During market hours|

**Store your key:** In GCP Secret Manager as `polygon-api-key`. Load via:

```python
from google.cloud import secretmanager

def get_secret(name: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    response = client.access_secret_version(name=f"projects/YOUR_PROJECT/secrets/{name}/versions/latest")
    return response.payload.data.decode("UTF-8")

POLYGON_KEY = get_secret("polygon-api-key")
```

---

### 7.2 NewsAPI — News headlines for sentiment analysis

**What it provides:** Aggregated news articles and headlines from 150,000+ sources. Filterable by keyword, language, date range, and source.

**Pricing:** Free tier = 100 requests/day, developer articles only (1-month delay on free). **Paid plan required for real-time news ($449/month for production).** For private beta on a budget, use the free tier with a 15-minute polling window and accept that headlines may be slightly delayed, or use an alternative below.

**Better free alternative for finance:** [GNews API](https://gnews.io/) — 100 requests/day free, real-time, finance-friendly. Same integration pattern.

**Sign up:** https://newsapi.org → Get API Key.

**How to use in the project:**

```python
# Install
pip install requests

import requests
from datetime import datetime, timedelta

NEWS_API_KEY = get_secret("newsapi-key")

def fetch_headlines(ticker: str, company_name: str) -> list[str]:
    """Fetch last 48 hours of headlines for a ticker."""
    from_date = (datetime.utcnow() - timedelta(hours=48)).strftime("%Y-%m-%dT%H:%M:%S")

    response = requests.get(
        "https://newsapi.org/v2/everything",
        params={
            "q": f'"{ticker}" OR "{company_name}"',
            "language": "en",
            "from": from_date,
            "sortBy": "publishedAt",
            "pageSize": 20,
            "apiKey": NEWS_API_KEY,
        },
        timeout=10
    )
    response.raise_for_status()
    articles = response.json().get("articles", [])

    # Return list of headline strings for the LLM prompt
    return [
        f"[{a['publishedAt'][:10]}] {a['source']['name']}: {a['title']}"
        for a in articles
        if a.get("title")
    ]

# Example output:
# ["[2026-04-19] Reuters: Apple beats Q2 earnings estimates by 8%",
#  "[2026-04-18] Bloomberg: iPhone shipments rise in China"]
```

**Rate limit strategy:** Cache headlines per ticker in Redis with a 15-minute TTL. If the cached value exists, skip the API call. This reduces NewsAPI calls to ~4/hour/ticker regardless of how many users track it.

```python
def fetch_headlines_cached(ticker: str, company_name: str, redis_client) -> list[str]:
    cache_key = f"headlines:{ticker}"
    cached = redis_client.get(cache_key)
    if cached:
        import json
        return json.loads(cached)

    headlines = fetch_headlines(ticker, company_name)
    redis_client.setex(cache_key, 900, json.dumps(headlines))  # 15 min TTL
    return headlines
```

---

### 7.3 SEC EDGAR — Regulatory filings

**What it provides:** All public company filings to the US Securities and Exchange Commission. Free and open. No API key required.

**Most useful filing types:**

|Form|Contains|Useful for|
|---|---|---|
|8-K|Material events: earnings, M&A, executive changes|Immediate signal events|
|10-Q|Quarterly financial report|Fundamental context|
|10-K|Annual report|Long-term context|
|DEF 14A|Proxy statement|Governance signals|

**How to use in the project:**

```python
import requests

EDGAR_BASE = "https://efts.sec.gov/LATEST/search-index"
EDGAR_SUBMISSIONS = "https://data.sec.gov/submissions"

def get_cik(ticker: str) -> str | None:
    """Map a ticker symbol to an SEC CIK number."""
    response = requests.get(
        "https://efts.sec.gov/LATEST/search-index?q=%22{ticker}%22&dateRange=custom&startdt=2020-01-01&forms=10-K".format(ticker=ticker),
        headers={"User-Agent": "StockFolio contact@yourapp.com"},  # SEC requires User-Agent
        timeout=10
    )
    # Better: use the static ticker→CIK mapping file
    mapping_url = "https://www.sec.gov/files/company_tickers.json"
    mapping = requests.get(mapping_url, headers={"User-Agent": "StockFolio contact@yourapp.com"}).json()
    for entry in mapping.values():
        if entry["ticker"].upper() == ticker.upper():
            return str(entry["cik_str"]).zfill(10)
    return None


def fetch_recent_filings(ticker: str, form_type: str = "8-K", max_filings: int = 3) -> list[dict]:
    """
    Fetch the most recent filings of a given type for a ticker.
    Returns list of dicts with keys: date, form, description, text_preview.
    """
    cik = get_cik(ticker)
    if not cik:
        return []

    headers = {"User-Agent": "StockFolio contact@yourapp.com"}
    submissions_url = f"https://data.sec.gov/submissions/CIK{cik}.json"
    data = requests.get(submissions_url, headers=headers, timeout=10).json()

    filings = data.get("filings", {}).get("recent", {})
    forms = filings.get("form", [])
    dates = filings.get("filingDate", [])
    accession_numbers = filings.get("accessionNumber", [])
    descriptions = filings.get("primaryDocument", [])

    results = []
    for i, form in enumerate(forms):
        if form == form_type and len(results) < max_filings:
            acc = accession_numbers[i].replace("-", "")
            doc = descriptions[i]
            filing_url = f"https://www.sec.gov/Archives/edgar/data/{int(cik)}/{acc}/{doc}"

            try:
                text = requests.get(filing_url, headers=headers, timeout=10).text
                # Strip HTML tags crudely — use BeautifulSoup in production
                import re
                clean = re.sub(r"<[^>]+>", " ", text)
                clean = re.sub(r"\s+", " ", clean).strip()
                preview = clean[:2000]  # First 2000 chars for LLM context
            except Exception:
                preview = ""

            results.append({
                "date": dates[i],
                "form": form,
                "text_preview": preview
            })

    return results


def format_filings_for_prompt(filings: list[dict]) -> str:
    """Format filing data as text block for the LLM prompt."""
    if not filings:
        return "No recent SEC filings found."
    lines = []
    for f in filings:
        lines.append(f"[{f['date']}] Form {f['form']}:")
        lines.append(f["text_preview"][:500])  # Trim for token budget
        lines.append("")
    return "\n".join(lines)
```

**Important EDGAR rules:**

- The `User-Agent` header is **required** by SEC. Use your app name and a contact email.
- Rate limit: maximum 10 requests/second. Add `time.sleep(0.1)` between batch calls.
- CIK numbers are padded to 10 digits with leading zeros.
- The static ticker → CIK mapping file (`company_tickers.json`) is the most reliable way to resolve tickers. Cache it locally (it changes rarely).

---

### 7.4 Financial Modeling Prep — Company fundamentals

**What it provides:** P/E ratio, EPS, market cap, revenue, sector, and industry classification. Used to populate the stock detail page and sector allocation chart.

**Pricing:** Free tier = 250 calls/day. Sufficient for a private beta (fetch fundamentals once per ticker, cache in PostgreSQL).

**Sign up:** https://financialmodelingprep.com/developer → Get free API key.

**How to use in the project:**

```python
FMP_KEY = get_secret("fmp-api-key")

def fetch_company_profile(ticker: str) -> dict:
    """Fetch static company fundamentals. Cache in PostgreSQL — call once per ticker."""
    response = requests.get(
        f"https://financialmodelingprep.com/api/v3/profile/{ticker}",
        params={"apikey": FMP_KEY},
        timeout=10
    )
    response.raise_for_status()
    data = response.json()
    if not data:
        return {}

    profile = data[0]
    return {
        "name": profile.get("companyName"),
        "sector": profile.get("sector"),
        "industry": profile.get("industry"),
        "market_cap": profile.get("mktCap"),
        "pe_ratio": profile.get("pe"),
        "eps": profile.get("eps"),
        "52w_high": profile.get("range", "").split("-")[-1],
        "52w_low": profile.get("range", "").split("-")[0],
        "description": profile.get("description", "")[:500],
    }
```

---

### 7.5 Summary: API keys setup checklist

```bash
# Store all API keys in GCP Secret Manager (never in .env or code)
gcloud secrets create polygon-api-key --data-file=<(echo -n "YOUR_KEY")
gcloud secrets create newsapi-key     --data-file=<(echo -n "YOUR_KEY")
gcloud secrets create fmp-api-key     --data-file=<(echo -n "YOUR_KEY")
gcloud secrets create anthropic-api-key --data-file=<(echo -n "YOUR_KEY")

# Grant Cloud Run service account access
gcloud secrets add-iam-policy-binding polygon-api-key \
  --member="serviceAccount:YOUR_SERVICE_ACCOUNT@PROJECT.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

|Service|Free tier limit|Paid plan needed at|
|---|---|---|
|Polygon.io|5 calls/min, no WebSocket|Day 1 for live prices (Starter ~$29/mo)|
|NewsAPI|100 req/day, 1-month delay|When beta grows > 5 tickers active (consider GNews)|
|SEC EDGAR|No limit (10 req/sec)|Never — always free|
|Financial Modeling Prep|250 calls/day|When onboarding many new tickers|
|Anthropic (Claude)|Pay-per-token|From day 1 (~$0.003/signal compute, very cheap)|

---

## 8. System architecture

### 8.1 Architecture tiers

```
[ Next.js Frontend ]
        |
[ API Gateway (Kong / Nginx) ]
        |
┌───────┬──────────┬─────────────┬──────────────┐
│ User  │ Portfolio│ Market Data │  AI Signal   │
│Service│ Service  │   Service   │   Service    │
└───────┴──────────┴─────────────┴──────────────┘
        |               |               |
    [ Kafka: price-updates · trade-events · signal-alerts ]
        |               |               |
┌─────────────┬──────────────┬────────────────────┐
│ PostgreSQL  │    Redis     │  BigQuery (OLAP)   │
│ (primary)   │   (cache)    │  (analytics)       │
└─────────────┴──────────────┴────────────────────┘
                                      |
                          [ Airflow ETL pipeline ]
                                      |
                         [ External APIs & SEC EDGAR ]
```

### 8.2 Data flow: live price update

```
Polygon.io WebSocket
    → Market Data Service (poll every 15s)
    → publish to Kafka topic: price-updates
        → Portfolio Service consumes → pushes to browser via WebSocket
        → AI Signal Service consumes → triggers re-score if price moved > 1%
        → Alert Service consumes → evaluates user price alerts
```

### 8.3 Data flow: signal score computation

```
Airflow (or on-demand trigger)
    → Market Data Service: fetch OHLCV from Polygon.io
    → AI Signal Service:
        → compute technical indicators (pandas-ta)
        → fetch headlines (NewsAPI, cached in Redis)
        → fetch SEC filings (EDGAR)
        → call Anthropic API → get sentiment JSON
        → combine scores (60/40 weighted)
        → store result in PostgreSQL (signal_scores table)
        → publish to Kafka topic: signal-alerts (if threshold crossed)
    → Frontend reads score from PostgreSQL via Portfolio Service
```

---

## 9. Microservices breakdown

### User service

- Language: Node.js (TypeScript) or Go
- Responsibilities: registration, login, JWT issuance, invite code management, password reset, profile CRUD
- Database: PostgreSQL (`users`, `invite_codes`, `refresh_tokens` tables)
- Exposes: REST `/auth/*`, `/users/*`

### Portfolio service

- Language: Go
- Responsibilities: position CRUD, portfolio value calculation, P&L computation, WebSocket server for live price push, notification delivery
- Database: PostgreSQL (`portfolios`, `positions`, `watchlists`, `notifications` tables)
- Cache: Redis (reads latest price per ticker)
- Exposes: REST `/portfolios/*`, `/watchlists/*`, WebSocket `/ws/portfolio`

### Market data service

- Language: Python
- Responsibilities: WebSocket connection to Polygon.io, quote polling, OHLCV history fetching, publishing price updates to Kafka, maintaining Redis quote cache
- Cache: Redis (`quote:{ticker}` keys with 30s TTL)
- Publishes to Kafka: `price-updates` topic
- Exposes: REST `/quotes/{ticker}` (internal only)

### AI signal service

- Language: Python
- Responsibilities: technical indicator computation, LLM sentiment analysis, signal score storage, alert evaluation
- Libraries: `pandas-ta`, `anthropic` SDK, `confluent-kafka`
- Database: PostgreSQL (`signal_scores`, `signal_history` tables)
- Consumes from Kafka: `price-updates`
- Publishes to Kafka: `signal-alerts`
- Exposes: REST `/signals/{ticker}` (internal only), `/signals/batch` for ETL use

### Notification service (lightweight, can be part of Portfolio service in v1)

- Consumes from Kafka: `signal-alerts`
- Writes notifications to PostgreSQL
- Portfolio service WebSocket push delivers them to the browser

---

## 10. Databases & storage design

### PostgreSQL (primary operational database)

Key tables (full schema to be designed separately):

```
users               — id, email, password_hash, created_at, role
invite_codes        — code, created_by, used_by, used_at
refresh_tokens      — id, user_id, token_hash, expires_at, revoked
portfolios          — id, user_id, name, created_at
positions           — id, portfolio_id, ticker, shares, purchase_price, purchase_date
watchlists          — id, user_id, ticker, added_at
alerts              — id, user_id, ticker, type (price|signal), condition, threshold, active
notifications       — id, user_id, alert_id, message, read, created_at
signal_scores       — id, ticker, score, signal (BUY|HOLD|SELL), technical_score,
                      sentiment_score, reasoning_json, computed_at
company_profiles    — ticker, name, sector, industry, market_cap, pe_ratio, updated_at
```

### Redis (cache layer)

|Key pattern|Value|TTL|
|---|---|---|
|`quote:{TICKER}`|JSON: `{price, change, change_pct, volume, timestamp}`|30 seconds|
|`headlines:{TICKER}`|JSON array of headline strings|15 minutes|
|`signal:{TICKER}`|JSON: full signal score object|15 minutes|
|`session:{user_id}`|WebSocket connection metadata|24 hours|

### BigQuery (OLAP / analytics)

|Dataset|Table|Description|
|---|---|---|
|`stockfolio_analytics`|`portfolio_daily_snapshots`|Daily portfolio value per user (loaded nightly by Airflow)|
|`stockfolio_analytics`|`price_history`|Daily OHLCV for all tracked tickers (partitioned by date)|
|`stockfolio_analytics`|`signal_history`|Signal score per ticker per day (for backtesting accuracy)|
|`stockfolio_analytics`|`trade_events`|All position opens/closes for P&L analysis|

---

## 11. Message queue design (Kafka)

### Topics

|Topic|Producer|Consumers|Message schema|
|---|---|---|---|
|`price-updates`|Market Data Service|Portfolio, AI Signal, Alert evaluation|`{ticker, price, change, volume, timestamp}`|
|`signal-alerts`|AI Signal Service|Notification Service|`{ticker, signal, score, user_ids_watching, timestamp}`|
|`trade-events`|Portfolio Service|BigQuery sink (Kafka Connect)|`{user_id, ticker, action, shares, price, timestamp}`|

### Configuration recommendations

- **Partitions:** 12 for `price-updates` (partitioned by ticker hash), 4 for others.
- **Retention:** 24 hours for `price-updates`, 7 days for `signal-alerts` and `trade-events`.
- **Consumer groups:** Each service that consumes a topic has its own consumer group, allowing independent offset management.

For the private beta, **Redpanda Cloud** (free tier: 10 GB/month) is recommended over self-hosted Kafka. It is Kafka-compatible (same client libraries) and requires zero broker management.

---

## 12. ETL pipeline (Airflow)

### DAGs

**`nightly_market_data_load`** — runs at 5:00am UTC daily (after US market close + settlement)

```
1. fetch_ohlcv_all_tickers
   └── For each tracked ticker: call Polygon.io REST for yesterday's OHLCV
   └── Write to BigQuery: price_history table

2. compute_technical_indicators
   └── For each ticker: fetch last 200 days from BigQuery
   └── Compute RSI, MACD, Bollinger, MAs using pandas-ta
   └── Store technical_score in PostgreSQL: signal_scores table

3. compute_sentiment_scores
   └── For each ticker: fetch headlines (NewsAPI) + SEC filings (EDGAR)
   └── Call Anthropic API with structured prompt
   └── Store sentiment_score and reasoning_json in PostgreSQL

4. combine_scores
   └── For each ticker: weighted blend (60/40)
   └── Update final signal + BUY/HOLD/SELL in PostgreSQL

5. generate_discovery_feed
   └── Score all S&P 500 + NASDAQ 100 tickers not in any portfolio
   └── Filter to score ≥ 70 + above-average volume
   └── Write top 20 to PostgreSQL: discovery_feed table

6. snapshot_portfolios
   └── For each user: compute today's portfolio value
   └── Write to BigQuery: portfolio_daily_snapshots table
```

**`market_hours_signal_refresh`** — runs every 15 minutes Mon–Fri 9:30am–4:00pm ET

```
1. fetch_live_quotes (Market Data Service handles this via Kafka — no Airflow needed)
2. trigger_signal_recompute
   └── POST to AI Signal Service /signals/batch with list of tickers
       that have had price movement > 1% since last compute
```

### Environment variables for Airflow workers

```bash
POLYGON_API_KEY      # from GCP Secret Manager
NEWSAPI_KEY          # from GCP Secret Manager
ANTHROPIC_API_KEY    # from GCP Secret Manager
FMP_API_KEY          # from GCP Secret Manager
POSTGRES_CONN_STR    # from GCP Secret Manager
BIGQUERY_PROJECT     # GCP project ID
KAFKA_BOOTSTRAP      # Redpanda/Kafka broker address
```

---

## 13. Deployment & infrastructure

### Recommended GCP stack

|Component|GCP service|Notes|
|---|---|---|
|Microservices|Cloud Run|Stateless, scales to 0, cheapest for beta|
|PostgreSQL|Cloud SQL (Postgres 15)|db-f1-micro is fine for beta (~$10/mo)|
|Redis|Memorystore (Redis 7)|Basic tier, 1GB (~$35/mo)|
|Kafka|Redpanda Cloud (free tier)|Or Confluent Cloud free tier|
|ETL|Cloud Composer 2|Managed Airflow, or use Cloud Scheduler + Cloud Run jobs to save cost|
|BigQuery|BigQuery|Serverless, pay-per-query (~$0 for beta volumes)|
|Object storage|Cloud Storage|Chart exports, ETL temp files|
|Secrets|Secret Manager|All API keys|
|Container registry|Artifact Registry|Docker images|
|DNS + HTTPS|Cloud Run built-in domain|Or custom domain via Cloud Run domain mapping|

### Terraform structure

```
terraform/
├── main.tf               # Provider config, project settings
├── variables.tf
├── outputs.tf
├── modules/
│   ├── cloud_run/        # One module reused per service
│   ├── cloud_sql/
│   ├── memorystore/
│   ├── bigquery/
│   ├── artifact_registry/
│   └── secret_manager/
└── environments/
    ├── dev/              # Smaller instance sizes, no HA
    └── prod/             # Production settings
```

Key Terraform resources:

```hcl
# Example: Cloud Run service
resource "google_cloud_run_v2_service" "portfolio_service" {
  name     = "portfolio-service"
  location = var.region

  template {
    containers {
      image = "${var.region}-docker.pkg.dev/${var.project_id}/stockfolio/portfolio-service:latest"
      env {
        name = "POSTGRES_CONN_STR"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.postgres_conn.secret_id
            version = "latest"
          }
        }
      }
    }
  }
}
```

### Cost estimate (private beta, ~20 users)

|Service|Estimated monthly cost|
|---|---|
|Cloud Run (5 services, low traffic)|~$5–15|
|Cloud SQL (db-f1-micro)|~$10|
|Memorystore (1 GB basic)|~$35|
|Cloud Composer 2 (smallest env)|~$100 ← most expensive, see note|
|BigQuery (< 1 TB)|~$0–5|
|Artifact Registry, Secret Manager|~$1–2|
|Redpanda Cloud|$0 (free tier)|
|**Total**|**~$55–170/mo**|

> **Cost tip:** Cloud Composer is expensive for a beta. Replace it with Cloud Scheduler triggering Cloud Run jobs for each DAG step. This drops the cost by ~$100/month and is fully sufficient for a simple nightly pipeline.

---

## 14. CI/CD pipeline

### GitHub Actions workflow per service

```yaml
# .github/workflows/deploy-portfolio-service.yml
name: Deploy portfolio service

on:
  push:
    branches: [main]
    paths: ["services/portfolio/**"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Build and push Docker image
        run: |
          gcloud auth configure-docker ${{ vars.REGION }}-docker.pkg.dev
          docker build -t ${{ vars.REGION }}-docker.pkg.dev/${{ vars.PROJECT_ID }}/stockfolio/portfolio-service:${{ github.sha }} \
            services/portfolio/
          docker push ${{ vars.REGION }}-docker.pkg.dev/${{ vars.PROJECT_ID }}/stockfolio/portfolio-service:${{ github.sha }}

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy portfolio-service \
            --image ${{ vars.REGION }}-docker.pkg.dev/${{ vars.PROJECT_ID }}/stockfolio/portfolio-service:${{ github.sha }} \
            --region ${{ vars.REGION }} \
            --platform managed \
            --no-traffic  # deploy but send no traffic yet

      - name: Run smoke tests
        run: |
          # Hit the health check endpoint of the new revision
          NEW_URL=$(gcloud run revisions describe --region ${{ vars.REGION }} \
            --format='value(status.url)' $(gcloud run revisions list \
            --service portfolio-service --region ${{ vars.REGION }} \
            --format='value(metadata.name)' --limit=1))
          curl --fail "$NEW_URL/health"

      - name: Promote to 100% traffic
        run: |
          gcloud run services update-traffic portfolio-service \
            --to-latest \
            --region ${{ vars.REGION }}
```

### Branch strategy

```
main         ← production, protected. Only merges via PR.
staging      ← mirrors main but deployed to a staging Cloud Run service.
feature/*    ← individual feature branches. PR into main.
```

---

## 15. Out of scope (v1)

The following are explicitly excluded from the first version and can be revisited after beta feedback:

- Mobile app (React Native / Flutter)
- Email or push notifications
- Options, futures, crypto, or international markets
- Social features (sharing portfolios, following friends)
- Brokerage account sync (e.g. Plaid, IBKR API)
- Backtesting the signal score against historical data
- User-configurable signal weights (60/40 is fixed in v1)
- Multi-portfolio per user (one portfolio per user in v1)
- Tax lot tracking or realized gains reporting
- Dark pool or unusual options activity signals