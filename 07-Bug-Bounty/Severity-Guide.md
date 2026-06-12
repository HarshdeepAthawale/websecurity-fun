# Severity Guide — Picking the Right P-Rating

← [[VRT-Overview]] | [[VRT-Quick-Reference]] | [[Reporting-Guide]]

---

The most common mistake in bug bounty is misrating your finding. Oversell a P5 as a P2 and you burn your validity ratio. Undersell a P2 as a P4 and you leave money on the table. This guide walks through how to think about severity correctly.

---

## The Core Question: What's the Real Impact?

Before looking at the VRT table, answer this:

> **If a real attacker exploited this right now, what's the worst realistic outcome?**

Not theoretical worst case. Not "if combined with 5 other bugs." The realistic, direct impact from this one finding.

| Realistic impact | Direction |
|---|---|
| Full account takeover, RCE, data breach of all users | P1 |
| Specific user's account compromised, significant data exposure | P2 |
| Significant degradation of security, moderate data exposure | P3 |
| Minor security improvement needed, limited real-world risk | P4 |
| Best practice violation, negligible real impact | P5 |

---

## P1 Checklist

You're at P1 when exploitation results in **immediate, significant business harm**:

- [ ] Arbitrary code execution on the server
- [ ] Authentication bypass — access any account without credentials
- [ ] SQL injection that dumps the full user database
- [ ] SSRF that reaches cloud metadata and returns IAM credentials
- [ ] Admin portal accessible with default credentials
- [ ] LFI that reads `/etc/shadow` or application secrets
- [ ] Publicly readable S3 bucket with sensitive customer PII

> [!WARNING]
> Don't claim P1 without a working PoC. Triagers will test it. A theoretical P1 without evidence of exploitability gets downgraded.

---

## P2 Checklist

Significant vulnerability, requires some user interaction or specific conditions:

- [ ] Stored XSS that fires for any user who visits the page (non-priv attacker)
- [ ] SSRF reaching internal services (Redis, Elasticsearch, internal APIs)
- [ ] IDOR allowing modification of another user's data (not just viewing)
- [ ] OAuth misconfiguration allowing account takeover
- [ ] Host header injection in password reset → attacker captures reset token
- [ ] CSRF on application-wide state-changing actions
- [ ] Exposed admin panel (non-default credentials) → significant access

---

## P3 Checklist

Real vulnerability but limited blast radius or requires user interaction:

- [ ] Reflected XSS (victim must click a crafted link)
- [ ] Stored XSS only triggerable by a privileged user (e.g., admin posting to admin panel)
- [ ] IDOR that only allows viewing (not modifying) sensitive data
- [ ] Subdomain takeover
- [ ] 2FA bypass
- [ ] SSRF that only does internal port scanning (no data returned)
- [ ] Session fixation (requires social engineering)
- [ ] Missing email spoofing protection on the primary email domain

---

## P4 Checklist

Vulnerability exists but exploitation is limited, unlikely, or the impact is low:

- [ ] Open redirect (GET-based) — useful as a chain component but low standalone
- [ ] Missing HttpOnly/Secure on session cookie
- [ ] No rate limiting on login or registration form
- [ ] Sensitive data in URL (token, session ID)
- [ ] Token leakage via Referer to untrusted third party
- [ ] Self-XSS that requires the victim to run the payload themselves
- [ ] CAPTCHA implementation weakness
- [ ] Basic SSTI (confirmed but no escalation to RCE)
- [ ] Cleartext session token transmission
- [ ] Username/email enumeration via non-brute force method

---

## P5 Checklist

Informational — document it but don't expect a payout:

- [ ] Missing security headers (CSP, HSTS, X-Frame-Options) alone — no exploit chain
- [ ] Software version/banner disclosure
- [ ] Directory listing on non-sensitive paths
- [ ] GraphQL introspection enabled (informational — note what the schema reveals)
- [ ] Internal IP disclosure
- [ ] Stack trace / error page (record what it reveals — could upgrade a chain)
- [ ] Self-XSS (victim must paste payload into their own console)
- [ ] Weak password policy (report it but don't expect bounty)
- [ ] Autocomplete enabled on sensitive form fields
- [ ] Outdated software version (without confirmed exploitability)

---

## Common Severity Mistakes

### Upgrading P5s

**"Missing CSP is P3 because it enables XSS"** — No. Missing CSP alone is P5. CSP is a *defense*, not a vulnerability. You need the actual XSS to have a finding. Missing CSP *could* be mentioned in the XSS report as an aggravating factor.

**"Open redirect is P2 because it could be used for phishing"** — Standalone open redirect is P4 (GET-based) per VRT. It becomes P2 if you demonstrate a complete chain: open redirect → OAuth token theft → account takeover.

**"Self-XSS is P3 because I could use it in a CSRF chain"** — If you haven't demonstrated the full chain, it's P5. Prove the chain or report the CSRF separately.

### Downgrading P2s

**"Stored XSS requires login so it's P3"** — If the stored XSS fires for any authenticated user who views the page and a non-privileged attacker can inject it, it's P2. Authentication requirement doesn't automatically drop severity.

**"SSRF only hits internal services, not the internet"** — Internal SSRF is *more* dangerous. Internal SSRF to Redis/Elasticsearch/metadata endpoint is P2. External-only SSRF is P4/P5.

---

## The "Varies" Category

Some VRT entries are marked "Varies." These require judgment based on:

| Factor | Lower severity | Higher severity |
|--------|---|---|
| Data sensitivity | Public/non-PII data | PII, financial, credentials |
| User interaction | Requires complex interaction | Zero-click |
| Authentication required | Authenticated only | Unauthenticated |
| Impact scope | Single user | All users / entire platform |
| Exploitability | Hard to reproduce reliably | Trivial, automated |

For "Varies" findings, write a clear impact statement and let the evidence guide the severity. Don't guess P1 — argue from evidence.

---

## Chaining for Severity Upgrades

Low-severity bugs can combine into high-severity chains. Document the full chain in your report:

| Chain | Result |
|-------|--------|
| Open redirect + OAuth → steal auth code | P4 + P5 → P2 |
| Self-XSS + CSRF → XSS on victim | P5 + P4 → P3 |
| Info disclosure (stack trace) + SQLi → RCE | Chain → P1 |
| SSRF + internal metadata → IAM credentials | P2 → P1 |
| IDOR (view) + sensitive PII + no logging | P3 → P2 with aggravation |

Always report the **highest severity the full chain achieves**, not the individual components.

---

*Related: [[VRT-Quick-Reference]] | [[Reporting-Guide]] | [[VRT-Overview]]*

*Sources: Bugcrowd VRT v1.18, Bugcrowd Researcher Resources*
