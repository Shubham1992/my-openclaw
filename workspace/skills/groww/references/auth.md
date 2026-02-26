# Groww API — Authentication Reference

The Groww Trade API uses a daily session token obtained by posting to the login endpoint with your **API key as the Bearer token**. All subsequent API calls use the returned `ACCESS_TOKEN`.

Tokens expire at **6:00 AM IST** each trading day.

---

## Credential File

Store credentials at `~/.groww/credentials` with strict permissions:

```bash
mkdir -p ~/.groww && chmod 700 ~/.groww
touch ~/.groww/credentials && chmod 600 ~/.groww/credentials
```

File format:
```
API_KEY=<your JWT api key>
SECRET='<your secret>'       # MUST be single-quoted if it contains $, @, #, etc.
TOTP_SEED=                   # optional; enables TOTP auth
ACCESS_TOKEN=                # auto-populated after login
TOKEN_FETCHED_AT=            # epoch seconds; auto-populated
```

> **Important:** Always single-quote the SECRET line in the credentials file. Shell special characters like `$` and `@` are expanded when the file is sourced if unquoted, silently corrupting the value and causing "Invalid credentials" errors.

Load with:
```bash
source ~/.groww/credentials
```

---

## Login Endpoint

```
POST https://api.groww.in/v1/token/api/access
Authorization: Bearer <API_KEY>
Content-Type: application/json
X-API-VERSION: 1.0
```

The API key itself is the Bearer token in the login request. Two `key_type` values are supported.

---

## Option A — API Key + Secret (checksum)

The checksum is `SHA256(secret + timestamp)` where timestamp is current epoch seconds (10 digits).

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

if [ -z "$TOKEN" ] || [ "$TOKEN" = "null" ]; then
  echo "Login failed: $(echo "$RESPONSE" | jq -r '.error.errorMessage // .message')"
  exit 1
fi

sed -i "s|^ACCESS_TOKEN=.*|ACCESS_TOKEN=$TOKEN|" ~/.groww/credentials
sed -i "s|^TOKEN_FETCHED_AT=.*|TOKEN_FETCHED_AT=$(date +%s)|" ~/.groww/credentials
echo "Authenticated. Token valid until 6 AM IST."
```

---

## Option B — TOTP Login

Requires `oathtool` (`sudo apt install oathtool`) and a TOTP seed configured in Groww.

```bash
source ~/.groww/credentials
TOTP=$(oathtool --base32 --totp "$TOTP_SEED")

RESPONSE=$(curl -s -X POST "https://api.groww.in/v1/token/api/access" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-API-VERSION: 1.0" \
  -d "{\"key_type\":\"totp\",\"totp\":\"$TOTP\"}")

TOKEN=$(echo "$RESPONSE" | jq -r '.token')

if [ -z "$TOKEN" ] || [ "$TOKEN" = "null" ]; then
  echo "TOTP login failed: $(echo "$RESPONSE" | jq -r '.error.errorMessage // .message')"
  exit 1
fi

sed -i "s|^ACCESS_TOKEN=.*|ACCESS_TOKEN=$TOKEN|" ~/.groww/credentials
sed -i "s|^TOKEN_FETCHED_AT=.*|TOKEN_FETCHED_AT=$(date +%s)|" ~/.groww/credentials
echo "Authenticated via TOTP."
```

**Python fallback** for TOTP generation if `oathtool` is unavailable:
```bash
TOTP=$(python3 -c "import pyotp; print(pyotp.TOTP('$TOTP_SEED').now())")
```

---

## Successful Login Response

```json
{
  "token": "<ACCESS_TOKEN>",
  "tokenRefId": "3365a2b2-...",
  "sessionName": "firstapikey",
  "expiry": "2026-02-27T06:00:00",
  "active": true
}
```

---

## Token Caching Logic

Before any API call, check whether a cached token is still valid:

```bash
source ~/.groww/credentials

TODAY_6AM_UTC=$(date -d "$(date +%Y-%m-%d) 00:30:00 UTC" +%s 2>/dev/null \
  || date -jf "%Y-%m-%d %H:%M:%S" "$(date +%Y-%m-%d) 00:30:00" +%s)

NEEDS_REFRESH=false
[ -z "$ACCESS_TOKEN" ] || [ "$ACCESS_TOKEN" = "null" ] && NEEDS_REFRESH=true
[ -z "$TOKEN_FETCHED_AT" ] || [ "$TOKEN_FETCHED_AT" -lt "$TODAY_6AM_UTC" ] && NEEDS_REFRESH=true

if [ "$NEEDS_REFRESH" = "true" ]; then
  echo "Token missing or expired — re-authenticating..."
  # Run Option A or B block above
fi
```

---

## 401 Handling

If any API call returns HTTP 401, re-authenticate and retry once:

```bash
source ~/.groww/credentials
HTTP_CODE=$(curl -s -o /tmp/groww_resp.json -w "%{http_code}" \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-API-VERSION: 1.0" \
  https://api.groww.in/v1/holdings/user)

if [ "$HTTP_CODE" = "401" ]; then
  echo "Token expired — refreshing..."
  # Re-run login block
  source ~/.groww/credentials
  # Retry the original request
fi

cat /tmp/groww_resp.json | jq .
```

---

## Security Notes

- Never commit `~/.groww/credentials` to git.
- Always single-quote the SECRET in the credentials file.
- Rotate the SECRET from the Groww Trade API dashboard if compromised.
- TOTP seed grants full account access — treat it like a password.
