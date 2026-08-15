# TryHackMe: W1seGuy — Writeup

## Overview

This room is a crypto challenge. We're given a `source.py` file plus a live service on port `1337`. The goal: figure out the encryption key and use it to reconstruct the flag.

## Understanding the Encryption

Looking at the source, the flag gets encrypted with a simple repeating-key XOR: each character of the flag is XORed against a character from a 5-byte key, cycling back to the start of the key once you run past its length.

Our job is to work backwards — recover the key from what we know, then use it to decrypt the ciphertext.

Since XOR is its own inverse, if we know both the plaintext and the ciphertext at a given position, we can pull out the key byte directly:

$$
C = P \oplus K \quad\Longrightarrow\quad K = C \oplus P
$$

where `C` is ciphertext, `P` is plaintext, `K` is key.

**Key insight:** THM flags always start with `THM{`. That gives us 4 known plaintext characters for free, which unlocks the first 4 bytes of the 5-byte key immediately.

For the 5th byte, there are two routes: brute-force it offline, or lean on the fact that flags close with `}` — since we know the key repeats every 5 characters, we can line up the final ciphertext byte against a plaintext `}` to recover the last key byte too.

## The Source Code

```python
import random
import socketserver 
import socket, os
import string

flag = open('flag.txt','r').read().strip()

def send_message(server, message):
    enc = message.encode()
    server.send(enc)

def setup(server, key):
    flag = 'THM{thisisafakeflag}' 
    xored = ""

    for i in range(0,len(flag)):
        xored += chr(ord(flag[i]) ^ ord(key[i%len(key)]))

    hex_encoded = xored.encode().hex()
    return hex_encoded

def start(server):
    res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))
    key = str(res)
    hex_encoded = setup(server, key)
    send_message(server, "This XOR encoded text has flag 1: " + hex_encoded + "\n")
    
    send_message(server,"What is the encryption key? ")
    key_answer = server.recv(4096).decode().strip()

    try:
        if key_answer == key:
            send_message(server, "Congrats! That is the correct key! Here is flag 2: " + flag + "\n")
            server.close()
        else:
            send_message(server, 'Close but no cigar' + "\n")
            server.close()
    except:
        send_message(server, "Something went wrong. Please try again. :)\n")
        server.close()

class RequestHandler(socketserver.BaseRequestHandler):
    def handle(self):
        start(self.request)

if __name__ == '__main__':
    socketserver.ThreadingTCPServer.allow_reuse_address = True
    server = socketserver.ThreadingTCPServer(('0.0.0.0', 1337), RequestHandler)
    server.serve_forever()
```

Two details worth pulling out:

1. The actual XOR loop:
```python
flag = 'THM{thisisafakeflag}' 
xored = ""
for i in range(0,len(flag)):
    xored += chr(ord(flag[i]) ^ ord(key[i%len(key)]))
```

2. How the key is generated — 5 random characters from letters + digits:
```python
res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))
key = str(res)
```

## Writing the Solver

Putting the key-recovery logic above into a script:

```python
import argparse

def derive_key_part(hex_encoded, known_plaintext, start_index):
    encrypted_bytes = bytes.fromhex(hex_encoded)
    derived_key = ""
    
    for i in range(len(known_plaintext)):
        derived_key += chr(encrypted_bytes[start_index + i] ^ ord(known_plaintext[i]))
    
    return derived_key

def xor_decrypt(hex_encoded, key):
    encrypted_bytes = bytes.fromhex(hex_encoded)
    decrypted_message = ""
    
    for i in range(len(encrypted_bytes)):
        decrypted_message += chr(encrypted_bytes[i] ^ ord(key[i % len(key)]))
    
    return decrypted_message

def main():
    parser = argparse.ArgumentParser(description='W1seGuy XOR Decryption')
    parser.add_argument('hex_encoded', type=str, help='Hex encoded string to decrypt')

    args = parser.parse_args()
    hex_encoded = args.hex_encoded
    key_length = 5

    known_start_plaintext = 'THM{'
    known_end_plaintext = '}'

    derived_key_start = derive_key_part(hex_encoded, known_start_plaintext, 0)
    print("Derived start of the key:", derived_key_start)

    derived_key_end = derive_key_part(hex_encoded, known_end_plaintext, len(hex_encoded) // 2 - 1)
    print("Derived end of the key:", derived_key_end)

    derived_key = (derived_key_start + derived_key_end)[0:key_length]
    print("Derived key:", derived_key)

    decrypted_message = xor_decrypt(hex_encoded, derived_key)
    print("Decrypted message:", decrypted_message)

if __name__ == '__main__':
    main()
```

## Getting Flag 1

Running the script against the hex blob the service sends us recovers both the key and the first flag in one shot.

## Getting Flag 2

Submit that recovered key back to the service, and it hands over the second flag as a reward for guessing it correctly.
.

There's also a cleaner solve using `pwntools` by Jaxafed, if you want to see the same idea automated more tightly:
https://jaxafed.github.io/posts/tryhackme-w1seguy/
