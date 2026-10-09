# TryHackMe: Forward — Write-up

**Author:** OD1Nn00b  
**Platform:** TryHackMe  
**Room:** [Forward](https://tryhackme.com/room/forwardchallenge)  
**Domain:** `ctf.local`  
**Target host:** `DC01.ctf.local`  
**Topics:** Active Directory enumeration, KeePass, credential reuse, BloodHound, Resource-Based Constrained Delegation (RBCD), Kerberos

> **Spoiler warning:** This walkthrough describes the main solution path. Run the commands only in your own TryHackMe lab or another environment where you have explicit authorization. The target IP, passwords, generated computer-account password, ticket-cache filename, and flag may vary. Use your own session's values and do not publish real secrets.

## 1. Room overview

The Forward room begins with assumed initial access to a low-privileged domain account, `j.smith`. The objective is to investigate the Active Directory environment, identify possible lateral-movement paths, and understand how a delegation misconfiguration can lead to privileged access.

The overall path is:

1. Scan the target and identify Active Directory services.
2. Enumerate SMB shares and domain relationships.
3. Connect over RDP and inspect the user's files.
4. Open the KeePass database available in the lab and validate a second account.
5. Investigate domain users and identify credential reuse in the lab.
6. Use BloodHound to confirm an RBCD relationship for `r.williams`.
7. In the authorized lab, follow the RBCD path and validate the impact.
8. Retrieve the room's flag and document the defensive lessons.

## 2. Prepare the lab

Start the room and connect to TryHackMe using the VPN or AttackBox. Copy the target IP address from your active room session.

Set a variable so you can reuse the address in commands:

```bash
export TARGET=<TARGET_IP>
```

The walkthrough uses these lab host details:

- Domain: `ctf.local`
- Domain Controller: `DC01.ctf.local`
- Initial user: `j.smith`

The IP address is intentionally a placeholder because TryHackMe assigns addresses dynamically.

## 3. Scan the target

Run a service scan against the target:

```bash
rustscan -b 500 -a "$TARGET" --top -- -sC -sV -Pn
```

**Why this matters**

- `rustscan` quickly checks for open ports.
- `-sC` runs Nmap's default scripts.
- `-sV` attempts to identify service versions.
- `-Pn` skips host discovery and scans the target directly.

The scan identifies a Windows host with services associated with Active Directory, including DNS, Kerberos, LDAP, SMB, RPC, and RDP. The combination of these services strongly suggests that the target is a Domain Controller.

Use your scan output—not another person's screenshot—as your evidence.

## 4. Enumerate SMB shares

Use the initial lab credentials to list SMB shares:

```bash
nxc smb "$TARGET" -u 'j.smith' -p '<INITIAL_PASSWORD>' --shares
```

Replace `<INITIAL_PASSWORD>` with the credential supplied by the room. Avoid saving passwords directly in shell history when working with real systems.

The enumeration identifies a readable share named `Downloads`. Inspect it using SMBClient:

```bash
smbclient "//$TARGET/Downloads" -U 'ctf.local/j.smith'
```

Enter the password when prompted. At the SMB prompt, list the directory contents:

```text
ls
```

The share does not provide the key information needed for the next step, so continue with domain enumeration rather than assuming an empty share means the account has no useful access.

### Add the Domain Controller to `/etc/hosts`

Some Active Directory tools work more reliably when the host's domain name resolves correctly. Add an entry using the IP address from your own session:

```text
<TARGET_IP>  DC01.ctf.local ctf.local DC01
```

Edit `/etc/hosts` with administrator privileges, and replace `<TARGET_IP>` with the real lab address.

## 5. Enumerate Active Directory with BloodHound

Collect directory relationship data using the credentials supplied by the lab:

```bash
bloodhound-ce.py \
  -u 'j.smith' \
  -p '<INITIAL_PASSWORD>' \
  -d ctf.local \
  -dc DC01.ctf.local \
  -ns "$TARGET" \
  -c All \
  --zip
```

The exact collector command can differ depending on the BloodHound version installed. If the command is unavailable, check which BloodHound collector your lab environment provides.

Import the generated archive into BloodHound and inspect the user and computer objects.

Important observations from the room include:

- `j.smith` belongs to **Remote Desktop Users**, explaining why RDP access is available.
- `svc.helpdesk` appears as a Kerberoastable account, but that finding alone does not prove it is the best route forward.
- `Administrator` is a Domain Admin.

BloodHound is useful because it maps permissions and relationships between objects, making it easier to spot paths that are not obvious from a list of accounts.

## 6. Access the host over RDP

Because the initial account has RDP access, connect to the Domain Controller:

```bash
xfreerdp /v:"$TARGET" /u:'ctf.local\j.smith' /dynamic-resolution
```

Enter the lab password when prompted. Depending on your FreeRDP version, the executable may be named `xfreerdp3` instead of `xfreerdp`.

Once connected, inspect the user's Documents folder. The room contains a KeePass database named:

```text
Database.kdbx
```

A KeePass file is a password database. Finding one does not automatically mean it can be opened, so the next step is to investigate how the lab's database is protected.

## 7. Inspect the KeePass database

The reference solution could not recover the database's master password through ordinary cracking. It instead identified that the database could be opened using the current Windows user-account context, and then opened it with KeePass2 installed on the target.

In your own lab session:

1. Locate `Database.kdbx` in the user's Documents folder.
2. Check whether KeePass2 is installed.
3. Open the database and examine its entries using the available lab access.
4. Identify the entry for `t.jones`.
5. Validate the recovered account in the lab without exposing its password in your public notes.

The key lesson is that password databases and saved credentials need appropriate access controls. A database may be protected by more than a master password, and the security of its key components depends on how the database was configured.

## 8. Validate the `t.jones` account

After discovering the account in the KeePass database, test the credentials against SMB:

```bash
nxc smb DC01.ctf.local -u 't.jones' -p '<T_JONES_PASSWORD>' --shares
```

Replace the placeholder with the credential discovered in your lab. The account authenticates, but it does not immediately provide the privileges needed to complete the room.

This is a useful reminder: successful authentication is not the same thing as administrative access. Continue enumerating permissions and relationships.

## 9. Enumerate users and investigate credential reuse

Use the authenticated account to enumerate domain users:

```bash
nxc smb DC01.ctf.local -u 't.jones' -p '<T_JONES_PASSWORD>' --users
```

Review the resulting usernames and save the relevant names to a local text file, one per line, if needed for the authorized lab.

The reference path tests whether the recovered password has been reused by another domain user. In your own lab, follow the room's password-reuse exercise and respect account lockout policies; do not perform password spraying against systems you do not own or have permission to test.

The room's path identifies `r.williams` as an account whose permissions are more significant than those of `t.jones`.

## 10. Confirm the RBCD relationship in BloodHound

Return to BloodHound and inspect the `r.williams` account's relationships with the Domain Controller.

The important finding is an **AddAllowedToAct** permission over the DC. This can allow a principal to configure Resource-Based Constrained Delegation (RBCD) on the target computer object, provided the necessary conditions and permissions are met.

### What is RBCD?

Resource-Based Constrained Delegation lets a target service specify which accounts may act on behalf of users when accessing that service. In a vulnerable configuration, an attacker who controls a principal that is trusted for delegation may be able to request a service ticket while impersonating another user.

The critical issue in this room is not simply that Kerberos is enabled. It is the combination of account control, the target computer object's delegation attribute, and the service that can be accessed through delegation.

## 11. Understand the lab's delegation path

The room demonstrates the following sequence in the authorized environment:

1. A controlled computer account is created in the domain.
2. The delegation configuration on the Domain Controller is modified to trust that account.
3. Kerberos S4U mechanisms are used to request a service ticket for the CIFS service while representing the Administrator identity.
4. The resulting ticket is used to validate the impact on the Domain Controller.

The relevant concepts are:

- **RBCD:** The target computer controls which principals can delegate to it.
- **S4U2Self:** A service can request a ticket representing a user under particular Kerberos conditions.
- **S4U2Proxy:** A service can request a ticket to another service on behalf of a user when delegation rules permit it.
- **CIFS SPN:** Identifies the file-sharing service accessed over SMB.
- **Kerberos ticket cache:** Stores tickets that compatible tools can use for authentication.

Use BloodHound's relationship details and the official room instructions to confirm the steps for your current lab instance. The required values and the generated ticket filename are session-specific, so they should not be copied blindly from another person's run.

## 12. Validate access using the lab's Kerberos ticket

After the lab's delegation exercise produces a ticket, configure the Kerberos cache environment variable to reference the exact cache file generated by your run:

```bash
export KRB5CCNAME=<PATH_TO_GENERATED_CCACHE>
```

Check the filename before setting the variable. Do not assume that the cache name in a reference screenshot will match your environment.

The reference solution uses Impacket's SMB execution utility with Kerberos authentication to demonstrate the impact. In your own lab, follow the room's provided instructions and confirm that the ticket is being used as expected.

The exercise demonstrates how a delegation misconfiguration can turn control of a less-privileged account into access to a highly privileged service.

## 13. Retrieve the flag

After completing the lab's delegation path, inspect the Administrator's Desktop for the room flag.

Do not include the actual flag, passwords, ticket contents, or dumped domain credentials in a public repository. You can document completion without exposing the solution secret:

```text
Flag: Retrieved successfully in my TryHackMe Forward session.
```

The reference solution also demonstrates the risk of directory credential extraction after obtaining privileged access. For a portfolio write-up, explain the security impact without publishing credential dumps or sensitive authentication material.

## 14. Findings and defensive recommendations

### Findings

- An initial low-privileged account had interactive access to the Domain Controller.
- A KeePass database in the user's profile exposed another account in the lab's workflow.
- The recovered account's password was useful for discovering credential reuse.
- BloodHound revealed an `AddAllowedToAct` relationship affecting the Domain Controller.
- The delegation configuration enabled a path to privileged service access.

### How defenders can reduce this risk

- Avoid granting ordinary users interactive logon access to Domain Controllers.
- Protect password databases and Windows user profiles with appropriate access controls.
- Use unique passwords and managed service accounts where appropriate; monitor for credential reuse.
- Audit computer-object permissions, especially who can modify delegation-related attributes.
- Review RBCD and other Kerberos delegation settings regularly.
- Apply least privilege and alert on unexpected changes to `msDS-AllowedToActOnBehalfOfOtherIdentity`.
- Monitor suspicious service-ticket requests and unusual SMB access to Domain Controllers.
- Use tiered administration and separate privileged accounts from everyday user accounts.

## 15. Conclusion

Forward demonstrates why post-compromise Active Directory work is about understanding relationships—not just finding open ports. The investigation starts with a low-privileged account, moves through local files and account validation, and then uses BloodHound to uncover a dangerous delegation relationship.

The main lesson is to combine careful enumeration with an understanding of Windows authentication and permissions. For defenders, auditing delegation rights, protecting credentials, and limiting interactive access to Domain Controllers can prevent similar paths to privilege escalation.

---

**Author:** OD1Nn00b  
**Room:** [TryHackMe — Forward](https://tryhackme.com/room/forwardchallenge)  
**Disclaimer:** This document is for learning in an authorized lab. Add screenshots from your own session, redact secrets, and verify every observation against your own results.
