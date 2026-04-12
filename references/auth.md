# OIDC Authentication Flow

The platform uses Keycloak OIDC with cookie-based sessions and `form_post` response mode. All steps use `curl` and store state in `/tmp/tenants-deploy/`.

## Step-by-step

Run each as a **separate** Bash call. Every command starts with `curl`.

### 1. Get Keycloak login page

```bash
mkdir -p /tmp/tenants-deploy
```

```bash
curl -s -c /tmp/tenants-deploy/cookies -b /tmp/tenants-deploy/cookies -L \
  "https://BASE_URL/auth/login" -o /tmp/tenants-deploy/login-form.html
```

### 2. Check if already authenticated or need to POST credentials

Step 1 may return either a **Keycloak login form** (fresh session) or an **OIDC form_post callback** (existing Keycloak session). Check which one:

- If `login-form.html` contains `NAME="code"` → already authenticated at Keycloak, copy to callback and skip to step 3.
- Otherwise → it's a login form, POST credentials.

```bash
if grep -q 'NAME="code"' /tmp/tenants-deploy/login-form.html 2>/dev/null; then
  cp /tmp/tenants-deploy/login-form.html /tmp/tenants-deploy/callback.html
else
  curl -s -c /tmp/tenants-deploy/cookies -b /tmp/tenants-deploy/cookies \
    -d "username=USERNAME&password=PASSWORD" \
    "$(grep -oiP 'action="[^"]*"' /tmp/tenants-deploy/login-form.html | head -1 | cut -d'"' -f2 | sed 's/&amp;/\&/g')" \
    -o /tmp/tenants-deploy/callback.html
fi
```

### 3. Complete OIDC callback

Parse the form_post HTML and POST the OIDC parameters back to the app. Note: the HTML may use uppercase (`NAME`, `VALUE`, `ACTION`) or lowercase attributes.

```bash
curl -s -c /tmp/tenants-deploy/cookies -b /tmp/tenants-deploy/cookies -L -o /dev/null \
  -d "$(grep -oiP 'NAME="\w+" VALUE="[^"]*"' /tmp/tenants-deploy/callback.html \
    | sed 's/NAME="//I;s/" VALUE="/=/I;s/"$//' \
    | tr '\n' '&' | sed 's/&$//')" \
  "https://BASE_URL/signin-oidc"
```

### 4. Verify

```bash
curl -s -b /tmp/tenants-deploy/cookies "https://BASE_URL/auth/me"
```

Should return `{"authenticated":true, ...}`. If not, credentials are wrong.

## Troubleshooting

- If step 1 already returns the form_post callback (contains `NAME="code"`), Keycloak has an existing session — step 2 handles this automatically.
- If step 2 returns a Keycloak error page instead of the callback HTML, the username or password is incorrect.
- The `callback.html` should contain `NAME="code"` — if it doesn't, authentication failed at Keycloak.
- Cookies are stored in `/tmp/tenants-deploy/cookies` and reused for all subsequent API calls.
