# XSS — Cross-Site Scripting

**Overview** | [[Reflected-XSS]] | [[Stored-XSS]] | [[DOM-XSS]] | [[XSS-Filter-Bypass]]

---

XSS is when an attacker injects malicious JavaScript into a page that gets executed in another user's browser. It's one of the most common web vulnerabilities and, in the right context, it's critical — account takeover, session theft, credential harvesting, and keylogging are all possible.

---

## The Core Idea

The browser trusts all JavaScript that runs in the context of a page. If an attacker can inject JavaScript into a legitimate page, the browser executes it with full trust — with access to cookies, DOM content, local storage, and the ability to make requests as the victim.

---

## Three Types of XSS

| Type | Where payload lives | Who triggers it | Persistence |
|------|---------------------|-----------------|-------------|
| Reflected | In the URL / request | Victim clicks a crafted link | Not stored |
| Stored | In the database | Anyone who views the content | Permanent |
| DOM-based | In client-side JS | Victim loads the page | Not stored |

---

## What You Can Do with XSS

The impact depends entirely on what the application does.

| Action | Impact |
|--------|--------|
| Steal `document.cookie` | Session hijacking / account takeover |
| Make requests as the victim | CSRF bypass (since the request comes from the same origin) |
| Capture keystrokes | Credential theft |
| Redirect to phishing page | Credential phishing |
| Read DOM content | Extract visible sensitive data |
| Modify the page | Fake login form, defacement |
| Exploit browser vulnerabilities | Potential local file access |
| XSS + CSRF | Perform state-changing actions as the victim |

> [!NOTE]
> `HttpOnly` cookies can't be stolen via `document.cookie`, but XSS can still *use* them — the attacker's injected script can make authenticated requests on behalf of the victim.

---

## XSS Contexts

Where the payload lands determines how you write it.

### 1. HTML Body

```html
<!-- Input reflected between tags -->
<p>Hello, <INPUT>!</p>

<!-- Payload -->
<script>alert(1)</script>
<!-- or -->
<img src=x onerror=alert(1)>
```

### 2. HTML Attribute (Quoted)

```html
<input value="<INPUT>">

<!-- Break out of the attribute first -->
" onmouseover="alert(1)
"><script>alert(1)</script>
```

### 3. JavaScript String

```html
<script>var name = '<INPUT>';</script>

<!-- Break out of the string -->
'-alert(1)-'
';alert(1)//
```

### 4. Inside a URL Attribute (`href`, `src`, `action`)

```html
<a href="<INPUT>">Click</a>

<!-- javascript: URI -->
javascript:alert(1)
```

### 5. Inside a `<script>` Block (Unquoted)

```html
<script>var x = <INPUT>;</script>

<!-- Just inject JS directly -->
alert(1)
</script><script>alert(1)</script>
```

---

## The Basic Test Flow

1. Find a reflection point — any place where your input appears in the response
2. Check the context — what surrounds your input in the HTML source?
3. Craft a payload that escapes that context
4. Check if filters are in place and bypass them
5. Escalate to a meaningful payload (cookie theft, etc.)

```bash
# Start with a unique marker to find where it reflects
?name=XSS_TEST_123abc

# View source, search for XSS_TEST_123abc
# Note what HTML surrounds it
# Then craft your payload based on the context
```

---

## Common Filters and Bypasses

See [[XSS-Filter-Bypass]] for a complete reference. Quick overview:

| Filter | Bypass |
|--------|--------|
| `<script>` blocked | `<img src=x onerror=alert(1)>` |
| `alert` blocked | `confirm(1)`, `prompt(1)`, `alert\`1\`` |
| Quotes filtered | Use template literals: `` ` `` |
| Spaces filtered | `<img/src=x/onerror=alert(1)>` |
| `javascript:` blocked | `jAvAsCrIpT:alert(1)` |
| Encoding applied | Double-encode, HTML entity encode |

---

## Content Security Policy

A strong CSP can neuter XSS by preventing injected scripts from running. Always check:

```bash
# Is there a CSP?
curl -sI https://example.com | grep -i "content-security-policy"
```

See [[CSP]] for bypass techniques.

---

## Deep Dives

- [[Reflected-XSS]] — URL-based, requires victim to click a link
- [[Stored-XSS]] — persists in the DB, most impactful
- [[DOM-XSS]] — client-side only, tricky to find
- [[XSS-Filter-Bypass]] — payloads for bypassing WAFs and filters

---

## Quick Reference

Payloads cheatsheet: [[XSS-Payloads]]

---

*Related: [[CSP]] | [[Browser-Security-Model]] | [[A03-Injection]]*

*Sources: PortSwigger Web Security Academy, OWASP Testing Guide v4*
