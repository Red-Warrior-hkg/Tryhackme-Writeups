# W1seGuy — TryHackMe CTF Writeup

**Platform:** TryHackMe  
**Room:** W1seGuy  
**Category:** Cryptography  
**Difficulty:** Easy  
**Status:** Completed ✅

![Room Page](room_page.png)

---

## Overview

This room presents a TCP server that encrypts a secret flag using XOR with a randomly generated key and challenges you to recover the key. The vulnerability lies in using a **short repeating key** to encrypt a flag whose **format is partially known**, making it susceptible to a known-plaintext attack combined with a short brute-force.

---

## How the Server Works

```python
res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))
key = str(res)

for i in range(0, len(flag)):
    xored += chr(ord(flag[i]) ^ ord(key[i % len(key)]))

hex_encoded = xored.encode().hex()
```

1. A **5-character alphanumeric key** is generated randomly
2. The real flag is XOR'd with the repeating key, character by character
3. The result is hex-encoded and sent to the client
4. The client must reply with the correct key to receive Flag 2

---

## Vulnerability

### Known-Plaintext XOR Attack

XOR encryption has a fundamental property:

```
plaintext  XOR key = ciphertext
ciphertext XOR key = plaintext
ciphertext XOR plaintext = key   ← key recovery
```

If any portion of the plaintext is known, the corresponding key bytes can be recovered directly by XOR-ing the ciphertext with the known plaintext.

All TryHackMe flags follow the format `THM{...}`, giving us **4 known plaintext bytes** at the start of every flag. Since the key is only 5 characters long, only **1 byte remains unknown**.

---

## Exploitation

### Step 1 — Recover 4 key bytes using known plaintext

```python
known_prefix = "THM{"
ct = bytes.fromhex(ciphertext_hex)

partial_key = [ct[i] ^ ord(known_prefix[i]) for i in range(4)]
```

### Step 2 — Brute-force the 5th key byte

The key uses `string.ascii_letters + string.digits` — only **62 possible values**.

```python
for c in ALPHANUM:
    key_candidate = partial_key + [ord(c)]
    decrypted = "".join(chr(ct[i] ^ key_candidate[i % 5]) for i in range(len(ct)))
    
    if decrypted.startswith("THM{") and decrypted.endswith("}") and decrypted.isprintable():
        # Key found
```

### Step 3 — Send the recovered key to get Flag 2

```python
s.send((key + "\n").encode())
response = s.recv(4096).decode()
```

---

## Full Exploit Script

```python
import socket
import string

HOST = "TARGET_IP"
PORT = 1337
KEY_LEN = 5
ALPHANUM = string.ascii_letters + string.digits  # same charset as server

def recover_key_and_flag(ciphertext_hex: str) -> tuple[str, str]:
    ct = bytes.fromhex(ciphertext_hex)
    
    # We know plaintext starts with "THM{" → recover first 4 key bytes
    known_prefix = "THM{"
    partial_key = [ct[i] ^ ord(known_prefix[i]) for i in range(4)]
    
    # Brute-force the 5th key byte
    for c in ALPHANUM:
        key_candidate = partial_key + [ord(c)]
        
        # Decrypt full ciphertext with this key
        decrypted = ""
        for i in range(len(ct)):
            decrypted += chr(ct[i] ^ key_candidate[i % KEY_LEN])
        
        # Valid flag: starts with THM{, ends with }, all printable
        if decrypted.startswith("THM{") and decrypted.endswith("}") and decrypted.isprintable():
            key_str = "".join(chr(b) for b in key_candidate)
            return key_str, decrypted
    
    return None, None

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect((HOST, PORT))

    data = s.recv(4096).decode()
    print("[*] Received:", data.strip())

    hex_encoded = data.strip().split(": ")[1]

    key, flag1 = recover_key_and_flag(hex_encoded)
    
    if key:
        print(f"[*] Recovered key: {key}")
        print(f"[*] Flag 1 (decrypted): {flag1}")
    else:
        print("[!] Key brute-force failed")
        exit()

    s.recv(4096)  # "What is the encryption key? "
    s.send((key + "\n").encode())

    response = s.recv(4096).decode()
    print("[*] Server response:", response.strip())
```

---

## Exploit Output

![Exploit Output](exploit_output.png)

The recovered key successfully decrypted Flag 1 and the server returned Flag 2 upon validation.

---

## Attack Flow

```
Server                                 Attacker
  │                                       │
  │──── hex(flag XOR key) ──────────────▶ │
  │                                       │  XOR with "THM{" → key[0..3]
  │                                       │  brute-force key[4] (62 attempts)
  │                                       │  decrypt & validate flag format
  │◀─── send recovered key ────────────── │
  │                                       │
  │──── "Congrats! Flag 2: THM{...}" ───▶ │
```

---

## Why This Works

| Factor | Detail |
|---|---|
| Known plaintext | All flags start with `THM{` — 4 bytes free |
| Short key | Only 5 characters, so only 1 byte needs brute-forcing |
| Small keyspace | 62 alphanumeric options for the last byte — trivially fast |
| No authentication | XOR alone has no integrity check — any guess can be verified locally |

---

## Remediation

| Issue | Fix |
|---|---|
| Short repeating key | Use a key at least as long as the message (one-time pad) or use a proper cipher |
| Predictable plaintext | Do not encrypt data with known/fixed prefixes without additional protection |
| No integrity check | Use authenticated encryption such as AES-GCM or ChaCha20-Poly1305 |

---

## Key Takeaway

> XOR encryption is completely broken when the plaintext is partially known and the key is short. A single known prefix is enough to recover the full key and decrypt the entire message.

---

## Room Completion

![Room Completed](room_completed.png)