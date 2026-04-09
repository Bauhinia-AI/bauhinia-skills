---
name: tradingnews-api
description: |
  Use the TradingNews API to fetch real-time financial news for trading bots, market analysis, or portfolio monitoring. Trigger this skill when the user wants to: get breaking financial news, build a news-driven trading bot, monitor market-moving events, integrate a financial news feed, fetch news by ticker symbol, stream real-time market news via WebSocket, or connect to TradingNews in any way. Also trigger when code imports or references tradingnews, or when the user mentions "trading news API", "financial news feed", "market news", or "news for trading".
---

# TradingNews API Integration Guide

TradingNews provides a unified, real-time financial news API. It aggregates breaking news from multiple curated sources into a single structured stream. This skill teaches you how to integrate it.

## Base URL

```
https://api.tradingnews.press
```

## Authentication

All requests require an API key via the `X-API-Key` header:

```bash
curl -H "X-API-Key: YOUR_KEY" https://api.tradingnews.press/v1/news
```

API keys are created in the [TradingNews Dashboard](https://tradingnews.press/dashboard/keys) after subscribing.

## Plans

| Plan | Price | REST Rate Limit | WebSocket |
|------|-------|----------------|-----------|
| Starter | $20/mo | 1 req/min | No |
| Pro | $50/mo | 30 req/min | Yes (1 connection) |

---

## REST API

### GET /v1/news — List articles

```bash
curl -H "X-API-Key: YOUR_KEY" \
  "https://api.tradingnews.press/v1/news?limit=5&ticker=AAPL&urgency=breaking"
```

**Query parameters:**
- `limit` (int, 1-100, default 50) — number of articles
- `ticker` (string, optional) — filter by ticker symbol (e.g., AAPL)
- `urgency` (string, optional) — filter: `breaking`, `flash`, or `regular`
- `since` (ISO datetime, optional) — articles after this time (Starter: 24h max, Pro: 7 days max)

**Response:**
```json
{
  "count": 2,
  "articles": [
    {
      "id": "01KNKV33C9DDADR1HRRWGCSA8V",
      "content": "Fed holds rates steady at 4.25-4.50%, signals patience on cuts",
      "tickers": ["SPY", "QQQ"],
      "urgency": "breaking",
      "sentiment": null,
      "published_at": "2026-04-07T14:30:00+00:00",
      "received_at": "2026-04-07T14:30:02+00:00"
    }
  ]
}
```

**Article fields:**
| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique ULID, sortable by time |
| `content` | string | Article text |
| `tickers` | string[] | Associated stock tickers |
| `urgency` | string | `breaking`, `flash`, or `regular` |
| `sentiment` | string? | Sentiment label (when available) |
| `published_at` | string | ISO 8601 publication time |
| `received_at` | string | ISO 8601 time API received it |

### GET /v1/news/{id} — Single article

```bash
curl -H "X-API-Key: YOUR_KEY" \
  "https://api.tradingnews.press/v1/news/01KNKV33C9DDADR1HRRWGCSA8V"
```

Returns 404 if not found (no charge for 404s).

### GET /v1/account/usage — Check your plan

```bash
curl -H "X-API-Key: YOUR_KEY" \
  "https://api.tradingnews.press/v1/account/usage"
```

**Response:**
```json
{
  "tier": "pro",
  "rate_limit": 30,
  "period_end": "2026-05-07T00:00:00+00:00"
}
```

This endpoint is free (does not count toward rate limit).

---

## WebSocket Streaming (Pro only)

Connect for real-time push delivery of all articles as they arrive:

```
wss://api.tradingnews.press/v1/stream?api_key=YOUR_KEY
```

- **No subscription message needed** — connect and receive (firehose)
- **Pro plan required** — Starter keys get rejected (code 1008)
- **1 connection per user** — second connection is rejected
- Articles arrive as JSON, same format as REST

### Python example

```python
import asyncio
import json
import websockets

async def stream():
    uri = "wss://api.tradingnews.press/v1/stream?api_key=YOUR_KEY"
    async with websockets.connect(uri) as ws:
        async for message in ws:
            article = json.loads(message)
            if article["urgency"] == "breaking":
                print(f"BREAKING: {article['content']}")
                print(f"Tickers: {article['tickers']}")

asyncio.run(stream())
```

### Reconnection pattern

```python
async def stream_with_reconnect():
    backoff = 1
    while True:
        try:
            async with websockets.connect(uri) as ws:
                backoff = 1
                async for msg in ws:
                    process(json.loads(msg))
        except websockets.ConnectionClosed:
            await asyncio.sleep(backoff)
            backoff = min(backoff * 2, 60)
```

---

## Error Handling

| Status | Meaning | Action |
|--------|---------|--------|
| 401 | Invalid or missing API key | Check `X-API-Key` header |
| 403 | Subscription expired or inactive key | Renew subscription |
| 404 | Article not found | Normal — no charge |
| 429 | Rate limit exceeded | Wait and retry |

---

## Python Integration Example

```python
import httpx

client = httpx.Client(
    base_url="https://api.tradingnews.press",
    headers={"X-API-Key": "YOUR_KEY"},
)

# Get latest breaking news
response = client.get("/v1/news", params={"urgency": "breaking", "limit": 10})
articles = response.json()["articles"]

for article in articles:
    print(f"[{article['urgency']}] {article['content']}")
    if article["tickers"]:
        print(f"  Tickers: {', '.join(article['tickers'])}")
```

## Node.js Integration Example

```javascript
const response = await fetch("https://api.tradingnews.press/v1/news?limit=10", {
  headers: { "X-API-Key": "YOUR_KEY" },
});
const { articles } = await response.json();

articles.forEach((a) => {
  console.log(`[${a.urgency}] ${a.content}`);
});
```

---

## Common Patterns

### Poll for new articles (Starter)
```python
import time
last_seen = None

while True:
    params = {"limit": 10}
    if last_seen:
        params["since"] = last_seen
    
    resp = client.get("/v1/news", params=params)
    articles = resp.json()["articles"]
    
    for article in articles:
        process(article)
        last_seen = article["received_at"]
    
    time.sleep(60)  # 1 req/min for Starter
```

### Filter by multiple tickers
The API filters by one ticker at a time. For multiple tickers, make separate calls or fetch all and filter client-side:

```python
# Client-side filtering
resp = client.get("/v1/news", params={"limit": 100})
my_tickers = {"AAPL", "TSLA", "NVDA"}
relevant = [a for a in resp.json()["articles"] if set(a["tickers"]) & my_tickers]
```

### News-driven trading signal
```python
async for msg in ws:
    article = json.loads(msg)
    if article["urgency"] == "breaking" and "AAPL" in article["tickers"]:
        # Execute trade logic
        execute_order("AAPL", signal_from_content(article["content"]))
```

---

## Docs & Support

- **Full API docs**: https://docs.tradingnews.press
- **Dashboard**: https://tradingnews.press/dashboard
- **Status**: https://tradingnews.press/status
