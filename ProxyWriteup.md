# TryHackMe: Proxy — Write-up

**Author:** OD1Nn00b  
**Platform:** TryHackMe  
**Room:** [Proxy](https://tryhackme.com/room/proxychallenge)  
**Focus:** Active Directory enumeration, SMB, NTLMv2, BloodHound, Kerberos delegation

> **Spoiler warning:** This document describes the solution path for the room. Run these commands only inside the TryHackMe lab or another environment where you have explicit permission. IP addresses, credentials, hashes, ticket filenames, and flags can differ between deployments; replace placeholders with values from your own session.

## 1. Overview

The Proxy room is an Active Directory lab built around a service account that processes files in a shared folder. The objective is to investigate the domain, identify a weakness in the service account's workflow, and follow the resulting access path through the domain.

The broad attack path is:

1. Enumerate the Domain Controller and its exposed services.
2. Inspect an accessible SMB share and read the files it contains.
3. Identify how the `svc.scanner` service account processes new files.
4. Observe an NTLM authentication attempt from the service account and audit the captured hash offline.
5. Use BloodHound to investigate the account's delegation permissions.
6. Validate the delegation path in the lab and access the target as the Administrator.
7. Locate the room's flag and review the security lessons.

## 2. Lab setup and reconnaissance

Start the room and connect through the TryHackMe VPN or AttackBox. Record the target IP shown in your own room session.

Set a shell variable to make later commands easier to read:

```bash
export TARGET=<TARGET_IP>
```

Scan the target for open TCP ports and identify the services:

```bash
rustscan -b 500 -a "$TARGET" --top -- -sC -sV -Pn
```

**What this does:**

- `rustscan` quickly discovers open ports.
- `-sC` runs Nmap's default scripts against discovered ports.
- `-sV` attempts to identify service versions.
- `-Pn` skips host discovery and treats the target as online.

The results in this room indicate a Windows Domain Controller. Services commonly associated with Active Directory include DNS, Kerberos, LDAP/LDAPS, SMB, RPC, and Remote Desktop. Use the actual scan output from your deployment as the source of truth.

## 3. Enumerate SMB shares

Try guest access to list the target's SMB shares:

```bash
nxc smb "$TARGET" -u 'guest' -p '' --shares
```

Look for a share named `IT-Shared` and note its listed permissions. The room's intended path allows the share to be inspected without an ordinary domain user's credentials.

Enumerate domain users through SMB RID enumeration:

```bash
nxc smb "$TARGET" -u 'guest' -p '' --rid
```

This may reveal account names useful for later investigation, including `svc.scanner`. RID enumeration is a way to discover domain security principals; it does not by itself grant access to those accounts.

If name resolution does not work in your lab, add the Domain Controller's IP and hostname to `/etc/hosts` using the hostname shown by your scan. For this room, the domain and host are represented as `ctf.local` and `DC01.ctf.local`:

```text
<TARGET_IP>  DC01.ctf.local ctf.local DC01
```

## 4. Inspect the shared files

Connect to the SMB share using Impacket's SMB client:

```bash
smbclient.py guest:''@DC01.ctf.local
```

At the SMB prompt, select the share and download the available files:

```text
use IT-Shared
mget *
```

Review the downloaded files locally. In this room, the onboarding checklist explains that `svc.scanner` checks the share periodically and processes newly added files. It also mentions metadata and icons.

That detail is important: a service that automatically processes attacker-controlled files may contact an external resource while handling the file. Treat this as a clue to investigate, not as proof of a vulnerability until you observe the behavior.

The credential-backup file also contains credential-related information. Validate any discovered credentials carefully; the presence of a credential in a file does not guarantee it works or that it has useful privileges.

## 5. Investigate the service account's file processing

The room demonstrates an NTLM authentication exposure caused by the way the service account handles a crafted file. In your own lab, use the file-processing clues from the share to understand what triggers the outbound authentication attempt.

Start an SMB listener on your lab machine and monitor it for an authentication attempt from the target. Make sure you use your own VPN-side IP address and the correct interface for the TryHackMe network.

When the service processes the file, the lab may expose an NTLMv2 challenge-response for `svc.scanner`. Save the captured lab output as evidence, but do not publish real passwords, reusable hashes, or private credentials in a public repository.

### Audit the captured hash offline

If the lab provides a captured NTLMv2 response for this exercise, Hashcat can test it against a local wordlist:

```bash
hashcat -a 0 -m 5600 <HASH_FILE> <WORDLIST>
```

To display a result Hashcat has already recovered:

```bash
hashcat -m 5600 <HASH_FILE> <WORDLIST> --show
```

- `-a 0` selects a straight wordlist attack.
- `-m 5600` selects the NetNTLMv2 hash mode.
- `<HASH_FILE>` is the file containing the lab's captured response.
- `<WORDLIST>` is the local wordlist you choose to test.

Only use this process for the authorized lab exercise. If a password is recovered, keep it out of screenshots and public write-ups; describe the validation result without disclosing the secret.

## 6. Enumerate Active Directory with BloodHound

After validating the service account in the lab, collect Active Directory relationship data using the BloodHound collector available in your environment. The precise collector command and DNS options depend on the installed version and your lab network configuration.

Review the resulting graph for the `svc.scanner` account and its relationships to the Domain Controller. The key finding in this room is a constrained-delegation configuration involving the CIFS service on `DC01.ctf.local`.

### Why delegation matters

Kerberos delegation allows a service to access another service on behalf of a user in specific circumstances. Constrained delegation limits which services can be delegated to, but a risky configuration can still create a path to elevated access—especially when protocol transition is permitted and the account's credentials are compromised.

Record the relevant BloodHound relationship and the exact service principal shown in your own results. Do not assume every account with delegation configured is automatically a Domain Admin; the allowed service and the surrounding permissions determine the impact.

## 7. Validate the delegation path in the lab

The room's intended route uses the service account's constrained-delegation permissions to request a Kerberos service ticket for the CIFS service while impersonating the Administrator account.

In your own TryHackMe session, follow the room's delegation exercise and use the exact domain controller name, service principal, account credentials, and ticket filename returned by your run. Avoid copying a ticket filename or password from another person's write-up because these are session-specific.

Once the lab issues the ticket, configure the Kerberos credential-cache environment variable to point to that actual cache file. Tools that support Kerberos can then use the ticket rather than a password. Confirm the identity and target carefully before proceeding.

## 8. Access the Domain Controller and retrieve the flag

The room demonstrates using the obtained Kerberos ticket to authenticate to the Domain Controller over SMB. Follow the lab's final access task and inspect the Administrator's Desktop for the flag.

Record the flag in your private notes if required for the room, but **do not put the actual flag in this public README**. You can instead write:

```text
Flag: Obtained in the TryHackMe Proxy lab.
```

The reference walkthrough also demonstrates directory credential-dumping after gaining high privileges. For a portfolio write-up, it is usually enough to explain the impact and the defensive significance rather than publishing additional credential material.

## 9. Findings and lessons learned

### Main findings

- Anonymous or guest-accessible SMB shares can disclose sensitive operational details.
- A service that processes untrusted files can unintentionally trigger outbound authentication.
- NTLM challenge-responses can be exposed to offline password auditing if a service authenticates to an attacker-controlled endpoint.
- BloodHound helps reveal privilege relationships that are difficult to spot by inspecting accounts individually.
- Kerberos constrained delegation must be configured carefully, especially when protocol transition is enabled.

### Defensive recommendations

- Remove anonymous or guest access from SMB shares unless it is explicitly required.
- Apply least privilege to shared folders and service accounts.
- Avoid automatically processing files from writable shares, or safely isolate and validate those files.
- Restrict outbound SMB connections and monitor unexpected NTLM authentication.
- Prefer stronger authentication configurations and reduce NTLM usage where feasible.
- Audit Kerberos delegation settings regularly and limit delegation to the minimum required services.
- Use managed service accounts where appropriate, rotate exposed credentials, and monitor suspicious ticket activity.
- Alert on unusual access to domain-controller shares and other privileged resources.

## 10. Conclusion

The Proxy room shows how several small weaknesses can combine into a serious Active Directory compromise. Share enumeration reveals how a service works; the service's file-processing behavior exposes authentication material; and delegation analysis explains how the compromised service account can reach a privileged resource.

The most useful takeaway is to connect individual findings into a complete attack path—and to understand how each weakness can be prevented or detected.

---

**Author:** OD1Nn00b  
**Lab:** TryHackMe — Proxy  
**Note:** This write-up is based on my learning notes. Replace placeholders with evidence from your own run, and add your own screenshots with sensitive information redacted.
