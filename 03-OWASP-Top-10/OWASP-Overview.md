# OWASP Top 10 Overview

**Overview** | [[A01-Broken-Access-Control]] | [[A03-Injection]] | [[A07-Auth-Failures]]

---

## What Is OWASP?

OWASP (Open Worldwide Application Security Project) is a nonprofit that produces free, open security resources. The most famous one is the **OWASP Top 10** — a list of the 10 most critical web application security risks, updated every few years based on real-world data from thousands of apps.

It's not a "top 10 vulnerabilities to find" — it's a "top 10 risk categories" that every developer and security person should understand.

---

## The 2021 Top 10

| # | Category | What It Covers |
|---|----------|---------------|
| A01 | [[A01-Broken-Access-Control]] | IDOR, privilege escalation, path traversal, misconfig |
| A02 | [[A02-Cryptographic-Failures]] | Weak encryption, plain-text data, TLS issues |
| A03 | [[A03-Injection]] | SQLi, XSS, command injection, SSTI, XXE |
| A04 | [[A04-Insecure-Design]] | Missing threat modeling, flawed business logic |
| A05 | [[A05-Security-Misconfiguration]] | Default creds, verbose errors, unnecessary features |
| A06 | [[A06-Vulnerable-Components]] | Outdated libraries, unpatched dependencies |
| A07 | [[A07-Auth-Failures]] | Broken auth, weak passwords, session issues |
| A08 | [[A08-Software-Integrity-Failures]] | Supply chain attacks, unsigned updates |
| A09 | [[A09-Logging-Failures]] | Missing logs, not alerting on attacks |
| A10 | [[A10-SSRF]] | Server-side request forgery |

---

## How to Use This Section

Each OWASP entry here is an **overview** — it explains the risk category with real-world context. For deep technical details and exploitation techniques, each note links to the relevant **04-Vuln-Classes/** notes.

**Think of it this way:**
- OWASP Top 10 → "what risk category is this and why does it matter?"
- 04-Vuln-Classes → "how do I find, exploit, and test for this?"

---

## Why This List Matters

The OWASP Top 10 shows up everywhere:
- **Bug bounty** — most programs reward bugs that fall into these categories
- **Pentesting** — client reports are typically structured around OWASP
- **Dev conversations** — when you find a bug and explain it to a developer, referencing OWASP helps
- **Certifications and interviews** — you will be asked about this list

---

## Start Here

The three highest-impact categories to learn first:

1. **[[A01-Broken-Access-Control]]** — #1 for a reason. More bugs fall here than anywhere else. IDOR alone is one of the most common and well-paid bug bounty finding classes.

2. **[[A03-Injection]]** — SQLi, XSS, command injection. Classic vulns that still appear everywhere, especially in older codebases and edge-case inputs that devs didn't think to sanitize.

3. **[[A07-Auth-Failures]]** — Broken authentication. JWT attacks, session mismanagement, password reset flaws. High severity when found.

---

*Sources: OWASP Top 10 2021, OWASP Testing Guide v4*
