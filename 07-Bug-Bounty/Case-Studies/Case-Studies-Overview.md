# Real Bug Bounty Case Studies

← [[VRT-Overview]] | [[Reporting-Guide]]

---

These are my real reports from real bug bounty programs — not CTF challenges, not synthetic labs. They've been sanitized (credentials and PII redacted) but the vulnerabilities, methodology, and reasoning are exactly as they happened.

Reading a real report is different from reading a textbook. The textbook says "CORS can be misconfigured." A real report shows you: which curl command, which response header, why it matters in this specific app's architecture, and how a triager will evaluate it.

> [!TIP]
> Start with CS-05 (Information Disclosure) and CS-06 (CORS) — they're lower complexity and show you what "actually submittable" looks like. Then move to CS-01 (GraphQL) and CS-04 (Brute Force). The supply chain and JWT cases are advanced.

---

## Case Studies Index

| # | Case Study | Vuln Class | Severity | Primary Lesson |
|---|-----------|------------|----------|----------------|
| [[CS-01-GraphQL-Unauthenticated]] | Unauthenticated GraphQL / Directus CMS | Broken Access Control | P2 / HIGH | Same root cause → 2 findings. Fingerprinting → default configs. |
| [[CS-02-Supply-Chain-Dependency-Confusion]] | Artifactory Anonymous Access → Supply Chain RCE | Broken Access Control + Supply Chain | P1 / CRITICAL | Two-step attack chain. Information disclosure that chains into RCE. |
| [[CS-03-JWT-Auth-Bypass-And-NA-Lesson]] | JWT Signature Bypass on Management API | Broken Auth | HIGH (closed N/A) | **Why reports get closed.** Differential testing. Partial bypass ≠ valid report. |
| [[CS-04-Brute-Force-And-Hardcoded-Creds]] | PIN Brute Force + Hardcoded Credentials in JS | Broken Auth | P2 / HIGH | Two CWEs in one report. JS bundle mining. Rate limit testing. |
| [[CS-05-Information-Disclosure-Chain]] | ASP.NET Verbose Errors → Recon Escalation | Information Disclosure | P3 / MEDIUM | Info disclosure isn't just "low severity." Stack trace → targeted attack. |
| [[CS-06-CORS-Misconfiguration]] | CORS Origin Reflection + Null Origin | CORS Misconfiguration | P4 / LOW | How to properly scope CORS impact. SameSite vs CORS nuance. |
| [[CS-07-Real-Hunt-Recon-Methodology]] | Full Recon Session on a 5900-domain VDP | Methodology | — | What a real hunt actually looks like. How to prioritize. How to track findings. |

---

## What These Reports Have in Common

Most of these findings came from the same hunt — a single VDP (Vulnerability Disclosure Program) with ~5900 domains in scope. That's an important lesson by itself:

**Wide scope → more surface → more findings.**

A narrow "just this one app" scope rarely yields 6 findings. Wide scope programs like large company VDPs are where you rack up findings fast because:
- More subdomains = more tech stacks = more configuration mistakes
- Patterns repeat: find one Directus CMS misconfigured, check all the Directus instances
- One finding gives you context for the next: Artifactory exposed → now you know the internal package namespaces → check for dependency confusion

This is called **lateral recon** — using what you found to look for more.

---

## Skill Progression

```
Level 1 → CS-05, CS-06       ← info disclosure, CORS; simple curl

Level 2 → CS-01, CS-04       ← GraphQL recon; JS bundle analysis; brute force math
Level 3 → CS-03              ← JWT internals; differential testing; understanding N/As
Level 4 → CS-02              ← supply chain attack chains; dependency confusion
Level 5 → CS-07              ← full session methodology; target triage; dead-end tracking
```

---

## On Redaction

These case studies are based on real submitted reports. The following have been sanitized:
- **API keys/credentials** → replaced with `REDACTED_KEY_X` placeholders
- **Employee PII (names, emails)** → replaced with `[REDACTED]`
- **Internal IPs/tokens** → generalized

The commands, CVSS reasoning, vulnerability mechanics, and triager responses are real.

---

*Related: [[VRT-Quick-Reference]] | [[Severity-Guide]] | [[Reporting-Guide]]*

*Sources: Real bug bounty submissions (Red Bull Intigriti VDP, HackerOne OPPO program)*
