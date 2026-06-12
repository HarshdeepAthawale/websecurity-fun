# websecurity-fun

Hey, welcome.

I've been in web security for a couple of years now and I've had a lot of juniors ask me where to start. The honest answer is that the internet is full of good content — but it's scattered everywhere, inconsistent in quality, and nobody tells you what order to learn things in.

So I made this.

This is a structured set of notes I wish I had when I was starting out. It covers web security from the ground up — how the web actually works, what you're attacking, and how the most common vulnerabilities work in practice. No fluff, no paywalls, no 4-hour YouTube videos where the good part is at the 2:47 mark.

It's built as an [Obsidian](https://obsidian.md) vault so everything is connected — follow the links, use the graph view, build your mental model.

---

## Who this is for

You already know what a browser is and roughly how the internet works. You're curious about security but don't know where to start or what actually matters.

## How to use it

1. Download [Obsidian](https://obsidian.md) (free)
2. Clone this repo
3. Open the folder as a vault in Obsidian
4. Start at `00-Setup/How-to-use-this-vault.md`

## What's inside

```
00-Setup/          → start here
01-Basics/         → HTTP, cookies, browser security model, SOP, CORS, CSP
02-Recon/          → subdomain enum, fingerprinting, JS recon, OSINT, ffuf
03-OWASP-Top-10/   → all 10 entries with real testing guidance
04-Vuln-Classes/   → deep dives with payloads and exploitation
   ├── XSS/        → reflected, stored, DOM, filter bypass
   ├── SQL-Injection/ → error-based, blind, to-RCE
   ├── SSRF/       → techniques + full bypass reference
   ├── IDOR, CSRF, Path Traversal, XXE, SSTI
   ├── JWT Attacks, OAuth, HTTP Request Smuggling
   ├── GraphQL, Business Logic, Prototype Pollution
05-Tools/          → Burp Suite (proxy, intruder, extensions)
06-Labs/           → walkthroughs and write-ups (growing)
99-Cheatsheets/    → XSS payloads, SQLi, SSRF bypass, WAF bypass, recon commands
```

**66 notes** across all sections. Everything is cross-linked — follow `[[wikilinks]]` in Obsidian and use the graph view to see how it all connects.

Reading alone won't make you good at this. Use these notes alongside actual practice — TryHackMe, HackTheBox, PortSwigger Web Academy labs. The notes tell you the theory; the labs build the muscle memory.

---

Good luck. Break things responsibly.
