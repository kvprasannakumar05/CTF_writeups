

---

# Nonce (nonce-rewind)

**Category:** Cryptography

**Author:** luvalentine

**Flag:** `f13{3cd54_n0nc3_r3u53_t1m3_buck37}`

<img width="871" height="427" alt="image" src="https://github.com/user-attachments/assets/0b640025-0377-48f8-b156-27fe5cb8e3d3" />



## Overview

"Nonce" was an Elliptic Curve Digital Signature Algorithm (ECDSA) challenge. We were given access to a TCP server called "TimeSign Gateway v2" that acts as a signing oracle. The objective was to forge a valid signature for the blocked admin ticket `676976655f6d655f666c6167` (which decodes to `give_me_flag`).

As the challenge name implies, the vulnerability stems from a weak nonce generation mechanism, allowing for a classic ECDSA nonce reuse attack to recover the server's private key.

## Initial Reconnaissance

Connecting to the server over `nc` provided a simple prompt with a few commands: `help`, `pubkey`, `sign`, and `verify`.

Plaintext

```
┌──(prasanna㉿Prasannakumar)-[~]
└─$ nc nonce-rewind.ctf.fr13nds.team 31337
== TimeSign Gateway v2 ==
We sign client tickets, but the admin ticket is blocked.
Admin ticket (hex): 676976655f6d655f666c6167

Commands:
  help
  pubkey
  sign <hex_message>
  verify <hex_message> <r_hex> <s_hex>
  quit
tsg> 
```

Testing the `sign` command with some arbitrary hex values returned an $r$ value, an $s$ value, and a suspicious `trace` hex value.

Plaintext

```
tsg> sign 41414141
r = fec336194d322152cb33ba5c869179d20c9ee9f529e5765bcfc9d1e6a3694287
s = fc0f573a691fe1f371364c0320cac36efe4c1449e049c1394afd533fde55e3db
trace = a158
```

The banner "TimeSign Gateway" was a massive hint. It suggested the server was seeding its PRNG using the current system time (likely `time.time()`). I wrote a quick `pwntools` script to fire off rapid, consecutive `sign` requests to observe the `trace` behavior.

The results confirmed the vulnerability: if requests were sent fast enough to hit the server within the same exact second, the `trace` value remained identical, and more importantly, the server generated the exact same nonce ($k$).

## The Vulnerability: ECDSA Nonce Reuse

In ECDSA, the $r$ value of a signature is derived purely from the nonce $k$ and the curve's generator point $G$:

$$r = (k \cdot G)_x \pmod n$$

If a server uses the same $k$ to sign two different messages (resulting in hashes $z_1$ and $z_2$), the signatures will share the exact same $r$ value. Once we have two signatures with the same $r$, the security of ECDSA completely collapses. We can isolate the nonce $k$ using simple algebra:

$$k \equiv (z_1 - z_2)(s_1 - s_2)^{-1} \pmod n$$

With $k$ recovered, calculating the server's private key $d$ is trivial:

$$d \equiv r^{-1}(k \cdot s_1 - z_1) \pmod n$$

## Exploitation and The Curveball

To exploit this, I needed to send two different messages fast enough to catch the 1-second time window, confirm $r_1 = r_2$, and execute the math above.

However, my initial exploit failed to generate a valid signature. While the nonce reuse was occurring, the calculated private key $d$ was incorrect. In ECDSA, calculating $k$ and $d$ relies heavily on $z$ (the integer representation of the message) and $n$ (the order of the curve).

CTF authors often tweak standard parameters to break basic scripts. To counter this, I updated my script to dynamically verify the recovered private key against the server's public key (`Qx`) using $Q = d \cdot G$.

I programmed the script to test a matrix of possibilities:

1. **Curves:** `NIST256p` vs `SECP256k1`
    
2. **Hashing Methods:** Raw Integer vs SHA256 of bytes vs SHA256 of ASCII string.
    

## The Exploit Script

The final script automatically brute-forces the server's configuration matrix after catching the time window, forges the admin signature, and captures the flag.

Python

```
from pwn import *
import hashlib
from Crypto.Util.number import inverse
from ecdsa import SigningKey, NIST256p, SECP256k1

context.log_level = 'error'

curves = {
    "NIST256p": (NIST256p, 0xffffffff00000000ffffffffffffffffbce6faada7179e84f3b9cac2fc632551),
    "SECP256k1": (SECP256k1, 0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141)
}

def solve():
    attempts = 1
    while True:
        print(f"[*] Attempt {attempts}...")
        io = remote('nonce-rewind.ctf.fr13nds.team', 31337)
        io.recvuntil(b'tsg> ')

        io.sendline(b'pubkey')
        io.recvuntil(b'Qx = ')
        qx_target = int(io.recvline().strip(), 16)
        io.recvuntil(b'tsg> ')

        msg1_hex = b'11111111'
        msg2_hex = b'22222222'

        io.sendline(b'sign ' + msg1_hex)
        io.recvuntil(b'r = ')
        r1 = int(io.recvline().strip(), 16)
        io.recvuntil(b's = ')
        s1 = int(io.recvline().strip(), 16)
        io.recvuntil(b'tsg> ')

        io.sendline(b'sign ' + msg2_hex)
        io.recvuntil(b'r = ')
        r2 = int(io.recvline().strip(), 16)
        io.recvuntil(b's = ')
        s2 = int(io.recvline().strip(), 16)
        io.recvuntil(b'tsg> ')

        if r1 != r2:
            io.close()
            attempts += 1
            continue

        print("[+] Nonce reuse confirmed!")
        
        z_methods = {
            "SHA256(bytes)": (
                int.from_bytes(hashlib.sha256(bytes.fromhex(msg1_hex.decode())).digest(), 'big'),
                int.from_bytes(hashlib.sha256(bytes.fromhex(msg2_hex.decode())).digest(), 'big')
            ),
            "No Hash (Raw Integer)": (
                int(msg1_hex.decode(), 16),
                int(msg2_hex.decode(), 16)
            ),
            "SHA256(ASCII String)": (
                int.from_bytes(hashlib.sha256(msg1_hex).digest(), 'big'),
                int.from_bytes(hashlib.sha256(msg2_hex).digest(), 'big')
            )
        }

        correct_d = None
        correct_z_method = None
        correct_curve = None
        correct_curve_obj = None

        for curve_name, (curve_obj, n) in curves.items():
            for method_name, (z1, z2) in z_methods.items():
                try:
                    k = ((z1 - z2) * inverse((s1 - s2) % n, n)) % n
                    if k == 0: continue
                    d = (inverse(r1, n) * (k * s1 - z1)) % n
                    
                    sk = SigningKey.from_secret_exponent(d, curve=curve_obj)
                    if sk.get_verifying_key().pubkey.point.x() == qx_target:
                        correct_d = d
                        correct_z_method = method_name
                        correct_curve = curve_name
                        correct_curve_obj = curve_obj
                        print(f"[+] Key verified! Server uses: {method_name} on {curve_name}")
                        break
                except Exception:
                    pass
            if correct_d: break

        if not correct_d:
            print("[-] Still failed to verify 'd'.")
            io.close()
            break

        print(f"[+] Verified Private Key (d): {hex(correct_d)}")

        sk = SigningKey.from_secret_exponent(correct_d, curve=correct_curve_obj)
        admin_hex = "676976655f6d655f666c6167"
        
        if correct_z_method == "No Hash (Raw Integer)":
            z_admin = int(admin_hex, 16)
            admin_bytes = z_admin.to_bytes(32, 'big')
            sig = sk.sign_digest(admin_bytes)
        elif correct_z_method == "SHA256(ASCII String)":
            sig = sk.sign(admin_hex.encode(), hashfunc=hashlib.sha256)
        else:
            admin_bytes = bytes.fromhex(admin_hex)
            sig = sk.sign(admin_bytes, hashfunc=hashlib.sha256)

        r_forged = sig[:32].hex()
        s_forged = sig[32:].hex()

        print(f"[+] Sending forged signature for admin ticket...")
        io.sendline(f"verify {admin_hex} {r_forged} {s_forged}".encode())
        
        print("[*] Catching final server output...")
        try:
            final_output = io.recvall(timeout=5).decode(errors='ignore').strip()
            print("\n" + "="*40)
            print(final_output)
            print("="*40 + "\n")
        except Exception as e:
            print(f"[-] Error reading final output: {e}")
            
        break

if __name__ == '__main__':
    solve()
```

## Execution

Running the script quickly bypassed the latency issue, confirmed the server was using the `SECP256k1` curve with standard byte hashing, and dropped the flag.

Plaintext

```
[*] Attempt 1...
[*] Attempt 2...
[*] Attempt 3...
[*] Attempt 4...
[+] Nonce reuse confirmed!
[+] Key verified! Server uses: SHA256(bytes) on SECP256k1
[+] Verified Private Key (d): 0xdf70bdf9a81956e69afc380de53d702d573ab8e05a746c2b451c48b615281a88
[+] Sending forged signature for admin ticket...
[*] Catching final server output...

========================================
signature valid
flag: f13{3cd54_n0nc3_r3u53_t1m3_buck37}
tsg>
========================================
```
