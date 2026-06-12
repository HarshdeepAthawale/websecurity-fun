# IDOR — Insecure Direct Object Reference

← [[A01-Broken-Access-Control]]

---

IDOR is one of the most commonly found and paid bugs in bug bounty. The concept is simple: change an ID in a request and access someone else's data.

---

## The Core Idea

The app exposes a direct reference to an internal object (a user ID, order number, file name) and doesn't verify that the requesting user is authorized to access that object.

```
GET /api/invoice/1234     ← your invoice
GET /api/invoice/1235     ← someone else's invoice — does the app let you?
```

---

## Where to Find IDORs

Look for any parameter that references a resource:

| Parameter type | Examples |
|---------------|----------|
| Numeric IDs | `user_id=123`, `order=456`, `invoice_id=789` |
| GUIDs/UUIDs | `id=550e8400-e29b-41d4-a716-446655440000` |
| Hashed IDs | `ref=5f4dcc3b5aa765d61d8327deb882cf99` |
| Encoded values | `data=dXNlcjoxMjM=` (base64: "user:123") |
| Filenames | `file=report_user123.pdf`, `?download=invoice_2023_456.pdf` |
| Email addresses | `email=victim@example.com` as an identifier |

---

## Types of IDOR

### Read IDOR

Access another user's data:
```
GET /api/user/1234/profile    → returns your data
GET /api/user/1235/profile    → returns another user's data
```

### Write IDOR

Modify another user's data:
```
PUT /api/user/1234/email      → updates your email
PUT /api/user/1235/email      → modifies another user's email
```

### Delete IDOR

Delete resources belonging to others:
```
DELETE /api/post/456          → deletes someone else's post
```

### Function-Level IDOR

Access functionality you're not supposed to have:
```
POST /api/admin/users/ban     → accessible with a regular user session
```

---

## Testing Methodology

### Step 1 — Create Two Accounts

Always test with two accounts: **Victim (A)** and **Attacker (B)**.

### Step 2 — Log Requests as Account A

Use Burp Suite. Browse the app as Account A, note all requests that contain IDs.

### Step 3 — Replay with Account B's Session

In Burp Repeater, swap the session cookie/token to Account B's. Change the ID to Account A's resource.

```bash
# Account A's session: session=aaa111
# Account B's session: session=bbb222

# Original request (as A, accessing A's data):
GET /api/profile/101
Cookie: session=aaa111

# IDOR test (as B, trying to access A's data):
GET /api/profile/101
Cookie: session=bbb222
```

If you get `200` with A's data — IDOR confirmed.

### Step 4 — Also Test with No Session

Some endpoints don't require authentication at all. Try without any `Cookie`/`Authorization` header.

---

## Tricky IDOR Patterns

### IDOR in POST Body vs URL

Most people check URL parameters. Don't forget request bodies:

```json
POST /api/update-profile
{
  "user_id": 1234,    ← change this
  "email": "new@email.com"
}
```

### IDOR in HTTP Headers

```
X-User-ID: 1234
Account-ID: 5678
```

### IDOR via Parameter Pollution

```
GET /api/profile?user_id=MINE&user_id=VICTIM
```

### IDOR in Indirect References

The app returns a key that maps to a resource — you don't see a direct ID, but:

```
GET /api/export          → returns a task_id=abc123
GET /api/download/abc123 ← can you guess/enumerate task IDs of other users?
```

### IDOR in Hashed IDs

Hashes seem opaque but if they're just `MD5(user_id)`, they're trivially reversible:

```python
import hashlib
# MD5 of IDs 1-1000
for i in range(1, 1001):
    print(hashlib.md5(str(i).encode()).hexdigest())
```

---

## Non-Numeric IDs (UUIDs)

UUIDs look unguessable — but:
- Can you enumerate them through a list endpoint?
- Can you find them in another response (email content, shared links)?
- Is the UUID format v1 (time-based)? Those can sometimes be predicted.

```
GET /api/users                → may return list of all user UUIDs
GET /api/activity-feed        → may leak other users' UUIDs in activity data
```

---

## Automating IDOR Testing

In Burp Intruder or with ffuf:

```bash
# Enumerate a numeric ID range
ffuf -u "https://example.com/api/invoice/FUZZ" \
  -w <(seq 1 10000) \
  -H "Cookie: session=YOUR_SESSION" \
  -mc 200 \
  -o idor_results.json

# In Burp Intruder: position on the ID, payload type = Numbers, range 1-10000
```

---

## Impact and Severity

| Scenario | Severity |
|----------|----------|
| View any user's PII (name, email, phone) | High |
| View any user's financial data (invoices, transactions) | High–Critical |
| Modify another user's account (email, password reset) | Critical |
| Delete another user's content | Medium–High |
| Access admin features | Critical |
| View system-level resources (config, logs) | Critical |

---

## Reporting IDOR

Key elements for a good IDOR report:
1. Two distinct accounts and their IDs
2. Exact request showing the ID substitution
3. Response showing the other user's data
4. Clear impact statement — what data is exposed? What can be done with it?

---

*Related: [[A01-Broken-Access-Control]] | [[A07-Auth-Failures]]*

*Sources: OWASP Testing Guide v4, PortSwigger Web Security Academy, HowToHunt*
