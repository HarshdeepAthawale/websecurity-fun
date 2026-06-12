# Bug Bounty Reporting Guide

← [[VRT-Overview]] | [[Severity-Guide]]

---

A great finding with a bad report gets downgraded, delayed, or dismissed. A mediocre finding with an excellent report gets triaged fast and paid correctly. The report is part of the job.

---

## The 7 Things Every Report Must Have

1. **Title** — one sentence, vuln type + location + impact
2. **Severity** — your P-rating with justification
3. **Summary** — 2–3 sentences, what the bug is and what an attacker can do
4. **Steps to Reproduce** — numbered, exact, reproducible
5. **Proof of Concept** — screenshots, video, or code that proves it works
6. **Impact** — what a real attacker actually gains
7. **Remediation** — how to fix it

---

## Title Formula

```
[Vuln Type] in [Feature/Endpoint] allows [Impact]
```

Good titles:
```
Stored XSS in user profile bio allows account takeover via cookie theft
IDOR in /api/invoices/{id} allows any authenticated user to view other users' invoices
Host header injection in password reset allows attacker to capture reset tokens
SSRF via webhook URL parameter reaches AWS metadata endpoint and leaks IAM credentials
```

Bad titles:
```
XSS found                          ← too vague
Security issue on your website     ← useless
Vulnerability in login             ← no impact stated
```

---

## Summary — Two Sentences Max

Sentence 1: What the vulnerability is and where it exists.
Sentence 2: What an attacker can do with it.

```
The /api/user/{id}/profile endpoint fails to verify that the requesting 
user owns the resource identified by the {id} parameter. An authenticated 
attacker can enumerate user IDs to access any user's full profile data, 
including email address, phone number, and home address.
```

Don't write a novel. Triagers read dozens of reports a day. They want the answer in 30 seconds.

---

## Steps to Reproduce

Be exact. Another person with no context should be able to follow these and see the same result.

**Template:**
```
1. Log in as User A (attacker) at https://example.com/login
2. Navigate to https://example.com/profile — note User A's user ID in the URL: 1234
3. Open Burp Suite and intercept the following request:
   GET /api/user/1234/profile HTTP/1.1
   Host: example.com
   Cookie: session=USER_A_SESSION
4. Change the user ID from 1234 to 1235 (another user's ID)
5. Forward the modified request
6. Observe: The response contains User B's full profile data including email, phone, and address
```

Rules:
- Numbered, sequential steps
- Include exact URLs, parameters, headers
- Include test account credentials if you have them (or note what permissions are needed)
- State what you expect to happen vs what actually happens

---

## Proof of Concept

**Minimum:** A screenshot showing the vulnerability and the response.

**Better:** A screen recording of the full exploit chain.

**Best (for injection):** A working PoC script.

For XSS — don't just show `alert(1)`. Show `alert(document.cookie)` or `alert(document.domain)`. It immediately shows the impact is real, not just theoretical.

For IDOR — include a screenshot of the request (with the modified ID) and the response (showing the other user's data). Redact actual PII — replace real data with `[REDACTED]` in the screenshot.

For SSRF — show the request and show your out-of-band server receiving the callback, OR show the response body if it's in-band.

---

## Impact Statement

This is where most reports are weak. Don't just say "an attacker could steal data" — be specific about what data, how many users, what they could do with it.

**Weak:**
```
An attacker could view other users' information.
```

**Strong:**
```
An unauthenticated attacker can enumerate all user IDs (sequential integers starting 
at 1) and retrieve the full profile of each user including: full name, email address, 
phone number, home address, and last 4 digits of payment card. Given the application 
has ~500,000 registered users (per the /api/stats endpoint), the entire user database 
is exposed. This data is sufficient for identity theft, spear phishing, and payment 
fraud.
```

Key components:
- Who can exploit it (unauthenticated? any user? only privileged users?)
- What data or action is accessible
- Scope (one user? all users? internal systems?)
- Real-world harm (identity theft, financial loss, account takeover, service disruption)

---

## Remediation

One or two sentences on how to fix it. You don't need to write the code — just point in the right direction.

```
Implement server-side ownership verification on all resource-fetching endpoints. 
The server should validate that the authenticated user's ID matches the ID parameter 
in the request before returning any data.
```

For XSS:
```
Encode all user-supplied data before rendering it in HTML. 
Implement a Content-Security-Policy header to prevent execution of injected scripts.
```

---

## Platform-Specific Tips

### Bugcrowd

- Use their severity guidelines (VRT) — they'll respect that you know the framework
- Include CVSS 3.1 score if you want to argue for a specific rating
- Duplicate claims: if you're not sure it's a dup, submit anyway with a note

### HackerOne

- Structure report with their built-in fields (Weakness, CVSS, etc.)
- They have a "Remediation Guidance" field — use it
- If you get triaged as duplicate, ask which report number it duplicated — learn from it

### Intigriti

- More European programs — be thorough on GDPR impact if it's a data exposure
- They appreciate well-structured Markdown reports

---

## What Gets Reports Closed as N/A

Know these. Submitting them tanks your signal-to-noise ratio.

| Finding | Why it's N/A |
|---------|-------------|
| Self-XSS (victim must run payload themselves) | Not exploitable without victim's own action |
| Missing security headers alone | Not a vulnerability, only a defense |
| Theoretical CSRF without a working PoC | Must demonstrate it's actually exploitable |
| Rate limiting on low-risk actions (view profile, public search) | No meaningful security impact |
| Banner disclosure / version numbers | Only useful for chaining — submit with a chain |
| Open redirect without a clear exploitation scenario | Too speculative without demonstrated chain |
| "Could be used for phishing" as the only impact | Not a security vulnerability in the application |
| Bugs on out-of-scope assets | Read the scope first |
| Bugs requiring physical access or privileged access | Usually out of scope or low severity |
| Bugs in third-party components outside the target's control | Not their responsibility |
| Best practices violations with no realistic attack path | Informational at best |

---

## Report Checklist (Before Submitting)

- [ ] Title follows the formula: `[Vuln Type] in [Location] allows [Impact]`
- [ ] Severity matches the VRT baseline — justified if arguing higher/lower
- [ ] Steps to reproduce are exact, numbered, and reproducible
- [ ] PoC included (screenshot or code)
- [ ] Impact statement is specific — not just "attacker could..."
- [ ] Checked program scope — is the asset in scope?
- [ ] Checked the "known issues" list on the program page
- [ ] Impact is realistic, not chained from 5 theoretical steps
- [ ] No real user PII included in screenshots (redact it)

---

*Related: [[Severity-Guide]] | [[VRT-Quick-Reference]] | [[VRT-Overview]]*

*Sources: Bugcrowd Researcher Resources, HackerOne Disclosure Guidelines*
