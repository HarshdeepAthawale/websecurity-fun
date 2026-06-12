# VRT Quick Reference — P1 to P5

← [[VRT-Overview]]

Full Bugcrowd VRT v1.18. Use `Ctrl+F` in Obsidian to search. For severity decision logic see [[Severity-Guide]].

---

## P1 — Critical

| Category | Vulnerability | Variant |
|----------|--------------|---------|
| Server-Side Injection | Remote Code Execution (RCE) | |
| Server-Side Injection | SQL Injection | |
| Server-Side Injection | XML External Entity (XXE) | |
| Server-Side Injection | File Inclusion | Local |
| Broken Authentication | Authentication Bypass | |
| Broken Access Control | IDOR | Modify/View Sensitive Info (Iterable IDs) |
| Server Security Misconfig | Exposed Portal | Admin Portal |
| Server Security Misconfig | Using Default Credentials | |
| Sensitive Data Exposure | Disclosure of Secrets | Publicly Accessible Asset |
| Cloud Security | IAM Misconfigurations | Publicly Accessible IAM Credentials |
| AI Application Security | Remote Code Execution | Full System Compromise |
| AI Application Security | Sensitive Information Disclosure | Cross-Tenant PII Leakage |
| AI Application Security | Sensitive Information Disclosure | Key Leak |
| AI Application Security | Training Data Poisoning | Backdoor Injection / Bias Manipulation |
| AI Application Security | Model Extraction | API Query-Based Reconstruction |
| Smart Contract | Reentrancy Attack | |
| Smart Contract | Smart Contract Owner Takeover | |
| Smart Contract | Unauthorized Transfer of Funds | |
| Smart Contract | Uninitialized Variables | |
| Insecure OS/Firmware | Command Injection | |
| Insecure OS/Firmware | Hardcoded Password | Privileged User |
| Zero Knowledge | Deanonymization of Data | |
| Zero Knowledge | Improper Proof Validation and Finalization Logic | |

---

## P2 — High

| Category | Vulnerability | Variant |
|----------|--------------|---------|
| Cross-Site Scripting | Stored XSS | Non-Privileged User to Anyone |
| Cross-Site Request Forgery | CSRF | Application-Wide |
| Server Security Misconfig | SSRF | Internal High Impact |
| Server Security Misconfig | OAuth Misconfiguration | Account Takeover |
| Sensitive Data Exposure | Weak Password Reset | Token Leakage via Host Header Poisoning |
| Broken Access Control | IDOR | Modify Sensitive Info (Iterable IDs) |
| Cloud Security | IAM Misconfigurations | Overly Permissive IAM Roles |
| Cloud Security | Storage Misconfigurations | Unencrypted Sensitive Data at Rest |
| Cryptographic Weakness | Key Reuse | Inter-Environment |
| AI Application Security | Prompt Injection | System Prompt Leakage |
| AI Application Security | Denial-of-Service | Application-Wide |
| AI Application Security | Remote Code Execution | Sandboxed Container Code Execution |
| AI Application Security | Vector/Embedding Weaknesses | Embedding Exfiltration / Model Extraction |
| Smart Contract | Integer Overflow / Underflow | |
| Smart Contract | Unauthorized Smart Contract Approval | |
| Application-Level DoS | Critical Impact / Easy Difficulty | |
| Insecure OS/Firmware | Hardcoded Password | Non-Privileged User |
| Insecure OS/Firmware | Local Administrator on Default Environment | |

---

## P3 — Medium

| Category | Vulnerability | Variant |
|----------|--------------|---------|
| Cross-Site Scripting | Reflected XSS | Non-Self |
| Cross-Site Scripting | Stored XSS | Privileged User to Privilege Elevation |
| Cross-Site Scripting | Stored XSS | CSRF/URL-Based |
| Broken Access Control | IDOR | View Sensitive Info (Iterable IDs) |
| Broken Authentication | 2FA Bypass | |
| Broken Authentication | Session Fixation | Remote Attack Vector |
| Server Security Misconfig | SSRF | Internal Scan / Medium Impact |
| Server Security Misconfig | Misconfigured DNS | Subdomain Takeover |
| Server Security Misconfig | Exposed Portal | Non-Admin Portal |
| Server Security Misconfig | Mail Server Misconfig | No Spoofing Protection on Email Domain |
| Server-Side Injection | HTTP Response Manipulation | CRLF / Response Splitting |
| Server-Side Injection | Content Spoofing | iFrame Injection |
| Sensitive Data Exposure | Disclosure of Secrets | Internal Asset |
| Cryptographic Weakness | Broken Cryptography | Use of Broken Cryptographic Primitive |
| Cryptographic Weakness | Insecure Key Generation | Insufficient Key Space |
| AI Application Security | Improper Output Handling | Cross-Site Scripting (XSS) |
| AI Application Security | Vector/Embedding Weaknesses | Semantic Indexing |
| Smart Contract | Function-Level Denial of Service | |
| Smart Contract | Improper Fee Implementation | |
| Smart Contract | Malicious Superuser Risk | |
| Application-Level DoS | High Impact / Medium Difficulty | |

---

## P4 — Low

| Category | Vulnerability | Variant |
|----------|--------------|---------|
| Cross-Site Scripting | Stored XSS | Privileged User to No Privilege Elevation |
| Cross-Site Scripting | Reflected XSS | Off-Domain / Data URI |
| Cross-Site Scripting | Universal XSS (UXSS) | |
| Broken Access Control | IDOR | Modify/View Sensitive Info (Complex IDs — GUID/UUID) |
| Broken Access Control | Bypass of Password Confirmation | Change Password |
| Broken Access Control | Username/Email Enumeration | Non-Brute Force |
| Broken Authentication | Failure to Invalidate Session | On Logout (Client and Server-Side) |
| Broken Authentication | Failure to Invalidate Session | On Password Reset/Change |
| Broken Authentication | Cleartext Transmission of Session Token | |
| Broken Authentication | Weak Login Function | Over HTTP |
| Server Security Misconfig | Clickjacking | Sensitive Click-Based Action |
| Server Security Misconfig | No Rate Limiting on Form | Login |
| Server Security Misconfig | No Rate Limiting on Form | Registration |
| Server Security Misconfig | No Rate Limiting on Form | Email-Triggering |
| Server Security Misconfig | No Rate Limiting on Form | SMS-Triggering |
| Server Security Misconfig | CAPTCHA | Implementation Vulnerability |
| Server Security Misconfig | OAuth Misconfiguration | Account Squatting |
| Server Security Misconfig | Missing Secure or HttpOnly Cookie Flag | Session Token |
| Server Security Misconfig | Mail Server Misconfig | Email Spoofing to Inbox (Missing/Misconfigured DMARC) |
| Server Security Misconfig | Lack of Security Headers | Cache-Control for Sensitive Page |
| Server Security Misconfig | DBMS Misconfiguration | Excessively Privileged User / DBA |
| Server-Side Injection | SSTI | Basic |
| Server-Side Injection | Content Spoofing | Email HTML Injection |
| Sensitive Data Exposure | Visible Detailed Error/Debug Page | Detailed Server Configuration |
| Sensitive Data Exposure | Sensitive Token in URL | User Facing |
| Sensitive Data Exposure | Token Leakage via Referer | Over HTTP |
| Sensitive Data Exposure | Token Leakage via Referer | Untrusted 3rd Party |
| Sensitive Data Exposure | Via localStorage/sessionStorage | Sensitive Token |
| Sensitive Data Exposure | Weak Password Reset | Password Reset Token Sent Over HTTP |
| Cryptographic Weakness | Insufficient Entropy | Predictable IV / PRNG Seed |
| Cryptographic Weakness | Side-Channel Attack | Padding Oracle Attack |
| Cryptographic Weakness | Side-Channel Attack | Timing Attack |
| Cryptographic Weakness | Use of Expired Cryptographic Key/Certificate | |
| Unvalidated Redirects | Open Redirect | GET-Based |
| AI Application Security | Improper Output Handling | Markdown/HTML Injection |
| AI Application Security | Insufficient Rate Limiting | Query Flooding / API Token Abuse |
| AI Application Security | Denial-of-Service | Tenant-Scoped |
| Cloud Security | Misconfigured Services/APIs | Insecure API Endpoints |

---

## P5 — Informational

| Category | Vulnerability | Variant |
|----------|--------------|---------|
| Cross-Site Scripting | Reflected XSS | Self |
| Cross-Site Scripting | Stored XSS | Self |
| Cross-Site Scripting | Cookie-Based | |
| Cross-Site Scripting | IE-Only | |
| Cross-Site Request Forgery | Action-Specific | Logout |
| Cross-Site Request Forgery | CSRF Token Not Unique Per Request | |
| Broken Authentication | Concurrent Logins | |
| Broken Authentication | Failure to Invalidate Session | Long Timeout |
| Broken Authentication | Failure to Invalidate Session | On Email Change |
| Broken Authentication | Failure to Invalidate Session | On 2FA Activation/Change |
| Server Security Misconfig | Clickjacking | Non-Sensitive Action |
| Server Security Misconfig | Clickjacking | Form Input |
| Server Security Misconfig | Fingerprinting/Banner Disclosure | |
| Server Security Misconfig | Directory Listing Enabled | Non-Sensitive Data Exposure |
| Server Security Misconfig | Lack of Security Headers | CSP |
| Server Security Misconfig | Lack of Security Headers | HSTS |
| Server Security Misconfig | Lack of Security Headers | X-Frame-Options |
| Server Security Misconfig | Lack of Security Headers | X-Content-Type-Options |
| Server Security Misconfig | Lack of Security Headers | X-XSS-Protection |
| Server Security Misconfig | Missing Secure or HttpOnly Cookie Flag | Non-Session Cookie |
| Server Security Misconfig | No Rate Limiting on Form | Change Password |
| Server Security Misconfig | SSRF | External DNS Query Only |
| Server Security Misconfig | SSRF | External Low Impact |
| Server Security Misconfig | Mail Server Misconfig | Missing/Misconfigured SPF/DKIM |
| Server Security Misconfig | Mail Server Misconfig | Email Spoofing to Spam Folder |
| Server Security Misconfig | Insecure SSL | Insecure Cipher Suite |
| Server Security Misconfig | Insecure SSL | Lack of Forward Secrecy |
| Server Security Misconfig | Missing Subresource Integrity | |
| Sensitive Data Exposure | GraphQL Introspection Enabled | |
| Sensitive Data Exposure | Internal IP Disclosure | |
| Sensitive Data Exposure | Full Path Disclosure | |
| Sensitive Data Exposure | Descriptive Stack Trace | |
| Sensitive Data Exposure | Token Leakage via Referer | Trusted 3rd Party |
| Sensitive Data Exposure | Mixed Content (HTTPS sourcing HTTP) | |
| Sensitive Data Exposure | Via localStorage/sessionStorage | Non-Sensitive Token |
| Insufficient Security Config | Weak Password Policy | |
| Insufficient Security Config | Weak Password Reset | Token Not Invalidated After Use |
| Insufficient Security Config | Weak Password Reset | Token Has Long Expiry |
| External Behavior | Browser Feature | Autocomplete Enabled |
| External Behavior | CSV Injection | |
| Using Components with Known Vulns | Outdated Software Version | |
| AI Application Security | Improper Input Handling | ANSI Escape Codes / RTL Overrides |
| AI Application Security | AI Safety | Misinformation / Wrong Factual Data |

---

## Varies — Context-Dependent

These require judgment — severity depends on the specific target, impact, and context.

| Category | Vulnerability |
|----------|--------------|
| Broken Access Control | Privilege Escalation |
| Cross-Site Request Forgery | Action-Specific Authenticated Action |
| Server Security Misconfig | Cache Poisoning |
| Server Security Misconfig | Cache Deception |
| Server Security Misconfig | Path Traversal |
| Server Security Misconfig | Race Condition |
| Server Security Misconfig | HTTP Request Smuggling |
| Server Security Misconfig | Unsafe CORS |
| Server Security Misconfig | OAuth Misconfiguration — Missing/Broken State Parameter |
| Server-Side Injection | SSTI — Custom |
| Server-Side Injection | LDAP Injection |
| Server-Side Injection | Sensitive Data Exposed |
| Insecure Data Transport | Cleartext Transmission of Sensitive Data |
| Cloud Security | Publicly Accessible Cloud Storage |
| Cloud Security | Exposed Debug / Admin Interfaces |
| Cryptographic Weakness | Multiple (see [[Cryptographic-Weaknesses]]) |
| Smart Contracts | Multiple DeFi variants |
| Zero Knowledge | Missing Constraint / Missing Range Check |

---

*Source: Bugcrowd VRT v1.18 — [github.com/bugcrowd/vulnerability-rating-taxonomy](https://github.com/bugcrowd/vulnerability-rating-taxonomy)*
