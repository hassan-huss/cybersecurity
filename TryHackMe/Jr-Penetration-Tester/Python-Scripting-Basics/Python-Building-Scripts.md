# Python: Building Scripts

Knowing individual Python concepts is like knowing musical notes; this room is playing the song. It adds the four things that turn a snippet into a real script — **functions**, **error handling**, **file I/O** and **libraries** — then combines everything from both rooms into a working **Password Strength Checker**.

> This is the **TryHackMe "Python: Building Scripts" room** (Jr. Penetration Tester → Python Scripting Basics, room 3). Whole room (Tasks 1–7). Example scripts live on the VM in `/home/ubuntu/Building-Scripts/` (including `common_passwords.txt` and the finished `password_checker.py`).
>
> 🔗 Previous: [Python: Core Concepts](Python-Core-Concepts.md) · Next: Python: Pentesting Scripts

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Glossary](#glossary)
- [Functions (Task 2)](#functions-task-2)
- [Error handling (Task 3)](#error-handling-task-3)
- [Reading and writing files (Task 4)](#reading-and-writing-files-task-4)
- [Libraries and pip (Task 5)](#libraries-and-pip-task-5)
- [Capstone: Password Strength Checker (Task 6)](#capstone-password-strength-checker-task-6)
- [Pentest-flavoured cheat sheet](#pentest-flavoured-cheat-sheet)
- [Key takeaways](#key-takeaways)

---

## In plain English

Four upgrades, each solving one problem:

| Problem | Fix | One-liner |
| --- | --- | --- |
| I'm copy-pasting the same code everywhere | **Functions** | Write it once, name it, call it many times (like a vending machine: button in, drink out). |
| One bad input crashes the whole script | **`try` / `except`** | Attempt the risky thing; if it fails, run a fallback instead of dying. |
| I need to read a wordlist / write a report | **Files + `with`** | Open, use, and auto-close a file safely. |
| I don't want to write everything myself | **Libraries** | `import` code other people already wrote; `pip install` for extras. |

The capstone shows them cooperating: load a wordlist from a **file**, wrap that in **error handling**, score a password in a **function** using the **`string` library**, and log the result to a file.

## Glossary

| Term | Plain meaning |
| --- | --- |
| **Function** | Reusable named block of code: takes input, does work, optionally returns a result. |
| **Parameter** | The variable name in the definition: `def greet(name)` → `name`. |
| **Argument** | The actual value passed in: `greet("Alice")` → `"Alice"`. |
| **Return value** | What a function hands back (`None` if there's no `return`). |
| **Scope** | Where a variable is visible. Variables made inside a function are *local* to it. |
| **Exception** | A runtime error object (`ValueError`, `KeyError`…). Uncaught → script crashes with a traceback. |
| **Context manager (`with`)** | Block that sets something up and **always cleans up** (e.g. closes the file) even on error. |
| **File mode** | How a file is opened: `"r"` read, `"w"` write (overwrites), `"a"` append. |
| **Library / module** | Pre-written code you `import`. |
| **Standard library** | Modules that ship with Python — no install needed. |
| **PyPI** | Python Package Index — the public store of third-party packages. |
| **pip** | Package manager that installs from PyPI: `pip install requests`. |
| **Tuple** | Fixed group of values; what you get back when a function returns several things. |
| **Docstring** | `"""Description"""` string right under `def`, documenting the function. |

---

## Functions (Task 2)

```python
def greet(name):                       # def + name + (parameters) + colon
    print(f"Hello, {name}. Welcome to the system.")

greet("Alice")     # call it
greet("Bob")       # call it again with a different argument
```

**Return values** — `return` sends a result back and **exits the function immediately**. No `return` → returns `None`.

```python
def check_length(password, min_length):
    if len(password) >= min_length:
        return True
    else:
        return False

result = check_length("Tr0ub4dor", 8)   # True
```

**Multiple parameters** — the scoring function reused in the capstone:

```python
def score_password(password, common_list):
    score = 0
    if len(password) >= 8:   score += 1
    if len(password) >= 12:  score += 1
    if any(c.isdigit() for c in password): score += 1
    if any(c.isupper() for c in password): score += 1
    if password not in common_list:        score += 1
    return score
```

**Default parameter values** — caller may omit them:

```python
def check_length(password, min_length=8):
    return len(password) >= min_length

check_length("short")      # False (default 8)
check_length("short", 4)   # True  (overrides default)
```

**Scope** — variables created inside a function don't exist outside it:

```python
def calculate():
    result = 42        # local
    return result

calculate()
# print(result)        # NameError: result is not defined here
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:10px 14px;border-radius:4px;margin:12px 0;">
<strong>💡 Parameter vs argument</strong> — parameter = the <em>name</em> in the definition; argument = the <em>value</em> you pass. People mix them up in speech, but docs use them precisely. And scope is a <em>feature</em>: functions can't accidentally trample each other's variables.
</div>

> ✏️ **Room exercise:** `def is_long_enough(password): return len(password) >= 12`

---

## Error handling (Task 3)

**The problem:** `int("hello")` raises a `ValueError` and the **whole program stops** — in a tool scanning thousands of hosts, one bad input could halt everything.

```python
try:
    text = input("Enter a number: ")
    number = int(text)
    print(f"You entered {number}")
except ValueError:
    print("That is not a valid number. Please try again.")
```

How it flows: code in `try` runs; **no error** → `except` is skipped; **error** → jump to the matching `except` instead of crashing.

**Common exceptions**

| Exception | When | Example |
| --- | --- | --- |
| `ValueError` | Bad type conversion | `int("abc")` |
| `FileNotFoundError` | File doesn't exist | `open("missing.txt")` |
| `ZeroDivisionError` | Divide by zero | `10 / 0` |
| `KeyError` | Dict key missing | `d["missing_key"]` |
| `IndexError` | List index out of range | `mylist[99]` |

**Handle different errors differently:**

```python
try:
    filename = input("File to open: ")
    with open(filename) as f:
        data = f.read()
    port = int(data.strip())
except FileNotFoundError:
    print(f"Error: '{filename}' does not exist.")
except ValueError:
    print("Error: the file does not contain a valid number.")
```

**Generic catch** — `except Exception as e` stores the message in `e`:

```python
try:
    risky_operation()
except Exception as e:
    print(f"Something went wrong: {e}")
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:10px 14px;border-radius:4px;margin:12px 0;">
<strong>⚠️ Use generic <code>except Exception</code> sparingly</strong> — it swallows <em>every</em> error, including bugs you'd want to see. Prefer catching the specific exception you expect.
</div>

**Pattern: loop until the input is valid** (used heavily in interactive tools):

```python
while True:
    try:
        age = int(input("Enter your age: "))
        break                      # valid -> leave the loop
    except ValueError:
        print("Invalid input. Please enter a whole number.")
```

---

## Reading and writing files (Task 4)

Pentest uses: read **wordlists**, parse **logs**, write **scan reports**.

`open(path, mode)`:

| Mode | Meaning |
| --- | --- |
| `"r"` | Read — file must exist |
| `"w"` | Write — creates file **or erases existing contents** |
| `"a"` | Append — creates file or adds to the end |

**Always use `with`** — it closes the file automatically, even if an error occurs. (Manual `open()`…`close()` leaks the handle if something fails in between.)

```python
with open("passwords.txt", "r") as f:
    content = f.read()
print(content)             # file is already closed here
```

**Reading methods**

| Method | Returns | Best for |
| --- | --- | --- |
| `.read()` | Whole file as **one string** | Small files |
| `.readline()` | **Next single line** | One line at a time |
| `.readlines()` | **List** of lines | When you need all lines as a list |

**Memory-efficient pattern** — loop over the file directly, and `.strip()` each line (lines end with `\n`):

```python
with open("common_passwords.txt", "r") as f:
    for line in f:
        password = line.strip()    # "password123\n" -> "password123"
```

**Load a wordlist into a list**, then use `in`:

```python
common_passwords = []
with open("common_passwords.txt", "r") as f:
    for line in f:
        common_passwords.append(line.strip())
print(f"Loaded {len(common_passwords)} common passwords.")

if user_password in common_passwords:
    print("This password appears in the common-passwords list.")
```

**Writing**

```python
with open("results.txt", "w") as f:        # creates / OVERWRITES
    f.write("Scan Results\n")
    f.write("============\n")
    f.write("Target: 192.168.1.1\n")

with open("results.txt", "a") as f:        # adds to the end
    f.write("Additional finding: port 3306 open\n")
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:10px 14px;border-radius:4px;margin:12px 0;">
<strong>⚠️ <code>"w"</code> destroys existing content immediately</strong> — the moment you open an existing file in write mode it's emptied. To keep what's there and add more, use <code>"a"</code>.
</div>

---

## Libraries and pip (Task 5)

A **library** is a toolbox of pre-written code — grab the wrench instead of forging one.

**Ways to import**

```python
import os                           # whole module  -> os.getcwd()
from datetime import datetime       # one name      -> datetime.now()
import datetime as dt               # with an alias -> dt.datetime.now()
```

**Standard library — no install needed**

| Module | Purpose | Example use |
| --- | --- | --- |
| `os` | Talk to the operating system | List files, env variables |
| `sys` | System-specific info | Command-line arguments |
| `random` | Random numbers | Pick items, shuffle lists |
| `datetime` | Dates and times | Timestamps, time maths |
| `json` | Read/write JSON | API responses, config files |
| `hashlib` | Cryptographic hashes | MD5, SHA-256 |
| `string` | String constants | `string.punctuation`, `string.digits` |

**`string` constants** let you check character variety without typing every symbol:

```python
import string
password = "S3cure!Pass"
has_upper   = any(c in string.ascii_uppercase for c in password)   # True
has_digit   = any(c in string.digits for c in password)            # True
has_special = any(c in string.punctuation for c in password)       # True
```

> 🔍 **Reading `any(...)`** (left unexplained in Core Concepts): `any(c in string.digits for c in password)` asks "is **at least one** character `c` a digit?" and returns `True`/`False`. It's a one-line replacement for a `for` loop with a flag.

**Third-party packages with pip** (from PyPI):

```bash
pip install requests
```

```python
import requests
response = requests.get("https://tryhackme.com")
print(response.status_code)    # 200
```

**Security libraries coming up in the path**

| Library | Use |
| --- | --- |
| `requests` | HTTP requests (scraping, API interaction) |
| `scapy` | Craft, send and sniff network packets |
| `pwntools` | CTF / binary-exploitation toolkit |
| `paramiko` | SSH client and server |
| `beautifulsoup4` | Parse and extract data from HTML |

No need to memorise them now — the point is that "I need to send a packet" becomes a few lines because someone built the hard parts.

> ✏️ **Room exercise:** `hashlib` computes a SHA-256 hash in `imports_demo.py` — change the input string by even one character and the hash changes **completely** (the avalanche effect).

---

## Capstone: Password Strength Checker (Task 6)

**What it does:** load a wordlist → prompt for a password → check length, character variety and "is it common?" → give a score + label (Weak / Moderate / Strong) → show suggestions → log the result. Full script: `password_checker.py`.

### Step 1 — imports and loading the wordlist

```python
import string

def load_common_passwords(filepath):
    """Load a list of common passwords from a text file."""
    common = []
    try:
        with open(filepath, "r") as f:
            for line in f:
                common.append(line.strip().lower())
    except FileNotFoundError:
        print(f"Warning: '{filepath}' not found. Skipping common-password check.")
    return common
```

- File missing → **warning, not a crash**; returns an empty list and the program carries on.
- `.lower()` normalises everything so the comparison is **case-insensitive**.

### Step 2 — the scoring function

```python
def check_password(password, common_list):
    """Evaluate a password and return (score, feedback_list)."""
    score = 0
    feedback = []

    # Length
    if len(password) >= 8:
        score += 1
    else:
        feedback.append("Password should be at least 8 characters.")
    if len(password) >= 12:
        score += 1

    # Character variety
    if any(c in string.ascii_uppercase for c in password):
        score += 1
    else:
        feedback.append("Add at least one uppercase letter.")
    if any(c in string.digits for c in password):
        score += 1
    else:
        feedback.append("Add at least one digit.")
    if any(c in string.punctuation for c in password):
        score += 1
    else:
        feedback.append("Add at least one special character (e.g., !, @, #).")

    # Common password -> overrides everything
    if password.lower() in common_list:
        score = 0
        feedback = ["This password is in the common-passwords list. Choose another."]

    return score, feedback
```

**Scoring rubric (max 5):**

| Check | Points |
| --- | --- |
| Length ≥ 8 | +1 |
| Length ≥ 12 | +1 |
| Has uppercase | +1 |
| Has digit | +1 |
| Has special character | +1 |
| **In common list** | **score forced to 0** (overrides all) |

Returns **two values** (`return score, feedback`) — the caller receives a **tuple** and unpacks it: `score, feedback = check_password(...)`.

### Step 3 — the main program

```python
def main():
    strength_labels = {
        0: "Weak", 1: "Weak",
        2: "Moderate", 3: "Moderate",
        4: "Strong", 5: "Strong"
    }

    common_list = load_common_passwords("common_passwords.txt")

    while True:
        password = input("\nEnter a password to check (or 'quit' to exit): ")

        if password.lower() == "quit":
            print("Goodbye.")
            break

        if len(password) == 0:
            print("Password cannot be empty. Try again.")
            continue

        score, feedback = check_password(password, common_list)
        label = strength_labels.get(score, "Unknown")

        print(f"\nStrength: {label} ({score}/5)")
        if feedback:
            print("Suggestions:")
            for tip in feedback:
                print(f"  - {tip}")

        # Log the result — mask the password
        with open("password_log.txt", "a") as log:
            log.write(f"Password: {'*' * len(password)} | Strength: {label} ({score}/5)\n")

main()
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:10px 14px;border-radius:4px;margin:12px 0;">
<strong>🔒 Never log plaintext passwords.</strong> The script writes <code>'*' * len(password)</code> (asterisks) to the log instead of the real password. Logs get read, copied and leaked; secrets in them are a classic finding.
</div>

**Sample run**

| Input | Result | Why |
| --- | --- | --- |
| `password` | **Weak (0/5)** | In the common list → forced to 0 |
| `Tr0ub4dor` | **Moderate (3/5)** | ≥8 chars + uppercase + digit; missing special char (not ≥12) |
| `C0mpl3x!Pass#99` | **Strong (5/5)** | ≥12, upper, digit, special |
| `quit` | exits | `break` out of the loop |

**Where each earlier concept appears**

| Concept | Where it shows up |
| --- | --- |
| Functions, return, tuple | `load_common_passwords`, `check_password`, `main` |
| Error handling | `try/except FileNotFoundError` around the wordlist |
| File I/O | read wordlist (`"r"`), append log (`"a"`) with `with` |
| Libraries | `import string` |
| Lists + loops | `common`, `feedback`, `for tip in feedback` |
| Dictionaries | `strength_labels` + `.get(score, "Unknown")` |
| Strings + methods | `.strip()`, `.lower()`, `len()`, `'*' * len(...)` |
| `while True` / `break` / `continue` | The main prompt loop |
| `in` operator | `password.lower() in common_list` |

> ✏️ **Ideas to extend it:** minimum number of special characters, detect consecutive repeated characters, reject dictionary words.

---

## Pentest-flavoured cheat sheet

| Goal | Python pattern |
| --- | --- |
| Reusable check | `def is_weak(pw, wordlist): return pw.lower() in wordlist` |
| Don't crash on bad input | `try: ... except ValueError: ...` |
| Re-prompt until valid | `while True:` + `try` + `break` on success |
| Load a wordlist | `with open(f) as fh: words = [l.strip() for l in fh]` *(or the explicit loop above)* |
| Write a report | `with open("report.txt", "w") as f: f.write(...)` |
| Add to an existing report | mode `"a"` (**not** `"w"`) |
| Missing file shouldn't kill the run | `except FileNotFoundError:` |
| Hash a string | `hashlib.sha256(b"text").hexdigest()` |
| Character variety checks | `string.ascii_uppercase / digits / punctuation` |
| HTTP request | `pip install requests` → `requests.get(url)` |

---

## Key takeaways

- **Functions** = `def name(params):` + `return`. Parameters are the names, arguments are the values. Variables inside are **local** (scope). No `return` → `None`.
- **`try`/`except`** stops one bad input from crashing a whole tool. Catch **specific** exceptions; use `except Exception as e` sparingly.
- Always open files with **`with`** — it auto-closes. **`"w"` overwrites, `"a"` appends.** `.strip()` each line to drop the trailing `\n`.
- **Libraries**: standard library needs no install (`os`, `string`, `hashlib`, `json`…); everything else comes from PyPI via **`pip install`**.
- A function returning several values gives you a **tuple** you can unpack: `score, feedback = check_password(...)`.
- Security habit from the capstone: **never write plaintext passwords to logs** — mask them.
- Both rooms together give you every building block for the **Python: Pentesting Scripts** room: web recon, port scanning, hash cracking, SSH brute-forcing.

---

*Python Scripting Basics room 3. Next: Python: Pentesting Scripts. Previous: [Python: Core Concepts](Python-Core-Concepts.md).*
