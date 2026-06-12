# How to Use This Vault

Welcome to the **Web Security Learning Vault** — a structured, practical guide to understanding web security from the ground up.

---

## Who Is This For?

This vault is built for people who already understand the basics of networking (what an IP is, what a browser does) but are stepping into **security** for the first time. You don't need a security background — just curiosity and a willingness to break things in a safe environment.

---

## How the Vault Is Organized

```
websecurity-fun/
├── 00-Setup/          ← You are here. Start here.
├── 01-Basics/         ← How the web actually works (read this first)
├── 02-Recon/          ← Information gathering before attacking
├── 03-OWASP-Top-10/  ← The 10 most critical web vulnerabilities
├── 04-Vuln-Classes/   ← Deep dives into each vulnerability type
├── 05-Tools/          ← Burp Suite, ffuf, nmap, and more
├── 06-Labs/           ← Walkthroughs, CTF write-ups, practice
└── 99-Cheatsheets/    ← Quick reference sheets
```

**Start at `01-Basics` and work top-to-bottom.** Each section builds on the previous one.

---

## How to Read the Notes

- **Overview pages** give you the full picture of a topic. Start here.
- **Deep-dive pages** (linked with `[[double brackets]]`) go into specifics when you need them.
- **Callout boxes** are used like this:

> [!NOTE]
> Extra context or a tip.

> [!WARNING]
> Common mistakes or dangerous misconceptions.

> [!TIP]
> Practical advice from real-world experience.

---

## Recommended Learning Path

1. [[01-Basics/How-the-Web-Works]] — understand the request/response cycle
2. [[01-Basics/HTTP-Deep-Dive]] — HTTP is the language of web hacking
3. [[01-Basics/Web-Architecture]] — know what you're attacking
4. [[01-Basics/Browser-Security-Model]] — why browsers restrict what they do
5. Then move to `02-Recon` → `03-OWASP-Top-10`

---

## Tools You'll Need

- **Burp Suite Community** — intercepting proxy (free version is enough to start)
- **A browser** — Firefox with FoxyProxy recommended
- **TryHackMe / HackTheBox account** — for practice labs
- This vault works alongside hands-on practice — reading alone won't make you good at security.

---

> [!TIP]
> Open the Obsidian graph view (`Ctrl/Cmd + G`) to see how topics connect. Web security is not linear — it's a web of interconnected concepts.
