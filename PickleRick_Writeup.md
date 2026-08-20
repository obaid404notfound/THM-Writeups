# Pickle Rick — TryHackMe Write-up

## Introduction

**Pickle Rick** is a beginner-friendly TryHackMe room based on the *Rick and Morty* universe. The objective is to investigate a vulnerable web server, gain access to the system, and locate **three ingredients** needed to help Rick return to his normal human form.

This room is useful for practicing:

- Web enumeration
- Source-code analysis
- `robots.txt` inspection
- Directory enumeration
- Web authentication
- Command execution
- Reverse shells
- Linux file permissions
- Basic privilege escalation

> **Note:** This walkthrough is intended for the authorized TryHackMe lab environment.

---

## 1. Initial Web Enumeration

After starting the machine, I first accessed the target web server through its IP address.

The homepage contained information related to Rick, which suggested that the website itself would be important for the investigation.

### Checking the Page Source

I inspected the HTML source of the homepage instead of relying only on what was displayed in the browser.

While reviewing the source, I discovered a username:

```text
R1ckRul3s
```

This looked like an important piece of information that could potentially be used later for authentication.

---

## 2. Investigating `robots.txt`

Next, I checked:

```text
/robots.txt
```

The file contained an unusual entry that provided another useful piece of information.

At this point, I had gathered:

- A possible username from the page source
- A useful clue from `robots.txt`

These pieces of information could be combined with further enumeration.

---

## 3. Directory Enumeration

To discover additional files and directories on the web server, I used **Gobuster**.

The purpose of this step was to identify resources that were not linked directly from the main webpage.

The enumeration revealed several interesting endpoints, including:

```text
login.php
```

The presence of a login page suggested that the information collected earlier might be useful.

---

## 4. Accessing the Login Portal

I opened the discovered login page.

The page requested authentication details. I used the username discovered earlier together with the password-related clue obtained during enumeration.

The credentials were accepted, giving access to the web interface.

This was an important turning point because the authenticated interface exposed functionality that was not available from the public homepage.

---

## 5. Investigating the Command Interface

After logging in, I examined the available functionality.

The interface allowed commands to be executed on the target system. I first tested basic commands to understand the environment and determine which commands were permitted.

One interesting file was:

```text
Sup3rS3cretPickl3Ingred.txt
```

This appeared to be directly related to the first ingredient.

However, normal attempts to read the file were restricted because some commands were disabled by the web interface.

Instead of repeatedly trying blocked commands, I looked for another way to interact with the underlying machine.

---

## 6. Obtaining a Shell

Since the command interface was restrictive, I investigated whether a scripting language was available on the machine.

Python 3 was available.

In the authorized lab environment, I used a Python-based reverse-shell technique and established a listener on my own machine.

Once the connection succeeded, I received an interactive shell running as:

```text
www-data
```

This gave me much more flexibility than the restricted web command interface.

---

## 7. Finding the First Ingredient

With shell access established, I navigated through the relevant directories and located the file containing the first ingredient.

The first ingredient was successfully recovered.

I also checked the nearby `clue.txt` file because it provided information pointing toward the remaining objectives.

The important lesson here is that once limited command execution is available, gaining a proper shell can make system enumeration significantly easier.

---

# 8. Finding the Second Ingredient

The clue indicated that another ingredient could be found under Rick's home directory.

I navigated to:

```text
/home/rick
```

After examining the files in this directory, I located the second ingredient.

At this stage, two of the three required ingredients had been recovered.

---

# 9. Investigating the Root Directory

The final ingredient appeared to be located somewhere under:

```text
/root
```

However, my current shell was running as:

```text
www-data
```

and this account did not have permission to access the root user's directory.

This meant that I needed to perform a basic privilege-escalation check.

---

# 10. Checking Sudo Permissions

I checked which commands the current user could execute with elevated privileges.

The result was particularly interesting: the account had extremely broad `sudo` permissions and could execute commands with elevated privileges without requiring a password.

This represented the privilege-escalation path for the room.

---

# 11. Obtaining Root Access

Using the available sudo permissions, I started a privileged shell.

After confirming that the shell now had elevated privileges, I verified the current user.

The shell had successfully changed from the low-privileged web account to:

```text
root
```

This demonstrated a common Linux security issue: **overly permissive sudo configuration** can allow a low-privileged account to obtain full system privileges.

---

# 12. Finding the Third Ingredient

With root privileges available, I could now access the previously restricted directory:

```text
/root
```

I enumerated the files there and located the final ingredient.

At this point, all three ingredients required by the Pickle Rick challenge had been obtained.

---

# Conclusion

The Pickle Rick room demonstrates a straightforward attack chain:

```text
Web Enumeration
      ↓
Source Code Analysis
      ↓
robots.txt
      ↓
Directory Enumeration
      ↓
Login Portal
      ↓
Command Execution
      ↓
Shell Access
      ↓
File Enumeration
      ↓
Sudo Enumeration
      ↓
Privilege Escalation
      ↓
Root Access
      ↓
Final Ingredient
```

## Key Takeaways

### 1. Always inspect source code

Information that isn't visible on a webpage can sometimes be present in its HTML source.

### 2. Don't ignore `robots.txt`

Although it is not a security mechanism, it can sometimes reveal interesting paths or information in CTF environments.

### 3. Enumerate before exploiting

Directory enumeration helped identify the login endpoint that wasn't obvious from the main page.

### 4. Restricted command interfaces can sometimes be bypassed within a lab

When a web interface limits available commands, understanding the underlying operating environment can reveal alternative ways to interact with it.

### 5. Check sudo permissions

For Linux privilege escalation, examining the current user's sudo privileges is an important enumeration step.

### 6. Follow clues carefully

The challenge is designed so that information discovered at one stage helps guide the investigation toward the next ingredient.

---

## Tools Used

- **Gobuster** — Web directory enumeration
- **Netcat** — Listener for the lab shell connection
- **Python 3** — Used as part of the shell-access technique
- **Linux commands** — File and system enumeration

## Final Result

All **three ingredients** were successfully discovered, completing the Pickle Rick TryHackMe challenge.
