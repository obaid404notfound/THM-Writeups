# Grep — TryHackMe Room Writeup

## Overview

Grep is an OSINT-and-web-exploitation room: the target box hides most of what you need in plain sight — GitHub source code, TLS certificate metadata, and a leaked API key — rather than behind anything you'd brute-force. Getting through it is less about heavy exploitation and more about reading what the application (and its public source) is quietly telling you.

## Initial Recon

Kick things off with an aggressive full-port scan to see what's actually listening:

```bash
sudo nmap -n -A -Pn --min-parallelism 100 -T5 -p- <target-ip>
```

Three ports came back open. Port 80 served nothing but the stock Apache landing page, and directory brute-forcing against it turned up nothing useful — a dead end on its own.

## Following the Certificate

Port 443 threw a certificate warning, which is usually worth a look rather than a click-through. Inspecting the certificate details revealed a hostname reference: `grep.thm`, tied to something called "SearchME."

Since `grep.thm` isn't a public DNS name, it had to be mapped manually:

```
<target-ip>  grep.thm
```

A quick ping confirmed the host resolved correctly, and browsing to `https://grep.thm` brought up the actual web application — a CMS calling itself SearchME.

## Finding the Registration API Key

The site had a registration form, but submitting it failed with an "Invalid or expired API key" error — meaning registration is gated behind a key the front end doesn't hand you.

Since the certificate had already pointed toward "SearchME," the next move was searching GitHub for a public repository matching that name. That search paid off: the project's source was public, and its `register.php` file contained a hardcoded API key that had been left in (or reintroduced via a since-reverted commit).

**API key found:** `ffe60ecaa8bba2f12b43d1a4b15b8f39`

### Answer 1 — API key that allows registration
```
ffe60ecaa8bba2f12b43d1a4b15b8f39
```

## Registering and Grabbing the First Flag

With the key in hand, the registration request was replayed through Burp Suite, swapping in the recovered API key before forwarding it. The server accepted it, registration succeeded, and logging into the new account surfaced the room's first flag.

### Answer 2 — First flag
```
THM{4ec9806d7e1350270dc402ba870ccebb}
```

## Getting Code Execution via Unrestricted Upload

Digging further into the same GitHub repo turned up another file, `upload.php`. Reading through it showed the upload handler doesn't check file extensions directly — instead it validates against a hex byte value tied to each allowed extension (`jpg`, etc.), which is a much weaker check than it looks.

The plan:

1. Grab a PHP reverse shell (the well-known [pentestmonkey php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)) and set it to callback to my IP and listening port.
2. Rename it with a `.jpg.php`-style double extension to line up with what the upload check expects.
3. Open the file in a hex editor and patch the byte sequence at the start to match the hex signature the app associates with an allowed extension.
4. Start a listener, upload the file, then browse to it from the upload directory to trigger execution.

> **Gotcha:** hex-editing the file can shift the leading `<?php` tag if you're not careful — if the shell doesn't fire, check that the PHP opening tag still sits correctly relative to whatever byte you patched in.

This landed a working reverse shell on the box.

## Digging Through the Filesystem

From the shell, a `users.sql` file turned up containing the site's user table — including the admin account's email address.

### Answer 3 — Email of the "admin" user
```
admin@searchme2023cms.grep.thm
```

Poking around further, a `leakchecker` directory contained a second application: a tool for checking whether a given email address has appeared in a known password leak.

### Answer 4 — Hostname of the email/password-leak checker
```
leakchecker.grep.thm
```

As with `grep.thm`, this subdomain needed to be added to the hosts file pointing at the same target IP before it would resolve.

## Checking the Admin's Email for a Leak

With `leakchecker.grep.thm` reachable, submitting the admin's email address (recovered from `users.sql`) returned a leaked password associated with that account.

### Answer 5 — Password of the "admin" user
```
admin_tryhackme!
```

## Summary

| # | Question | Answer |
|---|---|---|
| 1 | API key that allows registration | `ffe60ecaa8bba2f12b43d1a4b15b8f39` |
| 2 | First flag | `THM{4ec9806d7e1350270dc402ba870ccebb}` |
| 3 | Admin user's email | `admin@searchme2023cms.grep.thm` |
| 4 | Email-leak checker hostname | `leakchecker.grep.thm` |
| 5 | Admin user's password | `admin_tryhackme!` |

## Takeaways

- TLS certificates leak more than encryption — hostnames in the SAN/CN fields are free recon.
- Public GitHub repos tied to a custom application are worth searching for by name; leftover secrets in source history are common.
- Extension checks based on partial byte/hex matching are trivial to spoof — always verify uploads against actual file content, not a shallow signature check.
- Leftover database dumps (`.sql` files) sitting in a web root are a frequent source of credential and PII leakage.
