# A01 — Broken Access Control

← [[OWASP-Overview]]

---

Broken Access Control is the #1 web security risk. It moved to the top in 2021 because it's found in **94% of tested applications**. The concept is simple: a user can do or see something they shouldn't be able to.

---

## The Core Idea

Access control answers one question: **"Is this user allowed to do this action on this resource?"**

When access control is broken, the answer should be "no" but the application responds "yes."

---

## Attack Patterns

### IDOR — Insecure Direct Object Reference

You access a resource by its ID. The app doesn't check if you own it.

```
# You access your own profile:
GET /api/user/1234/profile

# You change the ID:
GET /api/user/1235/profile  ← returns another user's data
```

IDOR is the most common specific bug in this category. Deep dive: [[IDOR]]

### Vertical Privilege Escalation

A low-privileged user accessing admin-level functionality.

```
# Normal user accessing admin endpoint:
GET /admin/users/list        ← should return 403, returns 200

# Or:
POST /api/user/update
{"role": "admin"}            ← user can set their own role
```

### Horizontal Privilege Escalation

A user accessing another user's resources at the same privilege level. Same as IDOR in most cases.

### Path Traversal / Directory Traversal

Using `../` sequences to escape the intended directory.

```
GET /download?file=report.pdf
GET /download?file=../../../../etc/passwd
```

Deep dive: [[Path-Traversal]]

### Forceful Browsing

Accessing URLs that aren't linked but exist. The app relies on obscurity rather than actual access checks.

```
https://example.com/admin/export_users
https://example.com/reports/financial_2023.pdf
```

### CORS Misconfiguration Enabling Unauthorized Access

See [[CORS]].

### JWT / Token Manipulation

Modifying a token to escalate privileges. Deep dive: [[JWT-Attacks]]

---

## Why It Happens

Developers check access in some places but not all:
- The UI hides the button from non-admins, but the underlying API endpoint has no server-side check
- Access is checked at the route level but not per-object (`/admin/*` is protected, but `/api/users/:id` isn't)
- The mobile app has stricter checks than the web app — they share the same API
- Access control logic is copy-pasted and some copies get updated, some don't

---

## Testing Approach

### 1. Map Every Endpoint That Takes a Resource ID

```
GET /api/posts/{id}
GET /api/user/{id}/settings
GET /invoice/{id}/download
```

For each one: does it enforce ownership?

### 2. Test with Two Accounts

Create two test accounts (User A and User B). Log in as User A, get a resource ID. Log in as User B and try to access User A's resource.

```bash
# In Burp: log requests as User A, then replay with User B's session token
```

### 3. Check Vertical Escalation

With a non-admin account, try every admin endpoint you discovered during recon:
- `/admin/*`
- `/api/v1/admin/*`
- Any endpoint that showed 403 during unauthenticated testing

### 4. Try Parameter Manipulation

```
# Role/permission parameters
?role=admin
?admin=true
{"isAdmin": true}
{"role": "superuser"}
```

### 5. Method-Based Bypass

Protected endpoint with GET → try POST, PUT, PATCH, DELETE.

---

## Common Finding Examples

| Finding | Severity |
|---------|----------|
| View any user's PII by changing user ID | High |
| Download any user's invoice PDF | Medium–High |
| Access admin panel with non-admin account | Critical |
| Modify another user's account settings | High |
| Read unowned API keys / tokens | High |
| Access reports from other organizations | High |

---

## Mitigation (For Devs)

- Enforce access control at the **server side**, always — never rely on the client
- **Deny by default** — if a user's permission isn't explicitly granted, deny
- Check ownership on every object access, not just at the route level
- Log access control failures and alert on suspicious patterns
- Use consistent access control mechanisms — don't implement it differently per endpoint

---

*Related: [[IDOR]] | [[Path-Traversal]] | [[JWT-Attacks]] | [[OWASP-Overview]]*

*Sources: OWASP Top 10 2021, OWASP Testing Guide v4, PortSwigger Web Security Academy*
