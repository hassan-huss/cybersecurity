# Python: Core Concepts

Python is the scripting language of choice for pentesters: a short script can check 10,000 usernames against a default-password list in seconds. This room builds on *Simple Demo* (variables, `if`, `while`) and adds **type conversion, f-strings, string methods, lists, dictionaries, more operators, and `for` loops**. The next room (*Building Scripts*) adds functions, error handling, files and libraries.

> This is the **TryHackMe "Python: Core Concepts" room** (Jr. Penetration Tester → Python Scripting Basics, room 2). Tasks 1–6 are covered (Task 7 is just the conclusion). Example scripts live on the VM in `/home/ubuntu/Core-Concepts/` — run and tweak each one.
>
> 🔗 Previous: Python: Simple Demo · Next: Python: Building Scripts

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Glossary](#glossary)
- [Quick review: print, variables, conditionals (Task 2)](#quick-review-print-variables-conditionals-task-2)
- [What's new: type conversion, f-strings, `+=` (Task 2)](#whats-new-type-conversion-f-strings--task-2)
- [Working with strings (Task 3)](#working-with-strings-task-3)
- [Lists and dictionaries (Task 4)](#lists-and-dictionaries-task-4)
- [Arithmetic and membership operators (Task 5)](#arithmetic-and-membership-operators-task-5)
- [Loops: `for` and `while` (Task 6)](#loops-for-and-while-task-6)
- [Pentest-flavoured cheat sheet](#pentest-flavoured-cheat-sheet)
- [Key takeaways](#key-takeaways)

---

## In plain English

A program is mostly two things: **holding data** and **repeating decisions over it**.

- **Holding data:** a *string* is text, a *list* is an ordered row of items (e.g. IPs to scan), a *dictionary* is a lookup table (port `22` → `"SSH"`).
- **Repeating decisions:** an `if` asks a yes/no question once; a **loop** asks it for every item. `for` = "do this for each thing in the pile"; `while` = "keep doing this until a condition stops being true".

So the 10,000-usernames job is: put them in a list, `for` each one, `if` it matches a weak password, record it. Everything in this room is a tool for that picture.

## Glossary

| Term | Plain meaning |
| --- | --- |
| **String (`str`)** | Text in quotes. A sequence of characters, so it can be indexed and looped over. |
| **Integer / float / bool** | Whole number / number with decimals / `True` or `False`. |
| **Type conversion** | Changing a value's type: `int("42")` → `42`. Needed because `input()` always returns text. |
| **f-string** | `f"User {name}"` — a string with variables dropped in via `{}`. |
| **Index** | A character's/item's position number. Starts at **0**. `-1` = last. |
| **Slice** | A piece of a string/list: `x[start:end]` (end excluded). |
| **Method** | A function that belongs to a value: `"hi".upper()`. |
| **List** | Ordered, changeable collection in `[ ]`. |
| **Dictionary** | Key → value pairs in `{ }`. Look up by key. |
| **Iteration** | One pass through a loop body. |
| **Augmented assignment** | Shorthand like `x += 1` for `x = x + 1`. |

---

## Quick review: print, variables, conditionals (Task 2)

```python
print("Hello World")        # text must be in quotes; # starts a comment
age = 30                    # variable = name holding a value
age = age + 1               # values can change
print(type(age))            # <class 'int'>
```

**Core data types**

| Type | Example | Meaning |
| --- | --- | --- |
| `str` | `"hello"` | Text |
| `int` | `42` | Whole number |
| `float` | `3.14` | Decimal number |
| `bool` | `True` / `False` | Logical value |
| `list` | `[1, 2, 3]` | Ordered collection |

**Conditionals** — `if` / `elif` / `else`, with a colon and an **indented** block:

```python
if age < 17:
    print("Not old enough")
elif age == 17:
    print("Almost")
else:
    print("Old enough")
```

| Comparison | Meaning | Logical | Meaning |
| --- | --- | --- | --- |
| `==` | equal | `and` | both true |
| `!=` | not equal | `or` | at least one true |
| `< > <= >=` | ordering | `not` | flips the boolean |

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:10px 14px;border-radius:4px;margin:12px 0;">
<strong>⚠️ <code>=</code> vs <code>==</code></strong> — <code>=</code> <em>stores</em> a value; <code>==</code> <em>compares</em>. Mixing them up is the classic beginner bug.
</div>

---

## What's new: type conversion, f-strings, `+=` (Task 2)

**Type conversion** — `input()` always gives a *string*, so convert before doing maths:

| Function | Example | Result |
| --- | --- | --- |
| `int()` | `int("42")` | `42` |
| `float()` | `float("3.14")` | `3.14` |
| `str()` | `str(42)` | `"42"` |
| `bool()` | `bool(0)` | `False` |

```python
text = input("Enter a port number: ")   # user types 443 -> "443" (a string)
port = int(text)                        # now an integer
print(port + 1)                         # 444
# text + 1 would be an ERROR: can't add a number to text
```

**f-strings** — put `f` before the quote and variables inside `{}`:

```python
username, port = "admin", 443
print("User", username, "is on port", port)   # old comma style
print(f"User {username} is on port {port}")   # f-string: same output, cleaner
```

**Augmented assignment** — shortcuts for updating a variable:

```python
count = 0
count += 1    # count = count + 1
count -= 1    # count = count - 1
count *= 2    # count = count * 2
count /= 4    # count = count / 4
```

---

## Working with strings (Task 3)

Strings drive nearly every security script (passwords, URLs, logs). Quotes can be `'single'`, `"double"` or `"""triple"""` (multi-line) — same type.

**Length:** `len("Tr0ub4dor")` → `9` (is the password long enough?).

**Indexing & slicing** — positions start at **0**:

| Index | 0 | 1 | 2 | 3 | 4 | 5 |
| --- | --- | --- | --- | --- | --- | --- |
| Char | P | y | t | h | o | n |

```python
word = "Python"
word[0]      # 'P'
word[-1]     # 'n'   (negative = count from the end)
word[0:3]    # 'Pyt' (start included, end EXCLUDED)
word[2:]     # 'thon'
word[:4]     # 'Pyth'
```

**Useful methods**

| Method | Does | Example → Result |
| --- | --- | --- |
| `.upper()` / `.lower()` | Change case | `"hello".upper()` → `"HELLO"` |
| `.strip()` | Trim leading/trailing whitespace | `" hi ".strip()` → `"hi"` |
| `.replace(a, b)` | Replace all `a` with `b` | `"cat".replace("c","b")` → `"bat"` |
| `.split(sep)` | Split into a **list** | `"a,b,c".split(",")` → `["a","b","c"]` |
| `.startswith(x)` / `.endswith(x)` | Prefix / suffix check | `"file.txt".endswith(".txt")` → `True` |
| `.count(x)` | Count occurrences | `"banana".count("a")` → `3` |

**Character checks** (return `True`/`False`, ideal inside `if`):

```python
char = "A"
char.isupper()   # True
char.islower()   # False
char.isdigit()   # False
char.isalpha()   # True  (letter?)
char.isalnum()   # True  (letter or digit?)
```

**`in` operator** — is a substring inside a string?

```python
url = "https://tryhackme.com/room/pythoncoreconcepts"
"tryhackme" in url     # True
"hackthebox" in url    # False
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:10px 14px;border-radius:4px;margin:12px 0;">
<strong>💡 Tip</strong> — methods return a <em>new</em> value; they don't change the original string. Use <code>name = name.strip()</code> if you want to keep the result.
</div>

---

## Lists and dictionaries (Task 4)

### Lists — ordered, changeable, `[ ]`

```python
ports = [22, 80, 443, 8080]
mixed = ["server1", 443, True]     # any types allowed

ports[0]       # 22
ports[-1]      # 8080
ports[1:3]     # [80, 443]  (slicing works like strings)
ports[0] = 2222   # change an element
```

| Method | Does |
| --- | --- |
| `.append(x)` | Add `x` to the end |
| `.remove(x)` | Remove the first `x` |
| `.pop(i)` | Remove **and return** item at index `i` |
| `.sort()` | Sort ascending |
| `.reverse()` | Reverse order |
| `len(list)` | Number of items |

```python
common_passwords = ["123456", "password", "admin", "letmein"]
if "password" in common_passwords:
    print("This password is in the common list.")
```

### Dictionaries — key → value, `{ }`

Like a real dictionary: look up the **key** (word) to get the **value** (definition).

```python
services = {22: "SSH", 80: "HTTP", 443: "HTTPS", 3306: "MySQL"}

services[22]                  # 'SSH'
services[8080] = "HTTP-Alt"   # add
services[22] = "OpenSSH"      # update
del services[3306]            # remove
if 22 in services:            # check a KEY exists
    print(f"Port 22 runs {services[22]}")
```

| Method | Returns |
| --- | --- |
| `.keys()` | All keys |
| `.values()` | All values |
| `.items()` | All key-value pairs (as tuples) |
| `.get(key, default)` | Value, or `default` if the key is missing |

```python
services.get(9999, "Unknown")   # 'Unknown' — safe
services[9999]                  # KeyError — crashes
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:10px 14px;border-radius:4px;margin:12px 0;">
<strong>⚠️ Square brackets vs <code>.get()</code></strong> — <code>d[key]</code> on a missing key raises an error and stops your script. When the key might not exist (e.g. an unknown port), use <code>.get(key, default)</code>.
</div>

> 🔭 Preview: the *Building Scripts* room's Password Strength Checker loads weak passwords into a **list** and maps strength levels to labels with a **dictionary**.

---

## Arithmetic and membership operators (Task 5)

Beyond `+ - * /`:

| Operator | Name | Example | Result |
| --- | --- | --- | --- |
| `**` | Exponent | `2 ** 8` | `256` |
| `//` | Floor division (round **down**) | `7 // 2` | `3` |
| `%` | Modulus (remainder) | `7 % 2` | `1` |

- `/` **always** returns a float: `7 / 2` → `3.5`; `//` → `3`.
- `%` is the even/odd test: `n % 2 == 0` means even.
- `**` shows up with key sizes: a 128-bit key has `2 ** 128` possibilities.

**Membership: `in` / `not in`** — works on strings, lists, dictionary keys:

```python
common = ["123456", "password", "qwerty", "letmein"]
pw = "qwerty"
if pw in common:
    print("This password is too common.")
if pw not in common:
    print("Good. Not in the common list.")
```

**Combining operators** — a "moderate" password = ≥ 8 chars **and** has a digit:

```python
password = "Tr0ubador"
length = len(password)
has_digit = any(char.isdigit() for char in password)   # explained in Building Scripts

if length >= 8 and has_digit:
    print("Moderate strength")
elif length >= 8 or has_digit:
    print("Weak, but has some merit")
else:
    print("Very weak")
```

`and` needs **both** true; `or` needs **at least one**. (Don't worry about the `any(...)` line yet — it's explained next room.)

---

## Loops: `for` and `while` (Task 6)

### `while` — repeat *while a condition is true*

```python
attempts, max_attempts = 0, 3
while attempts < max_attempts:
    password = input("Enter password: ")
    attempts += 1
    print(f"Attempt {attempts} of {max_attempts}")
```

### `for` — repeat *for each item in a sequence*

```python
targets = ["192.168.1.1", "192.168.1.2", "192.168.1.3"]
for ip in targets:               # ip = each element, one at a time
    print(f"Scanning {ip}...")
```

**Looping over a string** (char-by-char — password validation):

```python
for char in "S3cure!":
    if char.isdigit():
        print(f"Found digit: {char}")
    elif char.isupper():
        print(f"Found uppercase: {char}")
# Found uppercase: S
# Found digit: 3
```

**`range()`** — loop a set number of times:

| Call | Produces |
| --- | --- |
| `range(5)` | 0, 1, 2, 3, 4 |
| `range(1, 6)` | 1, 2, 3, 4, 5 |
| `range(0, 20, 5)` | 0, 5, 10, 15 (step 5) |

Stop is **excluded**, and counting starts at 0 — so `range(5)` is five numbers, not 0–5.

**Looping over a dictionary** with `.items()`:

```python
services = {22: "SSH", 80: "HTTP", 443: "HTTPS"}
for port, name in services.items():
    print(f"Port {port} = {name}")
```

### `break` and `continue`

| Keyword | Effect |
| --- | --- |
| `break` | Exit the **whole loop** immediately |
| `continue` | Skip the **rest of this iteration**, go to the next |

```python
for port in [22, 80, 443, 8080]:
    if port == 443:
        print(f"Port {port} found. Stopping scan.")
        break                    # 8080 is never checked
    print(f"Checked port {port}")

for line in ["admin", "", "root", "", "guest"]:
    if line == "":
        continue                 # skip blanks
    print(f"Processing: {line}")
```

### Which loop?

| Use `for` when… | Use `while` when… |
| --- | --- |
| You know what/how many to visit (a list, a range, a string) | The count depends on something that changes (wait for valid input, retry until connected) |

> ✏️ **Room exercise:** print every even number 0–20 → `for i in range(0, 21, 2): print(i)` (stop is excluded, so use `21`).

---

## Pentest-flavoured cheat sheet

| Goal | Python pattern |
| --- | --- |
| Is a password in a leaked list? | `if pw in common_passwords:` |
| Password long enough? | `len(pw) >= 8` |
| Has an uppercase / digit? | loop `for c in pw:` + `c.isupper()` / `c.isdigit()` |
| Split a CSV line | `line.split(",")` |
| Clean a wordlist line | `line.strip()` |
| Skip blank lines | `if line == "": continue` |
| Map port → service | `services.get(port, "Unknown")` |
| Stop at first hit | `break` |
| Retry up to N times | `while attempts < N:` |
| Visit every target | `for ip in targets:` |

---

## Key takeaways

- `input()` returns **text** — convert with `int()`/`float()` before maths.
- **f-strings** (`f"... {var} ..."`) are the clean way to print variables; `+=` is the clean way to count.
- Strings are sequences: **index from 0**, **slice `[start:end]` (end excluded)**, negative index counts from the end.
- String methods (`strip`, `split`, `lower`, `isdigit`…) are the backbone of any text-handling script.
- **Lists** = ordered items; **dictionaries** = key → value lookups; use `.get()` when a key might be missing.
- New operators: `**` power, `//` floor-divide, `%` remainder; `in` / `not in` check membership in strings, lists and dict keys.
- `for` = visit each item; `while` = repeat until a condition fails; `range(start, stop, step)` generates numbers (stop excluded); `break` exits, `continue` skips.

---

*Python Scripting Basics room 2. Next: Python: Building Scripts → Python: Pentesting Scripts.*
