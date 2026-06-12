# Bugcrowd VRT — Overview

**Overview** | [[VRT-Quick-Reference]] | [[Severity-Guide]] | [[Reporting-Guide]]

---

## What Is the VRT?

The **Vulnerability Rating Taxonomy (VRT)** is Bugcrowd's public reference for how vulnerabilities are categorized and prioritized. It's the closest thing the bug bounty industry has to a standardized severity scale.

When you submit a bug on Bugcrowd, the triage team references the VRT to assign (or adjust) your severity. Understanding it means:
- You submit with the right priority the first time
- You write impact statements that match what Bugcrowd expects
- You don't waste time submitting P5s thinking they're P2s
- You don't undersell a P2 by calling it P3

Current version: **VRT v1.18** (March 2026) — [GitHub](https://github.com/bugcrowd/vulnerability-rating-taxonomy)

---

## The Priority Scale

| Priority | Name | What It Means |
|----------|------|---------------|
| **P1** | Critical | Direct, immediate business impact. RCE, auth bypass, critical data breach. |
| **P2** | High | Significant impact requiring exploitable conditions. Stored XSS on privileged targets, SSRF to internal systems. |
| **P3** | Medium | Real vulnerability, limited impact or requiring user interaction. Reflected XSS, subdomain takeover, 2FA bypass. |
| **P4** | Low | Vulnerability exists but impact is minimal or requires significant preconditions. Missing security headers, most info disclosure. |
| **P5** | Informational | Behavior is technically a weakness but the real-world impact is negligible. Self-XSS, missing CAPTCHA on low-risk forms. |

> [!NOTE]
> Programs can override VRT ratings. A company that handles healthcare data might rate token leakage higher than the baseline VRT suggests. Always read the program brief — the VRT is a **baseline**, not a hard rule.

---

## How the VRT Is Structured

Three levels:

```
Category → Specific Vulnerability → Variant

Example:
Cross-Site Scripting (XSS) → Stored → Non-Privileged User to Anyone → P2
```

The variant level is where most severity decisions happen. A stored XSS where the attacker is a non-privileged user and it fires for anyone is P2. The same stored XSS where it only fires for the user who posted it (self-XSS) is P5. Same vuln class, very different severity.

---

## VRT vs OWASP

| | OWASP Top 10 | Bugcrowd VRT |
|--|---|---|
| Purpose | Developer awareness of risk categories | Severity rating for submitted findings |
| Structure | 10 broad categories | 300+ specific variants with P-ratings |
| New categories | Lags behind | Includes AI, Smart Contracts, Automotive, Zero Knowledge |
| Severity ratings | None | P1–P5 for every variant |
| Use case | Secure development | Bug bounty submission triage |

They complement each other. OWASP tells you *what to look for*. VRT tells you *how to rate what you found*.

---

## What's New in the VRT (vs OWASP)

The VRT covers categories that OWASP barely touches:

- **[[AI-Security]]** — prompt injection, model extraction, LLM-specific attacks (P1–P4)
- **[[Cloud-Security]]** — IAM misconfigs, storage misconfigs, publicly accessible credentials (P1–P4)
- **[[Cryptographic-Weaknesses]]** — padding oracle, timing attacks, key reuse, broken crypto (P2–P5)
- **[[Race-Conditions]]** — covered under Server Security Misconfiguration with `Varies` rating
- **[[Smart-Contracts]]** — reentrancy, unauthorized fund transfer, integer overflow (P1–P3)
- Automotive Security — CAN injection, RF hub attacks (P1–P4)
- Zero Knowledge — missing constraints, proof validation failures (P1, Varies)

---

## Using the VRT When You Find a Bug

1. **Identify the category** — what type of vuln is it?
2. **Find the specific variant** in [[VRT-Quick-Reference]]
3. **Check the baseline P-rating**
4. **Read [[Severity-Guide]]** — does your specific scenario match the variant, or is it higher/lower?
5. **Write the report** using [[Reporting-Guide]] with the correct severity

---

*Sources: Bugcrowd VRT v1.18, Bugcrowd Researcher Resources*
