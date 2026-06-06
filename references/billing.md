# Billing — balance & buying compute credits

All calls use the user's `tn_…` token as a Bearer header (see [auth.md](auth.md)).
Base path: `https://BASE_URL`. The wallet is **prepaid**: compute (managed
servers/databases) is metered per minute against the balance.

## Check balance & history

```bash
curl -s -H "Authorization: Bearer $TOKEN" "https://BASE_URL/api/v1/billing/balance"   # {"balance_cents": <int>}
curl -s -H "Authorization: Bearer $TOKEN" "https://BASE_URL/api/v1/billing/ledger"     # recent entries
```

## Buy a compute package (auto-purchase)

The agent can top up the wallet in a single API call — no browser, no redirect
— once the user has saved a card and enabled auto-purchase in the dashboard.

### One-time prerequisite (user action, not agent)

The user must do this once in the Tenants dashboard **Billing tab**:
1. Add a card via the "Enable auto-purchase" button.
2. Set a daily spending cap.

The agent cannot do this step. If it hasn't been done, `POST /api/v1/billing/purchase` returns `403`.

### Packages

| Label | `amountCents` |
|-------|--------------|
| $5    | 500          |
| $10   | 1000         |
| $20   | 2000         |

Maximum per-purchase hard cap: **$20 (2000 cents)**. The user also controls a daily cap.

### Purchase call

```bash
# Prereq: user enabled auto-purchase + saved a card once in the dashboard Billing tab.
# Buy a $5 package (off-session, no browser):
curl -s -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -X POST "https://BASE_URL/api/v1/billing/purchase" -d '{"amountCents":500}'
# → {"status":"succeeded","balance_cents":1234}
# If it returns {"status":"action_required","checkout_url":"..."} the user must open that link once (bank 3DS).
# If it returns 403 {"error":"auto-purchase not enabled"} the user hasn't set up a card / enabled it.
```

### Response shapes

| HTTP | Body | Meaning |
|------|------|---------|
| `200` | `{"status":"succeeded","balance_cents":<int>}` | Charged and credited. Done. |
| `200` | `{"status":"action_required","checkout_url":"<url>"}` | Bank requires 3DS — give the user the URL to open once, then retry. |
| `403` | `{"error":"<reason>"}` | Not set up or over cap. Reasons: `"auto-purchase not enabled"`, `"no saved card"`, `"amount exceeds per-purchase cap"`, `"daily cap exceeded"`. |

> **Note:** The old ACP/SPT (`/acp/test/mint_spt`, `/acp/checkout_sessions`) path is disabled and no longer available.
