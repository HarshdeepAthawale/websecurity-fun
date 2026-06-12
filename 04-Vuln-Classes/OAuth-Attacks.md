# OAuth 2.0 Attacks

← [[A07-Auth-Failures]]

---

OAuth 2.0 is the standard for authorization delegation — "Login with Google/GitHub/Facebook". It's widely used and widely misimplemented. Bugs here are often critical (account takeover).

---

## OAuth Flow (Authorization Code Grant)

The most common and most secure grant type:

```
1. User clicks "Login with Google"
2. App redirects: 
   https://accounts.google.com/oauth/authorize
   ?client_id=APP_ID
   &redirect_uri=https://app.example.com/callback
   &response_type=code
   &scope=email profile
   &state=RANDOM_STATE

3. User logs in to Google and approves
4. Google redirects back:
   https://app.example.com/callback?code=AUTH_CODE&state=RANDOM_STATE

5. App's back-end exchanges code for token:
   POST https://accounts.google.com/oauth/token
   {code: AUTH_CODE, client_id: ..., client_secret: ..., redirect_uri: ...}

6. Google returns: access_token, refresh_token, id_token
7. App uses access_token to fetch user profile → creates/logs in user
```

---

## Attack 1 — Open Redirect in redirect_uri

The `redirect_uri` tells the OAuth provider where to send the auth code. If the app validates it loosely, you can redirect the code to your server.

```
# App's legit redirect_uri:
https://app.example.com/oauth/callback

# Bypass attempts:
https://app.example.com@attacker.com/callback       ← @-notation
https://app.example.com.attacker.com/callback       ← subdomain
https://app.example.com/oauth/../redirect?url=https://attacker.com
https://attacker.com?app.example.com                ← query string
https://app.example.com/callback#https://attacker.com
```

If any of these work, you can steal the auth code:

```
1. Craft link: /oauth/start?redirect_uri=https://attacker.com
2. Victim clicks → auth code sent to attacker.com
3. Attacker exchanges code for token → account takeover
```

---

## Attack 2 — Missing State Parameter (CSRF)

The `state` parameter is a CSRF token for OAuth. If it's missing or not validated:

```
1. Attacker starts OAuth flow, gets to provider's auth page
2. Does NOT complete the flow — stops before the callback
3. Notes the URL that would be sent back (with attacker's code)
4. Sends victim a link to: https://app.example.com/callback?code=ATTACKERS_CODE
5. App logs the victim in using the attacker's OAuth account
6. Attacker now controls the victim's app account
```

**Test:** Start OAuth flow, check if `state` parameter is present. If not — CSRF is possible.

---

## Attack 3 — Account Linking Takeover

Many apps let you link OAuth providers to existing accounts. If this isn't properly validated:

```
1. Victim has account: victim@example.com
2. Attacker creates attacker@example.com
3. Attacker connects their Google account (google_id: 12345)
4. Attacker changes their email to victim@example.com (if allowed)
   OR uses IDOR to link Google ID 12345 to victim's account
5. Now attacker can log in as victim via Google
```

---

## Attack 4 — Leaking Tokens via Referer

If the access token or auth code appears in the URL and the page loads third-party resources, the token leaks in the `Referer` header:

```
# Code in URL after redirect
https://app.example.com/callback?code=AUTH_CODE

# Page loads Google Analytics
→ GET https://google-analytics.com/collect?...
  Referer: https://app.example.com/callback?code=AUTH_CODE
```

The third party receives the auth code.

---

## Attack 5 — Implicit Grant Misuse

The implicit grant type returns the access token directly in the URL fragment (`#access_token=...`) — no code exchange step. It's deprecated but still used in SPAs.

```
# Token in fragment
https://app.example.com/#access_token=TOKEN&token_type=bearer

# If the app passes this to the server in a way that leaks it:
POST /api/complete-login
{"token": "[value from fragment]"}

# Intercept this request in Burp — you now have the access token
```

---

## Attack 6 — JWT (id_token) Attacks

If the app receives an `id_token` (JWT) from the OAuth provider and validates it poorly, standard JWT attacks apply. See [[JWT-Attacks]].

---

## Testing OAuth

```
1. Map the full OAuth flow in Burp
2. Check: is state parameter present and validated?
3. Test redirect_uri validation (see bypass list above)
4. Check: does the auth code work more than once?
5. Check: is the token bound to the state? Can you swap codes between users?
6. Look for token leakage in Referer, logs, error messages
7. Check account linking flows for IDOR/CSRF
```

---

## Common Parameters to Manipulate

```
client_id         → try other client IDs (maybe a more privileged app)
redirect_uri      → bypass validation (see above)
scope             → request more permissions than intended
state             → remove entirely, or set to attacker value
response_type     → change from "code" to "token" (implicit)
```

---

*Related: [[JWT-Attacks]] | [[CSRF]] | [[A07-Auth-Failures]]*

*Sources: PortSwigger Web Security Academy, OAuth 2.0 RFC 6749*
