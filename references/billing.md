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

The platform self-hosts the **Agentic Commerce Protocol (ACP)** checkout: the
agent (this skill) buys a credit pack on the user's behalf and pays with a
Stripe Shared Payment Token (SPT). The credit always lands on **the token
owner's** wallet — there is no "buy for someone else".

Packs (NZD, 1:1 into the wallet): `nz5`=500c, `nz20`=2000c, `nz50`=5000c.

Three steps — all authenticated with the same `tn_` token:

```bash
BASE="https://BASE_URL"; AUTH=(-H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json")
PACK="nz5"; AMOUNT=500          # match AMOUNT to the pack: nz5=500 nz20=2000 nz50=5000

# 1) obtain a (test-mode) shared payment token for the amount
SPT=$(curl -s "${AUTH[@]}" -X POST "$BASE/acp/test/mint_spt" -d "{\"amount_cents\":$AMOUNT}" | jq -r .shared_payment_token)

# 2) create the checkout session for the pack
SID=$(curl -s "${AUTH[@]}" -X POST "$BASE/acp/checkout_sessions" -d "{\"pack\":\"$PACK\"}" | jq -r .id)

# 3) complete — Stripe validates the SPT, charges, and the wallet is credited
curl -s "${AUTH[@]}" -X POST "$BASE/acp/checkout_sessions/$SID/complete" -d "{\"shared_payment_token\":\"$SPT\"}"
#   -> {"order_id":"acp_...","status":"completed"}

# verify
curl -s "${AUTH[@]}" "$BASE/api/v1/billing/balance"
```

### Notes / limits
- **Test mode only:** `/acp/test/mint_spt` mints a *test* SPT (test card, no real
  money). It exists so the flow is end-to-end demoable. Real card payment needs a
  real agent-payment integration (not wired yet). For real top-ups today, the
  dashboard's Stripe Checkout flow takes real cards.
- Completing the same session twice is idempotent (no double charge).
- `complete` returns `402` if payment fails, `403` if the session isn't yours.
