# Burp Intruder

← [[Burp-Overview]]

---

Intruder automates sending many requests with varying payloads. It's how you brute force, fuzz, and enumerate at scale from within Burp.

> [!NOTE]
> Intruder is rate-limited in Community Edition (intentionally slow). For serious fuzzing, use [[ffuf.md]] instead. Intruder is still useful for smaller tasks and for its tight Burp integration.

---

## Setting Up an Intruder Attack

1. Right-click a request → **"Send to Intruder"** (or `Ctrl+I`)
2. Go to **Positions** tab
3. Mark the position(s) you want to fuzz with `§payload§`
4. Go to **Payloads** tab, configure your payload list
5. Click **"Start attack"**

---

## Attack Types

### Sniper (Most Common)

One payload list, one position at a time. If you have multiple positions, it cycles through each position independently.

Use for: testing one parameter at a time, IDOR enumeration, fuzzing a single field.

### Battering Ram

One payload list, all positions get the same payload simultaneously.

Use for: when you want to send the same value to multiple parameters at once.

### Pitchfork

Multiple payload lists, one per position. List 1 goes to Position 1, List 2 goes to Position 2, in parallel.

Use for: username + password pairs (list 1 = usernames, list 2 = corresponding passwords from a breach dump).

### Cluster Bomb

Multiple payload lists, every combination is tested.

Use for: brute force with username + password where you want all combinations.
Warning: can generate huge request counts (list1_size × list2_size).

---

## Payload Types

### Simple List

Paste or load a wordlist. One item per line.

### Numbers

Generate a range of numbers:
- From 1 to 10000, step 1 → enumerate IDs
- From 0000 to 9999 → OTP brute force

### Character Frobber

Modifies each character position in the original payload one at a time. Useful for fuzzing token formats.

### Brute Forcer

Generate all combinations of a character set up to a specified length.

### Bit Flipper

Flips each bit in the original value. Useful for binary format manipulation.

---

## Common Intruder Use Cases

### IDOR Enumeration

```
Position: GET /api/invoice/§1234§
Payload type: Numbers, 1 to 50000, step 1

Filter results:
- Response length differs → something there
- Status 200 with different content → valid invoice
```

### OTP Brute Force

```
Position: POST /verify-otp → {"code": "§000000§"}
Payload type: Numbers, 0 to 999999, formatted as 6 digits with leading zeros

Look for: Different response body or redirect on the correct code
```

### Username Enumeration

```
Position: POST /login → username=§user§&password=wrongpassword
Payload type: Simple list → common-usernames.txt

Filter by: Response length difference (valid username → different error message)
```

### Fuzzing Parameters

```
Position: GET /page?id=§FUZZ§
Payload type: Simple list → common test strings:
  '
  "
  <script>
  {{7*7}}
  ../../../etc/passwd
  null
  true
  false
  0
  -1
  99999999
```

---

## Reading Intruder Results

The attack results table shows: request number, status code, response length, response time.

**Filtering tricks:**
- Click "Length" column header to sort — unusually large/small responses stand out
- Click "Status" to group by response code
- Right-click → "Show response in browser" to see the rendered result
- Use the "Filter" bar to show only certain status codes

---

## Grep — Match

In Options tab → Grep - Match:
Add strings to highlight in results: `password`, `admin`, `error`, `true`, `success`.

Useful for quickly spotting interesting responses without reading each one.

---

*Back to [[Burp-Overview]] | Related: [[Burp-Extensions]]*
