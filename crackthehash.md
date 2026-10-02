# [Hacking walkthrough] Cracking the hashes

Today I worked through part of the [tryhackme](https://tryhackme.com) challenge called [crackthehash](https://tryhackme.com/room/crackthehash). It's a fun one — you're handed a pile of hashes and have to figure out what algorithm each one is before you can even think about cracking it. Below are the three toughest ones from Level 2: a salted SHA1, a SHA512crypt, and a bcrypt hash.

Instead of firing up hashcat on a GPU rig, I cracked all three with short Python scripts against the classic [rockyou.txt](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt) wordlist — `hashlib`/`hmac` and the stdlib `crypt` module already speak most of these formats natively, and the `bcrypt` pip package handles the rest. I've included the equivalent hashcat command under each one too, in case you'd rather point a GPU at it.

## Task 2-1: SHA1 with salt hash

**Hash: e5d8870e5bdd26602cab8dbe07a942c8669e56d6**
**Salt: tryhackme**

**Solution**: This one's a trap. A 40-character hex string screams plain SHA1 (hashcat mode 100), and the hash-identifier tools agree — but running it against rockyou.txt as `sha1(word)` or `sha1(word+salt)` or `sha1(salt+word)` gets you nowhere. The room's hint gives it away: this is actually **HMAC-SHA1** (hashcat mode 160), where "salt" is really the HMAC key, not something you concatenate onto the password.

```python
import hmac, hashlib

target = "e5d8870e5bdd26602cab8dbe07a942c8669e56d6"
key = b"tryhackme"

with open("rockyou.txt", "rb") as f:
    for line in f:
        word = line.rstrip(b"\n")
        if hmac.new(key, word, hashlib.sha1).hexdigest() == target:
            print("Password:", word)
            break
```

Hashcat version, if you'd rather:
```
hashcat -m 160 'e5d8870e5bdd26602cab8dbe07a942c8669e56d6:tryhackme' rockyou.txt
```

**Answer**: 481616481616

## Task 2-2: SHA512crypt, $6$ hash

**Hash: $6$aReallyHardSalt$6WKUTqzq.UQQmrm0p/T7MPpMbGNnzXPMAXi4bJMl9be.cfi3/qxIf.hsGpS41BqMhSrHVXgMpdjS6xeKZAs02.**

**Solution**: The `$6$` prefix is the giveaway for SHA512crypt (the classic Unix `/etc/shadow` format), hashcat mode 1800. The salt (`aReallyHardSalt`) is embedded right in the hash string between the 2nd and 3rd `$`. This algorithm is intentionally slow — it runs thousands of rounds internally — so cracking it online isn't an option. Python's built-in `crypt` module speaks this format natively, so no third-party library is even needed:

```python
import crypt

target = "$6$aReallyHardSalt$6WKUTqzq.UQQmrm0p/T7MPpMbGNnzXPMAXi4bJMl9be.cfi3/qxIf.hsGpS41BqMhSrHVXgMpdjS6xeKZAs02."
salt = "$6$aReallyHardSalt"

with open("rockyou.txt", encoding="latin-1") as f:
    for line in f:
        word = line.rstrip("\n")
        if crypt.crypt(word, salt) == target:
            print("Password:", word)
            break
```

Hashcat version:
```
hashcat -m 1800 hash.txt rockyou.txt
```

**Answer**: waka99

## Task 2-3: bcrypt-blowfish hash

**Hash: $2y$12$Dwt1BZj6pcyc3Dy1FWZ5ieeUznr71EeNkJkUlypTsgbX1H68wsRom**

**Solution**: `$2y$12$` identifies bcrypt at cost factor 12 (hashcat mode 3200) — no online cracker touches this one. Cost 12 means each guess is deliberately expensive (~0.3s per try on a single CPU core here), so throwing the full 14-million-line rockyou.txt at it would take weeks. The trick is to shrink the wordlist first. The room hints that the password is a short, lowercase word, so filtering rockyou down to 4-letter entries cuts the search space to under 10,000 candidates:

```bash
grep -E '^[a-zA-Z]{4}$' rockyou.txt > four_letter.txt   # 9,786 candidates
```

```python
import bcrypt

target = b"$2y$12$Dwt1BZj6pcyc3Dy1FWZ5ieeUznr71EeNkJkUlypTsgbX1H68wsRom"

with open("four_letter.txt") as f:
    for line in f:
        word = line.strip()
        if bcrypt.checkpw(word.encode(), target):
            print("Password:", word)
            break
```

Hashcat version:
```
hashcat -m 3200 hash.txt four_letter.txt
```

**Answer**: bleh

## Identify hash

If you're ever unsure what kind of hash you're looking at, a quick [hash type checker](https://md5hashing.net/hash_type_checker) will list every plausible format — just don't trust it blindly, since (as Task 2-1 shows) it can't tell plain SHA1 apart from HMAC-SHA1.

## Conclusion

These three were the "hard mode" hashes in the room, and each one teaches a different lesson: don't trust the identifier tool over the room's hints, know which hash formats run native in your language's standard library, and when an algorithm is slow by design, shrink the wordlist instead of the patience. Hope this helps if you're stuck on the same room — happy cracking!

#### Reference and link

- Crack the hash room –> <https://tryhackme.com/room/crackthehash>
- hashcat –> <https://hashcat.net>
- hashcat mode list –> <https://hashcat.net/wiki/doku.php?id=example_hashes>
- Hash identifier –> <https://md5hashing.net/hash_type_checker>
- rockyou.txt –> <https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt>
