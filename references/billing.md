# Billing — balance & buying compute credits

All calls use the user's `tn_…` token as a Bearer header (see [auth.md](auth.md)).
Base path: `https://BASE_URL`. The wallet is **prepaid**: compute (managed
servers/databases) is metered per minute against the balance.

## Check balance & history

```bash
curl -s -H "Authorization: Bearer $TOKEN" "https://BASE_URL/api/v1/billing/balance"   # {"balance_cents": <int>}
curl -s -H "Authorization: Bearer $TOKEN" "https://BASE_URL/api/v1/billing/ledger"     # recent entries
```

## Buy compute credits via ACP (agentic checkout)

The platform self-hosts an **Agentic Commerce Protocol (ACP)** checkout so the
agent (this skill) can top up the user's wallet on their behalf. Payment settles
**off-session against the user's saved card** (a card-on-file charge) — there is
no Shared Payment Token and nothing US-gated. The credit always lands on **the
token owner's** wallet; there is no "buy for someone else".

Packs (NZD, 1:1 into the wallet): `nz5`=500c, `nz20`=2000c, `nz50`=5000c.

### Prerequisite: the user must enable "Agent payment" once

Before this flow works, the user has to save a card in the dashboard:
**Billing panel → Agent payment → enable → enter card** (a Stripe SetupIntent,
no charge). That stores a Stripe customer + payment method server-side and turns
on the daily cap. If it isn't set up, `complete` returns **403 "agent payment
not set up"**. This is a one-time action the agent cannot do for them.

### Two steps — both authenticated with the same `tn_` token

```bash
BASE="https://BASE_URL"; AUTH=(-H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json")
PACK="nz5"                       # nz5 | nz20 | nz50

# 1) create the checkout session for the pack
SID=$(curl -s "${AUTH[@]}" -X POST "$BASE/acp/checkout_sessions" -d "{\"pack\":\"$PACK\"}" | jq -r .id)

# 2) complete — charges the saved card off-session and credits the wallet
curl -s "${AUTH[@]}" -X POST "$BASE/acp/checkout_sessions/$SID/complete" -d '{}'
#   -> {"status":"completed","balance_cents":<new balance>}

# verify
curl -s "${AUTH[@]}" "$BASE/api/v1/billing/balance"
```

`complete` takes **no body** — the saved card is charged automatically.

### Outcomes

| Response | Meaning |
|---|---|
| `200 {"status":"completed","balance_cents":N}` | Charged + wallet credited. Replaying `complete` on the same session returns this idempotently (no double charge). |
| `200 {"status":"action_required","checkout_url":"…"}` | The card needs 3‑D Secure. Hand the URL to the user to finish in a browser; the wallet is credited by webhook once they do. |
| `403 {"error":"agent payment not set up"}` | No saved card — user must enable Agent payment first (see prerequisite). |
| `403 {"error":"…cap…"}` | Blocked by a guardrail (see limits below). |
| `403 {"error":"forbidden"}` | The session isn't owned by this token. |
| `502 {"error":"charge failed"}` | Stripe declined / errored. Surface to the user; nothing was credited. |

### Limits / guardrails (enforced server-side)

- **Per-purchase hard cap:** $20 (2000c). A single pack above this is rejected
  regardless of settings — so `nz50` ($50) will be refused. Use `nz5`/`nz20`.
- **Daily cap:** the user-configured per-day limit (default $50) across all
  agent-payment credits in a rolling 24h window. Exceeding it → 403.
- **Minimum charge:** Stripe's floor is $0.50, so all packs clear it.
- Completing the same session twice is idempotent (no double charge).
