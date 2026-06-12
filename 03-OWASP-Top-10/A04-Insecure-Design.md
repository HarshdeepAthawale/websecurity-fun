# A04 — Insecure Design

← [[OWASP-Overview]]

---

Insecure design covers flaws in the **architecture and logic** of an application — not implementation bugs, but conceptual failures in how the system was designed. You can't patch your way out of insecure design; the feature needs to be rethought.

---

## The Core Idea

Most vulnerabilities in the other OWASP categories are implementation bugs — a developer wrote code that could have been written securely. Insecure design is different: the security problem is in the **concept** itself.

**Example:**
- Storing passwords as MD5 → implementation bug, fix it by changing the algorithm
- Designing a "security question" password reset that uses your mother's maiden name → insecure design, the whole mechanism is fundamentally weak regardless of implementation

---

## Business Logic Flaws

The biggest category under insecure design. The app's logic can be abused in ways the developers didn't anticipate.

### Price Manipulation

```
# E-commerce: buy item for negative price?
POST /cart/add
{"product_id": 1, "quantity": -1, "price": -100}

# Coupon stacking when coupons should be mutually exclusive
# Applying a discount code multiple times
```

### Workflow Bypass

```
# Multi-step process — can you skip step 2?
Step 1: /order/review    ← skip this
Step 2: /order/pay       ← go directly here
Step 3: /order/confirm

# Account verification — can you skip email verification?
# Can you access paid features without completing payment?
```

### Race Conditions

Multiple requests hitting the same endpoint simultaneously, exploiting a window between a check and an action.

```
# Classic: gift card redemption
# Code checks: is card valid? Yes.
# [race window — two threads both pass the check]
# Both redeem the same card simultaneously → double spending
```

See [[Race-Conditions]] for exploitation techniques.

### Parameter Tampering

```
# Can you change your own user role in a profile update?
PUT /api/user/me
{"name": "Hacker", "role": "admin"}

# Can you change the price in a checkout request?
POST /checkout
{"items": [{"id": 1, "price": 0.01}]}

# Can you reference internal/admin resources from user-facing features?
```

### Insufficient Anti-Automation

- No rate limiting on OTP entry → brute-force a 6-digit OTP in 1,000,000 attempts
- No CAPTCHA on account creation → mass account creation
- Bulk API operations without limits → enumerate every user ID in one loop

---

## Flawed Trust Models

**Trusting the client too much:**
```javascript
// Client sends the price to the server and server trusts it
// Fix: always calculate price server-side
```

**Trusting HTTP headers from untrusted sources:**
```
# App uses X-Forwarded-For to determine the user's location for access control
# But X-Forwarded-For is attacker-controlled
```

**Insufficient separation between user types:**
```
# Regular user and admin use the same API
# Admin endpoints are only "hidden" in the UI, not actually protected
```

---

## Missing Threat Modeling

Insecure design typically comes from not asking "how could this be abused?" during design.

**Questions that should be asked:**
- What happens if a user sends unexpected values?
- Can a user access resources belonging to another user?
- What happens if the workflow is done out of order?
- What if the user runs this operation simultaneously in two browser tabs?
- Is there any combination of features that produces unintended behavior?

---

## Finding Business Logic Flaws

Unlike other vuln classes, there's no universal payload list. You need to understand the app and think creatively.

**Approach:**
1. Map every feature and its intended workflow
2. Ask "what if" for every step: What if I skip this? What if I repeat this? What if I use negative values?
3. Look at parameters and ask: what would happen if I change this value? What if I change the type?
4. Look for **implicit trust** — places where the app assumes something is true without checking
5. Test **combinations** of features — discount + payment + refund flow together

---

## Real-World Examples

| Finding | Description |
|---------|-------------|
| Negative quantity in shopping cart | Items subtracted from total, negative balance credited |
| OTP brute-forceable | 6-digit SMS OTP with no rate limit = 1M attempts |
| Skip payment step | Direct POST to order confirmation skips payment validation |
| Referral abuse | Self-referral or circular referral for infinite credits |
| Free trial abuse | Account deletion + re-registration resets trial |
| Concurrent request abuse | Transfer same balance in parallel requests |

---

*Related: [[A01-Broken-Access-Control]] | [[OWASP-Overview]]*

*Sources: OWASP Top 10 2021, PortSwigger Web Security Academy*
