# A09 — Security Logging and Monitoring Failures

← [[OWASP-Overview]]

---

This category isn't a vulnerability you exploit — it's a defense failure that lets attackers operate undetected. It's included in the OWASP Top 10 because without logging and monitoring, **breaches go undetected for months**.

The average time to detect a breach is 207 days (IBM Cost of a Data Breach Report). Good logging shortens that.

---

## Why It Matters for Offensive Security

As a security tester or bug hunter, you care about logging failures in two ways:

1. **You're the attacker:** Missing logging means your attacks aren't detected, making your testing easier — but also highlighting a real risk
2. **You're writing the report:** "The application has no rate limiting and failed login attempts are not logged" = increased impact statement for your brute force/IDOR findings

---

## What Should Be Logged

| Event | Why |
|-------|-----|
| All authentication attempts (success and failure) | Detect brute force, credential stuffing |
| All access control failures (401, 403) | Detect IDOR probing, privilege escalation attempts |
| All input validation failures | Detect injection attempts |
| High-value transactions | Detect fraud, logic abuse |
| Admin actions | Audit trail for privileged operations |
| Session creation and destruction | Detect session hijacking patterns |

---

## Common Logging Failures

### No Logging at All

No records of failed logins, errors, or suspicious activity. An attacker can brute force thousands of passwords with no trace.

### Logs Not Monitored / Alerted On

Logs exist but no one reads them. There's no alert when 1000 failed logins happen in 10 minutes.

### Insufficient Log Detail

```
# Unhelpful log entry:
[2023-01-15 10:23:45] ERROR: Authentication failed

# Useful log entry:
[2023-01-15 10:23:45] AUTH_FAILURE ip=192.168.1.1 user=admin endpoint=/login 
  user_agent="Mozilla/5.0" attempt_count=47
```

Without IP, user, timestamp, and context, the log is useless for investigation.

### Logs Stored Insecurely

- Logs stored in the same database as the app — SQLi can tamper with them
- Logs stored in a world-readable file
- Logs deleted when an attacker gains access

### Sensitive Data in Logs

The opposite problem — logs contain too much sensitive data:
- Passwords logged in plaintext (e.g., full request body logged)
- Session tokens in log files
- PII (credit card numbers, SSNs) in logs

---

## Testing for Logging Failures

You typically can't directly verify logging unless you have server access, but you can infer it:

1. **Perform obvious attacks with no response changes:** Try 50 failed logins. Does the app slow down, add a CAPTCHA, or lock the account? If not, it's likely not monitoring.

2. **Check if you get locked out or rate-limited:** No lockout = probably no monitoring.

3. **Look for logging endpoints exposed:** Some apps expose their log viewer:
   ```
   /logs/
   /admin/logs
   /debug/logs
   ```

4. **Check error responses:** Do 400/500 errors expose log lines or stack traces? (Also a [[A05-Security-Misconfiguration]] finding.)

---

## Impact for Bug Reports

Logging failures amplify the impact of other vulnerabilities. When writing reports:

- **IDOR with no logging:** "An attacker could enumerate all user records and the application has no logging or alerting to detect such enumeration, meaning the attack could go undetected indefinitely."
- **Brute force with no rate limiting AND no logging:** Two findings, not one. The rate limit issue is the vuln; the logging failure means there's no compensating control.

---

## Compliance Context

Many compliance frameworks require logging:
- **PCI-DSS** — requires logging all access to cardholder data
- **HIPAA** — requires audit logs for PHI access
- **SOC 2** — logging is a core trust principle
- **GDPR** — processing activities must be documented

If you're testing a healthcare or finance app and find missing logs, the compliance angle significantly increases severity.

---

*Related: [[A07-Auth-Failures]] | [[A01-Broken-Access-Control]] | [[OWASP-Overview]]*

*Sources: OWASP Top 10 2021, IBM Cost of a Data Breach 2023*
