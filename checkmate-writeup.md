# Checkmate — TryHackMe Writeup

**Room:** [tryhackme.com/room/checkmate](https://tryhackme.com/room/checkmate)
**Category:** Password Security / Credential Attacks
**Difficulty:** Easy–Medium

## Scenario

Marco Bianchi, a systems administrator, deployed four internal services under
deadline pressure: a firewall management console, an employee portal, a
social platform, and SSH access to production infrastructure. Instead of
using unique credentials per service, he reused weak, guessable,
pattern-based passwords. The objective is to walk through each layer and
demonstrate exactly how predictable password practices fail in the real
world — this room is effectively a live demo of five classic credential
attack techniques, stacked on top of each other.

Rather than just listing "here's the password," this writeup explains the
*reasoning* behind each technique, so you can apply the same logic to a real
engagement rather than just this room.

---

## Level 1 — Default Credentials (Port 5001)

The first service is a firewall management console. Weak deployments like
this are frequently shipped with vendor default credentials that
administrators forget — or don't bother — to change.

**Approach:** Before brute-forcing anything, always try the obvious defaults
first. `12345`, `admin`, `password`, and `changeme` account for a surprising
share of real-world initial access.

```
Password: 12345
```

**Why this matters:** This isn't really a "hack" — it's a reminder that
default-credential audits should be step one of any password assessment,
before reaching for automated tools.

---

## Level 2 — Custom Wordlist from Site Content (Port 5002)

Level 2 exposes an "Internal Employee Portal" with a login form for user
`marco`. The hint is that the password is built from vocabulary pulled from
the site itself — a common pattern when non-technical staff pick
"memorable" words tied to their workplace.

**Step 1 — Harvest the page content into a wordlist:**

```bash
wget -q -O - http://<TARGET_IP>:5002 | grep -oP '\w{4,}' | sort -u > wordlist.txt
```

This pulls the raw HTML, extracts words of 4+ characters (filtering out
noise like single letters and HTML tags), and de-duplicates the result into
`wordlist.txt`.

> Tip: for a more thorough scrape across multiple pages, `CeWL` does this
> more intelligently than a raw `wget`/`grep` pipe — worth using on a real
> assessment where the target has more than one page:
> `cewl -w wordlist.txt -d 2 http://<TARGET_IP>:5002`

**Step 2 — Brute-force the login form with Hydra:**

```bash
hydra -l marco -P wordlist.txt <TARGET_IP> http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid" -s 5002 -V
```

- `-l marco` — the known username
- `-P wordlist.txt` — our custom wordlist as the password source
- `http-post-form "path:params:failure-string"` — tells Hydra how the login
  form is structured and what failure text to watch for
- `-s 5002` — target port
- `-V` — verbose, so you can watch attempts live

```
Password: excellence
```

**Why this matters:** This demonstrates *content-derived password guessing*
— if a password is inspired by something publicly visible (a company name,
a slogan on the homepage, a product name), scraping that content into a
wordlist beats blindly throwing `rockyou.txt` at it.

---

## Level 3 — OSINT-Based Wordlist with CUPP (Port 5003)

Having authenticated as Marco in Level 2, we now have access to personal
details about him (name, surname, birth year, pet's name, etc.) — the
classic ingredients people use to construct "personal" passwords.

**Tool: CUPP (Common User Passwords Profiler)** — generates a targeted
wordlist from personal information rather than a generic dictionary.

```bash
python3 cupp.py -i
```

Feed in Marco's known details when prompted (first name, surname, birthdate,
nickname, partner's/pet's name, etc.). CUPP outputs a file like `marco.txt`
containing likely password permutations (`Bianchi1990!`, `MarcoB95`, and so
on).

**Brute-force the social platform login:**

```bash
hydra -l marco -P marco.txt <TARGET_IP> http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid" -s 5003 -V
```

```
Password: Bianchi2495
```

**Why this matters:** This is textbook **OSINT-driven password guessing** —
real attackers do exactly this using LinkedIn, social media bios, and data
breaches to build a targeted wordlist instead of a generic one. It's far
more efficient than brute-forcing blind, and it's why "your password
shouldn't be guessable from your own public profile" is a genuinely
important rule, not just a compliance checkbox.

---

## Level 4 — Hash Cracking (SHA-256)

Now logged into the social platform, we find a post from Marco containing a
SHA-256 hash — presumably something he thought was "safe" to share since
it's hashed.

**Step 1 — Save the hash:**

```bash
echo "<hash_value>" > hash.txt
```

**Step 2 — Crack it with Hashcat against rockyou.txt:**

```bash
hashcat -m 1400 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

- `-m 1400` — hash mode for raw SHA-256
- `-a 0` — straight/dictionary attack mode

```
Password: family
```

**Why this matters:** A hash is not "encryption" — it's a one-way
fingerprint, and if the underlying plaintext is weak/common (a dictionary
word, in this case), it cracks almost instantly regardless of the hash
algorithm's strength. This is the core reason password *complexity* matters
even when systems store hashes rather than plaintext.

---

## Level 5 — Pattern-Based Wordlist with Crunch → SSH (Port 22)

The final clue comes from another post on the social platform, revealing
Marco's password *structure*: a capitalized word, followed by a year, ending
in a special character — e.g. `Word####!`. This mirrors a very real
password anti-pattern: "creative" but ultimately predictable
company-policy-compliant passwords (capital letter + numbers + symbol,
built around a guessable base word).

**Step 1 — Generate a pattern-matching wordlist with Crunch:**

```bash
crunch 13 13 -t Security20%%! -o security_wordlist.txt
```

- `13 13` — fixed password length of 13 characters
- `-t Security20%%!` — pattern template, where `%` = a digit placeholder,
  producing `Security20XX!` for every 2-digit combination
- `-o` — output file

**Step 2 — Brute-force SSH with Hydra:**

```bash
hydra -l marco -P security_wordlist.txt <TARGET_IP> ssh -V -t 4
```

- `ssh` — target service
- `-t 4` — 4 parallel connection threads (SSH brute-forcing is slow and
  often rate-limited, so don't crank this too high or you'll get
  locked out / throttled)

```
Password: Security2024!
```

**Why this matters:** This is the room's strongest lesson. `Security2024!`
*technically* satisfies most corporate password policies (uppercase, digits,
special character, 13 characters) — and is still trivially guessable once
an attacker knows the structure. Policy compliance ≠ actual strength.

---

## Summary Table

| Level | Service           | Technique                          | Password        |
|-------|--------------------|-------------------------------------|------------------|
| 1     | Firewall (5001)     | Default credential                  | `12345`          |
| 2     | Employee Portal (5002) | Site-content wordlist + Hydra    | `excellence`     |
| 3     | Social Platform (5003) | OSINT wordlist via CUPP + Hydra  | `Bianchi2495`    |
| 4     | Hash on social post | SHA-256 crack via Hashcat           | `family`         |
| 5     | SSH (22)            | Pattern wordlist via Crunch + Hydra | `Security2024!`  |

## Key Takeaways

- **Never reuse passwords across systems** — each level here fell because
  Marco's password *logic* (not just the password itself) was consistent
  and therefore predictable.
- **Publicly visible content is attacker fuel** — website text, personal
  bios, and even hashed values posted "safely" all leak information that
  narrows down password guessing dramatically.
- **Policy compliance isn't the same as strength.** A password can tick
  every complexity box and still be one Crunch pattern away from cracked.
- **Layered weak security compounds fast.** No single mistake here was
  catastrophic on its own — it was the accumulation across five systems
  that gave full compromise, from firewall console to root SSH access.
