# Token Authentication

Every API request carries the user's personal access token as a Bearer header.

## Get a token

1. User opens the Tenants dashboard at `https://{BASE_URL}/dashboard`.
2. They click the **Skill** tab → **Generate token**.
3. The dashboard shows a `tn_…` string exactly once. The user copies it and pastes it into the chat with the agent.

## Use the token

Store it in a variable at the start of the session:

```bash
TOKEN="tn_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

Every API call:

```bash
curl -s -H "Authorization: Bearer $TOKEN" "https://BASE_URL/api/v1/managed-servers"
```

## Verify the token works

```bash
curl -sf -H "Authorization: Bearer $TOKEN" "https://BASE_URL/auth/me" >/dev/null \
  && echo "OK" || echo "Token invalid — ask user to regenerate in dashboard"
```

The backend returns 401 if the token is missing, malformed, or revoked.

## Revoke / rotate

From the dashboard Skill panel the user can **Revoke** (stop the token working) or **Regenerate** (revoke + create a new one in one click). Regeneration is the right flow when the token may have leaked.

## Troubleshooting

- **401 Unauthorized** — the token is wrong, expired, or revoked. Ask the user to regenerate.
- **Token format** — always starts with `tn_` followed by 32 alphanumerics. Anything else is not a Tenants token.
- **No cookies involved** — do not pass `-b cookies` / `-c cookies` anywhere. All state is in the Bearer header.
