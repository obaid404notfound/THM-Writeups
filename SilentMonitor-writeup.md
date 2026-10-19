# TryHackMe Writeup – Silent Monitor

Oct 19, 2026 · @OD1Nn00b

## Enumeration

Silent Monitor exposes a small web application described in-room as a "corporate NOC portal" for monitoring internal hosts. Browsing the site turns up an internal operations dashboard guarded by a login page at `/internal`.

The dashboard sits behind authentication, so the first goal was finding a way past the login form or another way in.

## Web Exploitation – Command Injection

Once logged in as an `operator` account, the dashboard's **Operations → Host Health** page stood out: it takes a `target` host and presumably runs a ping-style check against it server-side. Watching the request in the browser's network tab confirmed the backend was shelling out to an actual `ping`/`cat`-style command with the `target` parameter appended more or less directly.

Appending a URL-encoded newline (`%0a`) after the target value broke out of the intended command and let a second command run alongside it:

```
target=127.0.0.1%0acat secret.config
```

The response came back rendered inside the page (visible under the Network tab's Response view), and `secret.config` turned out to hold internal service credentials, including a block for the backup automation:

```
[backup_agent]
run_as   = sysadmin
password = S3cur3Backup$Acc3ss!
```

A comment left in the same file (`# TODO: migrate to secrets manager before Q2 audit`) was a nice confirmation that this config was never meant to be reachable – exactly the kind of corner-cutting the room's briefing hints at.

## Initial Access – SSH

The leaked `backup_agent` credentials (`sysadmin` / `S3cur3Backup$Acc3ss!`) weren't valid on the web portal, but the room also exposes SSH – and they worked there.

```
ssh sysadmin@<target-ip>
```

```
sysadmin@tryhackme-2204:~$ cat user.txt
THM{sQLi_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}
```

Poking around the home directory turned up a `backups/` folder containing a README and a KeePass database:

```
sysadmin@tryhackme-2204:~/backups$ ls
README.txt  infrastructure.kdbx
sysadmin@tryhackme-2204:~/backups$ cat README.txt
Backup archive – infrastructure credentials
```

`infrastructure.kdbx` was clearly the next target: a KeePass vault sitting in a backup directory, almost certainly holding credentials for something higher-privileged than `sysadmin`.

## Privilege Escalation – Cracking the KeePass Vault

`.kdbx` files hold their contents behind a master password, so the database itself first had to be pulled off the box and its hash extracted for offline cracking:

```
scp sysadmin@<target-ip>:~/backups/infrastructure.kdbx .
keepass2john infrastructure.kdbx > infra_hash.txt
hashcat -m 13400 infra_hash.txt rockyou.txt
```

The hash cracked against a standard wordlist, which opened the vault in KeePassXC. Inside, a single entry ("Root User") held the username `root` and its password, copied straight from the entry's context menu:

> Right-click entry → Copy Password

With that password in hand, escalating to `root` over the existing SSH session was a one-liner:

```
sysadmin@tryhackme-2204:~$ su root
Password:
root@tryhackme-2204:/home/sysadmin# cd /root
root@tryhackme-2204:~# cat root.txt
THM{KDBx_V4ul7_H4s_b33n_cr4ck3d_0peN}
```

The vault reuse is the whole point of the room's briefing: credentials "hidden in places no one thinks to look" – in this case a backup archive the sysadmin account could read but presumably wasn't meant to be cracked so easily.

## Conclusion

A clean chain from web to root: SQL/command-injection style thinking on the Host Health feature exposed a config file, the config leaked SSH creds, and a KeePass backup that should never have been crackable handed over root. A good room for practicing the full loop – web exploitation, lateral movement through leaked credentials, and offline cracking – without any single step being too obscure.
