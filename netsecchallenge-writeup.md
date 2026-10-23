---
title: Net Sec Challenge — TryHackMe Writeup
platform: TryHackMe
category: Network Security
tools: nmap, telnet, ssh, hydra
room: https://tryhackme.com/room/netsecchallenge
---

# Net Sec Challenge

A pure network-enumeration challenge — no exploit dev, just `nmap`, `telnet`, `ssh`, and `hydra` used methodically to answer every question.

**Target:** `[MACHINE_IP]`

---

## Step 1 — Full Port Scan

Start with a full TCP port sweep so nothing above the common 1000 ports gets missed — the challenge specifically hides things outside that range.

```bash
nmap -p- -T4 [MACHINE_IP]
```

```
[PASTE FULL PORT SCAN OUTPUT HERE]
```

**Q: How many TCP ports are open?**
A: `[COUNT]`

**Q: What is the highest port number open that is below 10,000?**
A: `[PORT]`

**Q: There's an open port above 10,000 — what is it?**
A: `[PORT]`

---

## Step 2 — Service/Version Scan

Now run a version scan against the discovered ports to pull banners.

```bash
nmap -sV -sC -p [PORT1],[PORT2],[PORT3],... [MACHINE_IP]
```

```
[PASTE SERVICE SCAN OUTPUT HERE]
```

---

## Step 3 — HTTP Server Header Flag

The HTTP service's server header has a flag baked into it. Grab it with curl headers or just check what nmap/the browser shows.

```bash
curl -I http://[MACHINE_IP]:[PORT]
```

```
[PASTE HEADER OUTPUT — Server: ... THM{...}]
```

**Q: What is the flag hidden in the HTTP server header?**
A: `[FLAG]`

---

## Step 4 — SSH Server Header Flag

No creds needed here — just connect verbosely and read the banner exchange before auth fails.

```bash
ssh -v [MACHINE_IP]
```

```
[PASTE RELEVANT DEBUG LINE — e.g. debug1: match: OpenSSH_X.Yp1 'THM{flag}' pat OpenSSH* ...]
```

**Q: What is the flag hidden in the SSH server header?**
A: `[FLAG]`

---

## Step 5 — FTP Version on a Non-Standard Port

FTP isn't on 21 here — it showed up in the full port scan. Connect or let nmap grab the banner.

```bash
nmap -sV -p [FTP_PORT] [MACHINE_IP]
# or
telnet [MACHINE_IP] [FTP_PORT]
```

```
[PASTE FTP BANNER OUTPUT]
```

**Q: What is the version of the FTP server?**
A: `[FTP_VERSION]`

---

## Step 6 — Hydra Brute Force Against FTP

Two candidate usernames were given via the room's social-engineering lead-in: `eddie` and `quinn`. Brute-force both against the FTP port using rockyou.

```bash
echo -e "eddie\nquinn" > usr
hydra -L usr -P /usr/share/wordlists/rockyou.txt ftp://[MACHINE_IP]:[FTP_PORT]
```

```
[PASTE HYDRA OUTPUT — valid pair found]
```

Log in with the cracked credentials and look for the flag file in the account's FTP directory.

```bash
ftp [MACHINE_IP] [FTP_PORT]
# login with cracked username/password
ls
get [FLAG_FILE]
cat [FLAG_FILE]
```

**Q: What is the flag hidden in one of the two account files, accessible via FTP?**
A: `[FLAG]`

---

## Step 7 — Web Challenge at Port 80

Browsing to the target's root HTTP port presents a small puzzle/challenge that yields a flag once solved.

```bash
curl http://[MACHINE_IP]/
```

[Describe what the page/challenge actually asks for, and how you solved it — e.g. a covert scan technique like `nmap -sN` (TCP NULL scan) that the page is checking for.]

```bash
nmap -sN [MACHINE_IP]
```

```
[PASTE RESULT / FLAG]
```

**Q: Browsing to http://MACHINE_IP displays a small challenge — what is the flag?**
A: `[FLAG]`

---

## Key Takeaways

1. **Always scan all 65535 ports, not just the top 1000.** Default `nmap` scans miss anything above the common port list — this room is built specifically to punish that shortcut.
2. **Banners leak more than service versions.** Both the HTTP and SSH banners here had flags stuffed directly into the version string — a reminder that verbose service banners are themselves a minor information-disclosure risk in the real world.
3. **Non-standard ports don't add real security.** Moving FTP off port 21 didn't stop it from being found or fingerprinted — obscurity isn't a control.
4. **Credential stuffing works because reused/weak passwords are common.** A short username list plus rockyou cracked the FTP login in a reasonable time, which is exactly the threat model `hydra` is built to simulate.
5. **Not every open port needs active enumeration from the front door.** Sometimes the "exploit" is just reading what a service hands you for free — no auth, no payload, just paying attention.

---

*Tools used: nmap, ssh, ftp, hydra · Flags: 8/8*
