# 🔐 Simple CTF — TryHackMe Walkthrough

> **Platform:** TryHackMe  
> **Difficulty:** Beginner  
> **Category:** CTF / Penetration Testing  
> **Objective:** Initial Access → User Flag → Privilege Escalation → Root

---

## ⚠️ Disclaimer

This walkthrough is intended for **authorized security labs and educational environments**, such as TryHackMe.

Do not use these techniques against systems that you do not own or have explicit permission to test.

---

# 📌 Overview

**Simple CTF** is a beginner-friendly TryHackMe machine that demonstrates a complete penetration-testing workflow.

The attack involves:

- Network reconnaissance
- Service enumeration
- Web directory enumeration
- CMS identification
- Vulnerability research
- SQL injection
- Credential discovery
- SSH access
- User flag retrieval
- `sudo` enumeration
- Privilege escalation
- Root access

The overall attack chain is:

```text
Nmap
   ↓
Service Enumeration
   ↓
Web Enumeration
   ↓
CMS Made Simple Discovery
   ↓
Version Identification
   ↓
Vulnerability Research
   ↓
SQL Injection
   ↓
Credential Discovery
   ↓
SSH Access
   ↓
User Flag
   ↓
sudo -l
   ↓
Vim Misconfiguration
   ↓
Privilege Escalation
   ↓
Root
```

---

# 1️⃣ Reconnaissance

The first step in a penetration test is understanding the target.

An Nmap scan can be used to identify open ports and running services.

```bash
sudo nmap -sV -A -O <TARGET_IP>
```

### Important findings

| Port | Service | Details |
|------|---------|---------|
| `21` | FTP | vsFTPd 3.0.3 |
| `80` | HTTP | Apache 2.4.18 |
| `2222` | SSH | OpenSSH 7.2p2 |

The scan also indicates that **anonymous FTP access is available**.

The presence of HTTP and SSH makes the web application and remote-login services particularly interesting.

---

# 2️⃣ Web Enumeration

The HTTP service should be investigated next.

Directory enumeration can help discover hidden paths and applications.

For example:

```bash
gobuster dir -u http://<TARGET_IP>/ -w <WORDLIST>
```

The enumeration reveals an additional web directory.

Opening the discovered directory reveals that the target is running:

> **CMS Made Simple**

The application version is identified as:

> **CMS Made Simple 2.2.8**

Identifying the exact version is important because it allows us to research vulnerabilities associated with that specific release.

---

# 3️⃣ Vulnerability Research

After identifying the CMS and version, vulnerability research can be performed.

`searchsploit` can be used to search for known vulnerabilities:

```bash
searchsploit "CMS Made Simple 2.2.8"
```

The results reveal a known **SQL injection vulnerability** affecting the identified version.

At this point, the investigation has moved from general enumeration to a specific potential attack vector.

---

# 4️⃣ SQL Injection

The identified vulnerability can be researched to understand how it can be exploited.

In the authorized TryHackMe environment, a suitable proof-of-concept can be used to demonstrate the vulnerability.

The important objective is to determine whether the vulnerable application exposes useful information from its database.

The SQL injection ultimately provides access to credential information.

This is a major turning point in the attack because the recovered credentials can potentially be reused against another service.

---

# 5️⃣ SSH Access

Earlier reconnaissance showed that SSH is running on **port 2222**, rather than the default port 22.

The recovered credentials can therefore be tested against SSH:

```bash
ssh -p 2222 <USERNAME>@<TARGET_IP>
```

If the credentials are valid, an interactive shell is obtained.

The attack path is now:

```text
Web Application
      ↓
SQL Injection
      ↓
Credential Discovery
      ↓
SSH
      ↓
User Shell
```

---

# 6️⃣ User Flag

After obtaining the user shell, inspect the current directory:

```bash
ls
```

The user flag can then be read:

```bash
cat user.txt
```

🎯 **User-level access achieved.**

---

# 7️⃣ Privilege Enumeration

Obtaining a normal user shell does not necessarily mean the machine is fully compromised.

The next step is to determine whether the current user has any elevated privileges.

Run:

```bash
sudo -l
```

The output reveals that the user can execute **Vim with root privileges**.

This is an important privilege-escalation opportunity.

---

# 8️⃣ Privilege Escalation

Because Vim is allowed through `sudo`, it can be launched with elevated privileges.

The elevated Vim process can then be leveraged to obtain a root-level shell within the authorized CTF environment.

After performing the escalation, verify the current user:

```bash
whoami
```

If the escalation was successful, the result is:

```text
root
```

🎯 **Root access achieved.**

---

# 🏁 Complete Attack Chain

The complete compromise can be represented as:

```text
┌──────────────────────┐
│      Nmap Scan       │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Open Ports Discovered│
│ 21 / 80 / 2222       │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│  Web Enumeration     │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ CMS Made Simple      │
│      2.2.8           │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Vulnerability        │
│ Research             │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│   SQL Injection      │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ Credential Discovery │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│ SSH - Port 2222      │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│    User Access       │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│      user.txt        │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│      sudo -l         │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│  Vim as Root         │
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│    Root Access       │
└──────────────────────┘
```

---

# 🧠 Key Lessons

### 1. Enumeration comes first

Before attempting exploitation, identify the services and technologies exposed by the target.

### 2. Version information matters

Knowing that the target uses **CMS Made Simple 2.2.8** makes vulnerability research much more focused.

### 3. Follow the information

Credentials obtained through one service may provide access to another service.

### 4. Check unusual ports

SSH was running on port `2222`, demonstrating why port scanning is important.

### 5. Always run `sudo -l`

After obtaining a shell, check what the current user can execute with elevated privileges.

### 6. Misconfigurations can be just as important as vulnerabilities

The final privilege escalation relies on an incorrectly configured `sudo` permission for Vim.

### 7. Think about the entire attack chain

The machine is solved by connecting multiple smaller discoveries:

```text
Reconnaissance
      +
Enumeration
      +
Vulnerability Research
      +
Credential Discovery
      +
Initial Access
      +
Privilege Enumeration
      +
Privilege Escalation
      =
Complete Compromise
```

---

# 🛠️ Tools Used

- **Nmap** — Network and service enumeration
- **Gobuster** — Web directory enumeration
- **Searchsploit** — Vulnerability research
- **SSH** — Remote access
- **sudo** — Privilege enumeration
- **Vim** — Relevant to the privilege-escalation misconfiguration

---

# 📚 Skills Practiced

This room provides practical experience with:

- Reconnaissance
- Port scanning
- Service enumeration
- Web enumeration
- CMS fingerprinting
- Vulnerability identification
- SQL injection concepts
- Credential discovery
- SSH authentication
- Linux enumeration
- Sudo misconfiguration
- Privilege escalation
- Post-exploitation enumeration

---

# 🎯 Conclusion

The **Simple CTF** machine demonstrates how a penetration test can progress from basic reconnaissance to complete system compromise.

The most important lesson is not a single command or exploit. It is learning how to connect findings together.

A seemingly simple discovery such as an open port can lead to service enumeration, which can lead to application identification, vulnerability research, credential discovery, initial access, and eventually privilege escalation.

```text
Enumerate → Identify → Research → Exploit → Access → Escalate
```

---

## 🔗 Reference

Original walkthrough used for comparison:

[Simple CTF — TryHackMe Walkthrough](https://medium.com/@akerkar34/simple-ctf-tryhackme-walkthrough-13fae8abe32d)

---

⭐ **If this walkthrough helped you, consider giving the repository a star!**