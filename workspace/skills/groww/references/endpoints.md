# Groww Trade API — Endpoint Reference

**Base URL:** `https://api.groww.in/v1`

**Required headers on every request:**
```
Accept: application/json
Authorization: Bearer $ACCESS_TOKEN
X-API-VERSION: 1.0
```

---

## Authentication

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/token/api/access` | Login with API key + secret or TOTP |

### POST /v1/token/api/access — secret (checksum)

```bash
source ~/.groww/credentials
TIMESTAMP=$(date +%s)
CHECKSUM=$(echo -n "${SECRET}${TIMESTAMP}" | sha256sum | awk '{print $1}')

curl -s -X POST "https://api.groww.in/v1/token/api/access" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-API-VERSION: 1.0" \
  -d "{\"key_type\":\"approval\",\"checksum\":\"$CHECKSUM\",\"timestamp\":\"$TIMESTAMP\"}" | jq .
```

Checksum formula: `SHA256(secret + timestamp_string)` — lowercase hex digest.

### POST /v1/token/api/access — TOTP

```bash
source ~/.groww/credentials
TOTP=$(oathtool --base32 --totp "$TOTP_SEED")

curl -s -X POST "https://api.groww.in/v1/token/api/access" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-API-VERSION: 1.0" \
  -d "{\"key_type\":\"totp\",\"totp\":\"$TOTP\"}" | jq .
```

---

## Portfolio

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/holdings/user` | DEMAT holdings (long-term equity) |
| GET | `/v1/positions/user` | All positions (intraday/short-term) |
| GET | `/v1/positions/user?segment=CASH` | Positions filtered by segment |
| GET | `/v1/positions/trading-symbol` | Position for a specific symbol |

### GET /v1/holdings/user

```bash
source ~/.groww/credentials
curl -s -X GET https://api.groww.in/v1/holdings/user \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

Response: `{ "status": "SUCCESS", "payload": { "holdings": [ ... ] } }`

### GET /v1/positions/user

```bash
source ~/.groww/credentials
curl -s -X GET https://api.groww.in/v1/positions/user \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

Optional query param: `segment=CASH|FNO|COMMODITY`

### GET /v1/positions/trading-symbol

```bash
source ~/.groww/credentials
curl -s -X GET "https://api.groww.in/v1/positions/trading-symbol?trading_symbol=RELIANCE&segment=CASH" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

---

## Orders

All order mutation endpoints use `POST` (not PUT/DELETE).

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/order/list` | All orders today |
| GET | `/v1/order/status/{groww_order_id}` | Order status |
| GET | `/v1/order/status/reference/{order_reference_id}` | Status by reference ID |
| GET | `/v1/order/detail/{groww_order_id}` | Full order detail |
| GET | `/v1/order/trades/{groww_order_id}` | Trades executed for an order |
| POST | `/v1/order/create` | Place a new order |
| POST | `/v1/order/modify` | Modify an open order |
| POST | `/v1/order/cancel` | Cancel an open order |

### GET /v1/order/list

```bash
source ~/.groww/credentials
curl -s -X GET https://api.groww.in/v1/order/list \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

### GET /v1/order/status/{groww_order_id}

```bash
source ~/.groww/credentials
curl -s -X GET "https://api.groww.in/v1/order/status/GMK39038RDT490CCVRO" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

### POST /v1/order/create

**Required fields:**

| Field | Type | Values |
|-------|------|--------|
| `trading_symbol` | string | e.g. `RELIANCE`, `WIPRO` |
| `exchange` | string | `NSE`, `BSE`, `NFO`, `MCX` |
| `segment` | string | `CASH`, `FNO`, `COMMODITY` |
| `product` | string | `CNC` (delivery), `MIS` (intraday), `NRML` (F&O overnight) |
| `transaction_type` | string | `BUY`, `SELL` |
| `order_type` | string | `MARKET`, `LIMIT`, `SL`, `SL-M` |
| `quantity` | integer | positive integer |
| `validity` | string | `DAY`, `IOC` |

**Optional fields:** `price` (required for LIMIT/SL), `trigger_price` (required for SL/SL-M), `order_reference_id` (custom tag, max 20 chars)

**Market buy (delivery):**
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

**Limit sell (intraday):**
```bash
source ~/.groww/credentials
curl -s -X POST https://api.groww.in/v1/order/create \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" \
  -d '{
    "trading_symbol": "INFY",
    "exchange": "NSE",
    "segment": "CASH",
    "product": "MIS",
    "transaction_type": "SELL",
    "order_type": "LIMIT",
    "quantity": 2,
    "price": 1450.00,
    "validity": "DAY"
  }' | jq .
```

**Stop-loss order:**
```bash
source ~/.groww/credentials
curl -s -X POST https://api.groww.in/v1/order/create \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" \
  -d '{
    "trading_symbol": "WIPRO",
    "exchange": "NSE",
    "segment": "CASH",
    "product": "CNC",
    "transaction_type": "BUY",
    "order_type": "SL",
    "quantity": 100,
    "price": 2500.00,
    "trigger_price": 2450.00,
    "validity": "DAY"
  }' | jq .
```

### POST /v1/order/cancel

```bash
source ~/.groww/credentials
curl -s -X POST https://api.groww.in/v1/order/cancel \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" \
  -d '{
    "segment": "CASH",
    "groww_order_id": "GMK39038RDT490CCVRO"
  }' | jq .
```

### POST /v1/order/modify

```bash
source ~/.groww/credentials
curl -s -X POST https://api.groww.in/v1/order/modify \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" \
  -d '{
    "groww_order_id": "GMK39038RDT490CCVRO",
    "segment": "CASH",
    "order_type": "LIMIT",
    "quantity": 100,
    "price": 2500.00,
    "trigger_price": 2450.00
  }' | jq .
```

---

## Live Market Data

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/live-data/quote` | Full snapshot for one instrument |
| GET | `/v1/live-data/ltp` | Last traded price, up to 50 instruments |
| GET | `/v1/live-data/ohlc` | OHLC today, up to 50 instruments |

Batch endpoints use `exchange_symbols` in the format `EXCHANGE_SYMBOL` (e.g. `NSE_RELIANCE`).

### GET /v1/live-data/quote

```bash
source ~/.groww/credentials
curl -s -X GET "https://api.groww.in/v1/live-data/quote?exchange=NSE&segment=CASH&trading_symbol=RELIANCE" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

Required params: `exchange`, `segment`, `trading_symbol`

### GET /v1/live-data/ltp (batch)

```bash
source ~/.groww/credentials
curl -s -X GET "https://api.groww.in/v1/live-data/ltp?segment=CASH&exchange_symbols=NSE_RELIANCE,NSE_INFY,NSE_TCS" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

### GET /v1/live-data/ohlc (batch)

```bash
source ~/.groww/credentials
curl -s -X GET "https://api.groww.in/v1/live-data/ohlc?segment=CASH&exchange_symbols=NSE_RELIANCE,NSE_INFY" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

---

## Historical Data

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/historical/candle/range` | OHLCV candles for a date/time range |

### GET /v1/historical/candle/range

```bash
source ~/.groww/credentials
curl -s -X GET "https://api.groww.in/v1/historical/candle/range?exchange=NSE&segment=CASH&trading_symbol=RELIANCE&start_time=2024-01-01%2009:15:00&end_time=2024-01-31%2015:30:00&interval_in_minutes=60" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" | jq .
```

**Parameters:**

| Param | Required | Format/Values |
|-------|----------|---------------|
| `exchange` | yes | `NSE`, `BSE`, `NFO`, `MCX` |
| `segment` | yes | `CASH`, `FNO`, `COMMODITY` |
| `trading_symbol` | yes | exchange-defined ticker |
| `start_time` | yes | `yyyy-MM-dd HH:mm:ss` or epoch seconds |
| `end_time` | yes | `yyyy-MM-dd HH:mm:ss` or epoch seconds |
| `interval_in_minutes` | no | `1`, `5`, `15`, `30`, `60`, `1440` (daily) |

Response: arrays of `[timestamp, open, high, low, close, volume]`

---

## Error Codes

| HTTP Code | Meaning | Action |
|-----------|---------|--------|
| 200 | Success | — |
| 400 | Bad request / invalid params | Check request body |
| 401 | Unauthorized / token expired | Re-authenticate |
| 403 | Forbidden | Check API subscription status |
| 404 | Not found | Check order ID or symbol |
| 429 | Rate limited | Wait 5s, retry |
| 500 | Server error | Retry once |

---

## Rate Limits

| Endpoint group | Per-second | Per-minute |
|----------------|-----------|-----------|
| Orders         | 10 req/s  | 250/min   |
| Market data    | 10 req/s  | 300/min   |
| Portfolio      | 5 req/s   | 100/min   |
| Authentication | 1 req/s   | 10/min    |
