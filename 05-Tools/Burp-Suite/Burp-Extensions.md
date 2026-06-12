# Burp Suite Extensions

← [[Burp-Overview]]

---

Extensions massively expand what Burp can do. Install via the **BApp Store** (Extender → BApp Store).

---

## Essential Extensions (Free)

### Param Miner

**What it does:** Discovers hidden parameters that aren't in the UI — parameters the backend accepts but doesn't document.

```
Right-click request → Extensions → Param Miner → Guess params
```

Useful for: finding hidden debug params, undocumented features, mass assignment parameters.

---

### Autorize

**What it does:** Automatically re-runs requests with a lower-privileged session cookie. Highlights when a privilege escalation or IDOR is possible.

```
1. Configure with a low-privileged cookie
2. Browse the app as an admin/high-priv user
3. Autorize replays each request with the low-priv cookie automatically
4. Green = same response (IDOR/priv esc) | Red = different response (protected)
```

This is the fastest way to find access control issues at scale.

---

### Logger++

**What it does:** More powerful logging than the built-in Logger. Filter, search, and export logs.

Useful when you need to search through hundreds of requests for a specific pattern.

---

### Turbo Intruder

**What it does:** Intruder on steroids — no rate limiting, massively faster, uses HTTP/2 pipelining. Also great for **race condition** testing.

```python
# Turbo Intruder script for race condition:
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                            concurrentConnections=30,
                            requestsPerConnection=100,
                            pipeline=True)
    for i in range(30):
        engine.queue(target.req)
    engine.start(timeout=10)
```

---

### JWT Editor

**What it does:** Decode, modify, and sign JWT tokens from within Burp. Supports algorithm confusion attacks, `alg: none`, and weak secret cracking.

```
Right-click request with JWT → Extensions → JWT Editor → JSON Web Token tab
```

See [[JWT-Attacks]] for the theory.

---

### Hackvertor

**What it does:** Complex encoding/decoding chains in request parameters. Tag-based encoding that updates dynamically.

```
<@base64><@urlencode>payload<@/urlencode><@/base64>
```

Useful for: WAF bypasses, SSRF filter bypass, injection in encoded contexts.

---

### ActiveScan++

**What it does:** Extends the active scanner with additional checks for injection, header issues, and more.

Requires Pro for full use but has some passive checks in Community.

---

### CSRF Scanner

**What it does:** Passively checks for missing CSRF tokens on state-changing requests as you browse.

---

### Retire.js

**What it does:** Flags JavaScript libraries with known vulnerabilities as you browse. Compares against the Retire.js database.

---

## Pro-Only Extensions (Worth Knowing About)

### DOM Invader (Built-in)

Instruments the browser to automatically detect DOM XSS. Injects canary values and monitors dangerous sinks.

### Collaborator Everywhere

Injects Collaborator payloads into all HTTP headers automatically. Detects blind SSRF, blind XSS, and injection points that trigger out-of-band connections.

---

## Installing Extensions

1. Extender → BApp Store → search → Install
2. For non-BApp extensions: Extender → Extensions → Add → select `.jar` or `.py` file

---

## Python/Jython Extensions

Some extensions require Python. Install Jython:

1. Download `jython-standalone-x.x.x.jar`
2. Extender → Options → Python Environment → select Jython jar

---

*Back to [[Burp-Overview]] | Related: [[Burp-Intruder]]*
