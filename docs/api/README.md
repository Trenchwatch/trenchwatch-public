# Trenchwatch API · v1 (api_version 2026-10)

Trenchwatch tells your product, in milliseconds, whether a new Solana token or address is linked to a known group of coordinated wallets, and pushes an alert to your systems seconds after a launch whose early buying is not organic. This guide covers everything you need to integrate: authentication, the sandbox, the four REST endpoints and the signed webhook.

> Alerts are probabilistic observations based on public on-chain data. They describe addresses and patterns, not people, and are not financial advice. Using alerts to trade alongside the addresses they describe is not permitted under the licence.

---

## 1. Environments

| Environment | Base URL | Keys | Data | Availability |
| --- | --- | --- | --- | --- |
| Sandbox | `https://sandbox-api.trenchwatch.dev/v1` | `tw_test_…` | Synthetic and fixed, flagged `"sandbox": true`. Addresses do not exist on-chain | Available now. Test keys on request |
| Production | Provided with your licence | `tw_live_…` | Live detections | Licensed partners |

A test key never works in production and a live key never works in the sandbox (`401`). Start in the sandbox: it returns the same shapes as production, and `GET /v1/health` lists example addresses for every case.

To request a sandbox key, write to info@trenchwatch.dev.

## 2. Authentication and limits

Send your key in the `X-API-Key` header on every request except `/v1/health`.

```bash
curl -H "X-API-Key: tw_test_XXXXXXXX" https://sandbox-api.trenchwatch.dev/v1/token/<mint>
```

- Keys are shown once when issued; we store only a hash. If a key leaks, ask for a new one and the old one is revoked.
- Rate limit: 600 requests per minute per key (higher on request). Over the limit: `429`.
- Every response carries the header `X-Trenchwatch-Api-Version: 2026-10` and a `took_ms` field.

## 3. Endpoints

### `GET /v1/health`

No key needed. Service status. In the sandbox it also returns example addresses to try.

```json
{"ok": true, "mode": "sandbox", "api_version": "2026-10",
 "examples": {"tokens": {"HIGH": "5WFk…NqNi", "ELEVATED": "Cwy…oq1f", "LOW": "5jsh…eP1p"},
              "operators": {"TW-9001": "HPGg…sJj8", "TW-9002": "Ubh…QUV"},
              "coordinated_wallet": "5JYE…o21p"}}
```

### `GET /v1/token/{mint}`

Risk of a token, asked before a swap, a listing or a promotion.

```json
{
  "mint": "5WFkboKPeci8vJbCBVicpqgDvFC7j8wSQ2vFxQuzNqNi",
  "risk": "HIGH",
  "created_at": 1791770343,
  "creator": "HPGgF4zjejv3ogpgXnsqM454RHgtCHuHTSoNvauZsJj8",
  "creator_operator": {"id": "TW-9001", "level": "confirmed", "first_seen": "2026-10-01 00:00:00",
                       "last_seen": "2026-10-12 02:09:03", "launches_per_hour": 12.0},
  "linked_operators_buying": [],
  "signals": ["early buyers from wallets linked to this creator", "sells within the first seconds", "repeated launches by this creator"],
  "early_buyers": 9,
  "early_sol": 4.2,
  "sandbox": true,
  "evidence_note": "SANDBOX DATA. Addresses are synthetic and do not exist on-chain.",
  "took_ms": 0.3
}
```

| Field | Meaning |
| --- | --- |
| `risk` | `HIGH`, `ELEVATED`, `LOW`, or `UNKNOWN` (token not seen by Trenchwatch) |
| `creator_operator` | The creator's entry in the Trenchwatch registry of addresses with repeated launch activity, or `null` |
| `linked_operators_buying` | Other registry entries whose linked wallets bought early |
| `signals` | Plain-language reasons, for display or logs |
| `early_buyers`, `early_sol` | Distinct buyers and SOL in the first seconds after creation |

Suggested use: `HIGH` → show a warning before the swap or hold the listing; `ELEVATED` → label the token; `LOW` / `UNKNOWN` → no action.

### `GET /v1/operator/{address}`

Is this address in the registry, or one of the wallets linked to a registry entry?

```json
{"address": "5JYE9C44QkhczUApEt4YJvoERXS37e8FMdT8nBsLo21p", "role": "coordinated_wallet",
 "operator": {"id": "TW-9001", "level": "confirmed", "first_seen": "2026-10-01 00:00:00",
              "last_seen": "2026-10-12 02:09:03", "launches_per_hour": 12.0}}
```

`role` is `operator`, `coordinated_wallet` or `unknown`.

### `GET /v1/alerts?since=<unix>&limit=<n>`

The latest alerts, newest first, for teams that prefer polling to a webhook. `since` defaults to one hour ago; `limit` defaults to 100, maximum 1000.

```json
{"alerts": [{"ts": 1791770940, "source": "watcher", "type": "operator_cluster_buy", "level": "COORDINATED",
             "token": "5WFkboKPeci8vJbCBVicpqgDvFC7j8wSQ2vFxQuzNqNi",
             "operator": {"id": "TW-9001", "level": "confirmed", "launches_per_hour": 12.0}}]}
```

## 4. Registry IDs, levels and alert types

**Registry IDs** (`TW-0001`…) are stable: the same entry keeps its ID across wallets and launchpads. Registry `level`: `confirmed` (verified after review), `probable` (linked wallets found), `serial` (many launches, no linked wallets found yet), `repeat`.

Field and type names such as `operator` are technical identifiers kept stable for compatibility. They refer to addresses and patterns, not to people.

**Evidence levels** are categories, not scores:

| Level | Meaning |
| --- | --- |
| `RELATED` | A transactional relationship was observed. On its own it does not prove common control |
| `COORDINATED` | Coordinated technical or timing behaviour between linked wallets |
| `SUSPECTED` | Several independent signals point to non-organic demand |
| `CONFIRMED` | Assigned only after external verification, never automatically |

**Alert types** (stable codes; new ones may be added, so ignore codes you do not know):

| `type` | What happened | Typical action |
| --- | --- | --- |
| `operator_cluster_buy` | Several wallets linked to one registry entry bought a token in its first seconds | Warn users before they buy; hold promotion |
| `coordinated_support` | Linked wallets bought a token created by the same registry entry | Warn users; review the listing |
| `operator_launch` | A registry entry created a token | Label the token |
| `linked_wallet_buy` | A linked wallet bought | Context for other alerts |
| `funding_batch` | A registry entry funded a set of fresh wallets at once | Add them to your watchlist |
| `linked_wallet` | A new wallet was linked to a registry entry | Treat it as linked in your risk rules |
| `profit_sweep` | Proceeds were consolidated to a common address | Review recent launches by that entry |
| `exit_route` | Proceeds converged on a common destination | Context |

## 5. Webhook

With a licence we push every alert to an HTTPS endpoint you choose. Your endpoint must answer `2xx` within 5 seconds; do the heavy work afterwards.

**Request**

```
POST /your/endpoint
Content-Type: application/json
User-Agent: Trenchwatch-Webhook/1
X-Trenchwatch-Timestamp: 1791770943
X-Trenchwatch-Signature: 6f1c…(hex)
```

```json
{
  "api_version": "2026-10",
  "id": "A-20261012-000145",
  "type": "operator_cluster_buy",
  "level": "COORDINATED",
  "detected_at_utc": "2026-10-12 02:00:00",
  "token": "<mint>",
  "operator": {"id": "TW-0145", "address": "<address>"},
  "summary": "<plain-language description of what was observed>",
  "evidence": [{"step": "buy", "wallet": "<address>", "s after launch": 2, "transaction": "<signature>"}],
  "transaction": "<signature>",
  "sandbox": false
}
```

**Verify every request.** The signature is the hex HMAC-SHA256 of `"<timestamp>.<raw body>"` with your webhook secret. Reject requests whose timestamp is more than 5 minutes old, and compare in constant time.

Python:

```python
import hmac, hashlib, time

def is_valid(secret: str, timestamp: str, raw_body: bytes, signature: str) -> bool:
    if abs(time.time() - int(timestamp)) > 300:
        return False
    expected = hmac.new(secret.encode(), timestamp.encode() + b"." + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)
```

Node.js:

```js
const crypto = require("crypto");

function isValid(secret, timestamp, rawBody, signature) {
  if (Math.abs(Date.now() / 1000 - Number(timestamp)) > 300) return false;
  const expected = crypto.createHmac("sha256", secret).update(`${timestamp}.${rawBody}`).digest("hex");
  return expected.length === signature.length &&
         crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
}
```

Ready-made verifiers for TypeScript, Python and Rust are in the open standard: https://github.com/Trenchwatch/solana-launch-alerts

**Retries and duplicates.** If your endpoint does not answer `2xx`, we retry 3 times over about 10 minutes (after 30 s, 2 min and 7.5 min). A retried alert keeps the same `id`: use it to ignore duplicates.

**Test event.** Ask us for a test event at any time: it arrives signed with your secret, with `"sandbox": true` and `"type": "operator_cluster_buy"`.

## 6. Errors

| Status | Meaning |
| --- | --- |
| `401` | Missing or invalid key, or a test key in production (or the reverse) |
| `404` | Unknown path |
| `429` | Rate limit exceeded |
| `500` | Internal error; retry with backoff and tell us if it persists |

Errors return `{"error": "<message>"}`.

## 7. Versioning and changes

- The path version (`/v1`) changes only for breaking changes, announced at least 90 days ahead.
- Within v1 we may add fields, alert types and endpoints. Your integration should ignore what it does not recognise.

| Date | Change |
| --- | --- |
| 2026-10-12 | v1: token, operator, alerts and health endpoints; signed webhook with timestamp; sandbox |
| 2026-11-20 | Public sandbox at `sandbox-api.trenchwatch.dev` |

Contact: info@trenchwatch.dev · https://trenchwatch.dev
