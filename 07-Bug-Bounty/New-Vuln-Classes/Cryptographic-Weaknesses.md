# Cryptographic Weaknesses

← [[VRT-Overview]]

---

Crypto bugs are a VRT category that OWASP mostly brushes past. The VRT breaks down cryptographic weaknesses into specific testable patterns — each with its own P-rating.

---

## VRT Severity Reference

| Category | Vulnerability | Variant | P-Rating |
|---------|--------------|---------|---------|
| Key Reuse | Inter-Environment | Same key used in prod + staging | P2 |
| Broken Cryptography | Use of Broken Cryptographic Primitive | MD5/SHA1/DES/RC4 | P3 |
| Insecure Key Generation | Insufficient Key Space | | P3 |
| Cryptographic Signature | Insufficient Verification | | Varies |
| Insecure Key Generation | Insufficient Key Stretching | | Varies |
| Insecure Key Generation | Improper Asymmetric Prime/Exponent Selection | | Varies |
| Insecure Key Generation | Improper Key Exchange Without Entity Auth | | P4 |
| Insufficient Entropy | Predictable IV | | P4 |
| Insufficient Entropy | Predictable PRNG Seed | | P4 |
| Insufficient Entropy | Small Seed Space in PRNG | | P4 |
| Side-Channel Attack | Padding Oracle Attack | | P4 |
| Side-Channel Attack | Timing Attack | | P4 |
| Use of Expired Cryptographic Key | | | P4 |
| Key Reuse | Lack of Perfect Forward Secrecy | | P4 |
| Insufficient Entropy | IV Reuse | | P5 |
| Weak Hash | Lack of Salt | | Varies |
| Weak Hash | Predictable Salt | | P5 |
| Weak Hash | Predictable Hash Collision | | Varies |
| Key Reuse | Intra-Environment | | P5 |
| Side-Channel Attack | Timing Attack (minor) | | P5 |

---

## Padding Oracle Attacks (P4)

A classic side-channel attack against CBC mode encryption. If the server reveals whether a decryption attempt had a padding error vs a MAC error, an attacker can decrypt ciphertext without the key.

**How to detect:**
- The app uses CBC mode encryption for cookies, tokens, or data
- Error responses differ for "bad padding" vs "bad data" — even slightly (different status code, different message, different response time)

**Testing:**
```bash
# padbuster — automated padding oracle exploitation
padbuster https://example.com/profile "ENCRYPTED_COOKIE_VALUE" 8 \
  -cookies "auth=ENCRYPTED_COOKIE_VALUE" \
  -encoding 0

# If vulnerable: padbuster will decrypt the cookie value and show plaintext
```

**Example scenario:**
```
Cookie: auth=AXfgHj+KL2mNo...  (CBC-encrypted user data)
Modify last byte → server responds differently for padding error
→ can decrypt the entire cookie byte-by-byte
→ forge any user cookie without knowing the key
```

---

## Timing Attacks (P4)

If a comparison function takes different amounts of time depending on input, an attacker can leak information via response timing.

**Classic example — comparing tokens with `==`:**
```python
# Vulnerable: short-circuits on first mismatch
if user_token == expected_token:
    ...

# Safe: constant-time comparison
import hmac
if hmac.compare_digest(user_token, expected_token):
    ...
```

**Testing:**
```python
import requests, time, statistics

def measure(token):
    start = time.perf_counter()
    requests.post('/verify', data={'token': token})
    return time.perf_counter() - start

# If timing differs significantly between wrong prefix vs wrong suffix:
# The comparison is not constant-time
wrong_prefix = 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX'
wrong_suffix = 'AXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX'  # first char matches

prefix_times = [measure(wrong_prefix) for _ in range(50)]
suffix_times = [measure(wrong_suffix) for _ in range(50)]

print(f"Prefix avg: {statistics.mean(prefix_times):.4f}")
print(f"Suffix avg: {statistics.mean(suffix_times):.4f}")
```

In practice, network jitter makes this unreliable over the internet. But local/LAN environments and some specific functions (e.g., password reset token validation) can still be exploited.

---

## Broken Cryptographic Primitives (P3)

Using algorithms that are cryptographically broken:

| Algorithm | Status | Problem |
|-----------|--------|---------|
| MD5 | Broken | Collision attacks, trivially fast to brute-force |
| SHA1 | Deprecated | Collision demonstrated (SHAttered attack, 2017) |
| DES | Broken | 56-bit key, brutable in hours |
| 3DES (TDEA) | Deprecated | Sweet32 attack, slow |
| RC4 | Broken | Biases in keystream, NOMORE attack |
| ECB mode | Broken | Identical plaintext → identical ciphertext |

**How to detect:**
- Password hashes in a database dump — 32-char hex = MD5, 40-char hex = SHA1
- API tokens or session IDs that are suspiciously short or structured
- JWT with `alg: RS256` using 512-bit key
- TLS negotiation accepting RC4 or DES cipher suites

```bash
# Check TLS cipher suites
testssl.sh https://example.com

# Identify password hash type
hashid '$2y$10$...'    ← bcrypt ✅
hashid '5f4dcc3b5aa765d61d8327deb882cf99'    ← MD5 ❌
```

---

## IV Reuse / Predictable IV (P4/P5)

In CBC mode, reusing the same IV (Initialization Vector) with the same key leaks plaintext relationships. In CTR mode, IV reuse is catastrophic — XOR the two ciphertexts and you get the XOR of two plaintexts.

**Testing:**
- Capture multiple encrypted values from the app (cookies, tokens)
- If they share a common prefix in the ciphertext, the IV may be reused or predictable
- If the IV is static (same bytes every time), it's reused

---

## Predictable PRNG / Weak Entropy (P4)

If security-critical values (session IDs, CSRF tokens, password reset tokens, API keys) are generated from a weak random number generator:

**Red flags:**
- Token based on timestamp: `reset_token = md5(time())`
- Sequential IDs used as tokens
- Short tokens (< 128 bits of entropy)
- Base64 of a predictable seed

**Testing:**
```python
# Collect multiple tokens, look for patterns
tokens = ['request_1_token', 'request_2_token', 'request_3_token']
# Are they sequential? Time-based? Small numeric range?
```

```bash
# Burp Sequencer — analyzes randomness of tokens
# In Burp: Proxy → HTTP history → right-click token request → "Send to Sequencer"
# Run live capture, then analyze randomness
```

---

## Key Reuse (P2/P5)

**Inter-environment (P2):** The same cryptographic key is used in both production and staging. A staging environment is typically less locked down — if you can recover the key in staging, you can forge production tokens.

**Intra-environment (P5):** Different functions sharing the same key (e.g., using the same key for both JWT signing and AES encryption). Less impactful but still a design flaw.

---

*Related: [[A02-Cryptographic-Failures]] | [[JWT-Attacks]] | [[VRT-Overview]]*

*Sources: Bugcrowd VRT v1.18, PortSwigger Web Security Academy, Cryptopals*
