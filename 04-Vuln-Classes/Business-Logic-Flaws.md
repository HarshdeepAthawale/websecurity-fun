# Business Logic Flaws

← [[A04-Insecure-Design]]

---

Business logic flaws are vulnerabilities in the application's intended functionality. The features work as implemented — but not as intended. No scanner catches these. You need to think like an attacker who understands what the app is *supposed* to do.

---

## Why They're Underrated

Logic flaws don't have signatures. You can't grep for them. A WAF doesn't know that sending `quantity: -1` is wrong. The only defense is understanding the business intent and testing every assumption.

They're also often high-severity because they go straight to the business impact: financial fraud, data theft, unauthorized access — without needing a "traditional" vulnerability.

---

## Category 1 — Price and Value Manipulation

### Negative Quantities

```
POST /cart/add
{"product_id": 1, "quantity": -5, "price": 9.99}

# Expected: error
# Vulnerable: subtracts $49.95 from cart total
```

### Zero Price

```
POST /checkout
{"items": [{"id": 1, "price": 0}]}

# Expected: server ignores client price
# Vulnerable: server uses client-supplied price
```

### Integer Overflow / Underflow

Large quantities or prices that wrap around:
```
quantity: 9999999999    → might overflow to negative number
price: 4294967296       → 2^32, wraps to 0 on 32-bit integer
```

### Discount Abuse

```
# Apply the same coupon multiple times
# Stack multiple coupons when only one should apply
# Apply a % discount, then a fixed discount, then the % again
# Apply employee discount without being an employee (IDOR on discount endpoint)
```

---

## Category 2 — Workflow Bypass

### Skipping Steps

Multi-step processes often don't verify that previous steps were completed:

```
# 3-step checkout: Cart → Payment → Confirm
# Test: go directly to /checkout/confirm without /checkout/payment
# Vulnerable: order confirmed without payment
```

### Completing in Wrong Order

```
# Account upgrade flow: choose plan → pay → activate
# Test: activate endpoint first, then pay
# Vulnerable: premium access without payment
```

### Re-using Tokens / Actions

```
# Password reset token — can it be used twice?
# Gift card — can it be redeemed twice (race condition)?
# Email verification link — still valid after email change?
```

---

## Category 3 — Race Conditions

Two operations that should be atomic aren't. Classic: check-then-act without locking.

```
Thread A: Is gift card valid? → Yes
Thread B: Is gift card valid? → Yes
Thread A: Redeem gift card (-$100)
Thread B: Redeem gift card (-$100)
# Balance reduced by $200 from a $100 card
```

**Testing race conditions:**
```python
# Turbo Intruder — send 30 parallel requests simultaneously
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                            concurrentConnections=30,
                            requestsPerConnection=100,
                            pipeline=True)
    for i in range(30):
        engine.queue(target.req)
```

Look for:
- Gift card / coupon redemption
- Limited-quantity item purchase
- Balance transfers / payments
- Account creation limits (only 1 free trial)
- Rate-limited endpoints (only 5 attempts)

---

## Category 4 — Trust Assumptions

### Trusting Hidden Form Fields

```html
<input type="hidden" name="price" value="99.99">
<input type="hidden" name="role" value="user">
<input type="hidden" name="discount" value="0">
```

Change hidden fields in Burp. The server should never trust values the client sends.

### Trusting HTTP Methods

```
DELETE on /api/user/me → protected by CSRF token
POST on /api/user/me with method override header → no CSRF check
```

### Trusting Referrer for Access

```
# Access control: only allow if Referer is internal
Referer: https://admin.example.com/

# Attacker sets this Referer manually
curl -H "Referer: https://admin.example.com/" https://example.com/sensitive
```

---

## Category 5 — Insufficient Validation

### Type Juggling

```json
# PHP: "0" == false, 0 == "admin"
{"role": 0}      → might match "admin" in loose comparison
{"score": true}  → might be cast to 1 or "true"
```

### Validation on Wrong Side

```
# Validation: JS only — done in browser
# Remove JS from browser or intercept with Burp
# Bypass all client-side validation
```

### Only Validating First Occurrence

```
POST /update
role=user&role=admin    ← second value used by app, first validated
```

---

## Category 6 — Excessive Privileges

### Mass Assignment

When the app automatically maps request fields to model properties:

```json
# Intended update fields: name, email
PUT /api/user/me
{
  "name": "John",
  "email": "john@example.com",
  "role": "admin",          ← not supposed to be here
  "credit": 99999           ← not supposed to be here
}
```

Try adding extra fields to any PUT/PATCH/POST request. Use the schema from introspection or JS source to know what fields exist.

---

## Finding Logic Flaws

**Questions to ask for every feature:**
- What should happen? What if I skip a step?
- What if I send negative/zero/huge values?
- What if I repeat this action?
- What if two users do this simultaneously?
- What if I do this in reverse order?
- What fields does the API accept that the UI doesn't show?
- What assumptions does this feature make about the user's state?

---

*Related: [[A04-Insecure-Design]] | [[IDOR]] | [[A01-Broken-Access-Control]]*

*Sources: PortSwigger Web Security Academy, Bug Bounty Bootcamp*
