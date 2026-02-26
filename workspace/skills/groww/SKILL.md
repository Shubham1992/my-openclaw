---
name: groww
description: Interact with a Groww trading account via the Groww Trade API. Use when asked about portfolio holdings, P&L, positions, placing/cancelling/modifying equity or F&O orders, or fetching live market prices and quotes. Requires a Groww Trade API subscription (₹499/month) and credentials stored in ~/.groww/credentials.
read_when:
  - Checking Groww portfolio or holdings
  - Placing or cancelling a stock order on Groww
  - Fetching live stock price or market data from Groww
  - Checking Groww order status or trade history
metadata: {"clawdbot":{"emoji":"📈","requires":{"bins":["curl","jq"]}}}
allowed-tools: Bash(curl:*), Bash(jq:*), Bash(cat:*), Bash(echo:*)
---

# Groww Trade API Skill

Interact with the user's Groww trading account — portfolio, orders, and live market data — using the official Groww Trade API via `curl` and `jq`.

See [references/auth.md](references/auth.md) for detailed authentication flows and token caching.
See [references/endpoints.md](references/endpoints.md) for the full endpoint reference.

**Base URL:** `https://api.groww.in/v1`

**Required headers on every request:**
```
Accept: application/json
Authorization: Bearer $ACCESS_TOKEN
X-API-VERSION: 1.0
```

---

## 1. Setup

**Subscribe:** https://groww.in/trade-api (₹499/month)

**Credential file** — `~/.groww/credentials` (chmod 600):

```
API_KEY=<your JWT api key>
SECRET='<your secret>'       # single-quote to prevent shell expansion of special chars
TOTP_SEED=                   # optional
ACCESS_TOKEN=                # auto-populated after login
TOKEN_FETCHED_AT=            # epoch seconds; auto-populated
```

> Always single-quote the SECRET value if it contains special characters like `$`, `@`, `#`.

```bash
mkdir -p ~/.groww && chmod 700 ~/.groww
touch ~/.groww/credentials && chmod 600 ~/.groww/credentials
```

---

## 2. Authentication

The API key itself is used as the Bearer token **in the login request**. The login returns a short-lived `ACCESS_TOKEN` valid until 6 AM IST daily.

### Login with API key + secret (checksum)

```bash
source ~/.groww/credentials
TIMESTAMP=$(date +%s)
CHECKSUM=$(echo -n "${SECRET}${TIMESTAMP}" | sha256sum | awk '{print $1}')

RESPONSE=$(curl -s -X POST "https://api.groww.in/v1/token/api/access" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-API-VERSION: 1.0" \
  -d "{\"key_type\":\"approval\",\"checksum\":\"$CHECKSUM\",\"timestamp\":\"$TIMESTAMP\"}")

echo "$RESPONSE" | jq .
TOKEN=$(echo "$RESPONSE" | jq -r '.token')

# Cache token
sed -i "s|^ACCESS_TOKEN=.*|ACCESS_TOKEN=$TOKEN|" ~/.groww/credentials
sed -i "s|^TOKEN_FETCHED_AT=.*|TOKEN_FETCHED_AT=$(date +%s)|" ~/.groww/credentials
```

### Login with TOTP (requires oathtool)

```bash
source ~/.groww/credentials
TOTP=$(oathtool --base32 --totp "$TOTP_SEED")

RESPONSE=$(curl -s -X POST "https://api.groww.in/v1/token/api/access" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-API-VERSION: 1.0" \
  -d "{\"key_type\":\"totp\",\"totp\":\"$TOTP\"}")

TOKEN=$(echo "$RESPONSE" | jq -r '.token')
sed -i "s|^ACCESS_TOKEN=.*|ACCESS_TOKEN=$TOKEN|" ~/.groww/credentials
sed -i "s|^TOKEN_FETCHED_AT=.*|TOKEN_FETCHED_AT=$(date +%s)|" ~/.groww/credentials
```

> If a request returns 401, re-run the login block then retry.

---

## 3. Portfolio & Holdings

```bash
source ~/.groww/credentials

# Holdings (DEMAT / long-term equity)
curl -s -X GET https://api.groww.in/v1/holdings/user \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .

# All positions
curl -s -X GET https://api.groww.in/v1/positions/user \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .

# Positions filtered by segment (CASH | FNO | COMMODITY)
curl -s -X GET "https://api.groww.in/v1/positions/user?segment=CASH" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .

# Position for a specific symbol
curl -s -X GET "https://api.groww.in/v1/positions/trading-symbol?trading_symbol=RELIANCE&segment=CASH" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

---

## 4. Order Management

> **Safety rule:** Before placing any order, always fetch the current live price and confirm all parameters with the user.

### List & inspect orders

```bash
source ~/.groww/credentials

# All orders today
curl -s -X GET https://api.groww.in/v1/order/list \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .

# Order status by ID
curl -s -X GET "https://api.groww.in/v1/order/status/<groww_order_id>" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .

# Order detail (full)
curl -s -X GET "https://api.groww.in/v1/order/detail/<groww_order_id>" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .

# Trades for an order
curl -s -X GET "https://api.groww.in/v1/order/trades/<groww_order_id>" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

### Place order

```bash
source ~/.groww/credentials
curl -s -X POST https://api.groww.in/v1/order/create \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" \
  -d '{
    "trading_symbol": "RELIANCE",
    "exchange": "NSE",
    "segment": "CASH",
    "product": "CNC",
    "transaction_type": "BUY",
    "order_type": "MARKET",
    "quantity": 1,
    "validity": "DAY"
  }' | jq .
```

For a limit order add `"price": <value>`; for SL orders also add `"trigger_price": <value>`.

### Cancel order

```bash
source ~/.groww/credentials
curl -s -X POST https://api.groww.in/v1/order/cancel \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" \
  -d '{
    "segment": "CASH",
    "groww_order_id": "<groww_order_id>"
  }' | jq .
```

### Modify order

```bash
source ~/.groww/credentials
curl -s -X POST https://api.groww.in/v1/order/modify \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" \
  -d '{
    "groww_order_id": "<groww_order_id>",
    "segment": "CASH",
    "order_type": "LIMIT",
    "quantity": 2,
    "price": 1450.00
  }' | jq .
```

---

## 5. Market Data

```bash
source ~/.groww/credentials

# Full live quote (single symbol)
curl -s -X GET "https://api.groww.in/v1/live-data/quote?exchange=NSE&segment=CASH&trading_symbol=RELIANCE" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .

# LTP for multiple symbols (up to 50, format: EXCHANGE_SYMBOL)
curl -s -X GET "https://api.groww.in/v1/live-data/ltp?segment=CASH&exchange_symbols=NSE_RELIANCE,NSE_INFY,NSE_TCS" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .

# OHLC for multiple symbols (up to 50)
curl -s -X GET "https://api.groww.in/v1/live-data/ohlc?segment=CASH&exchange_symbols=NSE_RELIANCE,NSE_INFY" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .

# Historical candles
curl -s -X GET "https://api.groww.in/v1/historical/candle/range?exchange=NSE&segment=CASH&trading_symbol=RELIANCE&start_time=2024-01-01%2009:15:00&end_time=2024-01-31%2015:30:00&interval_in_minutes=60" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

---

## 6. Rate Limits & Safety

| Endpoint group | Per-second | Per-minute |
|----------------|-----------|-----------|
| Orders         | 10 req/s  | 250/min   |
| Market data    | 10 req/s  | 300/min   |
| Portfolio      | 5 req/s   | 100/min   |

**Agent rules:**
- Never place an order without explicit user confirmation of all parameters.
- Always show current market price before placing a trade.
- On 401, re-authenticate and retry once.
- On 429, wait 5 seconds before retrying.
- Never store the access token anywhere other than `~/.groww/credentials`.
