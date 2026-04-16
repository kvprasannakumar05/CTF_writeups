# Brick City Office Space - Pwn Writeup 

## Challenge
**Name:** Brick City Office Space  
**Category:** Pwn / Binary Exploitation  
**Prompt:** "Help design the office space for Brick City's new skyscraper! read flag.txt for design specifications"  
**Remote:** `nc brick-city-office-space.pwn.ctf.umasscybersec.org 45001`

---

## My Goal
Get the binary to print the contents of `flag.txt` from the remote server.

---

## Initial Recon (What I Checked First)
I started by checking what files were given:
- `BrickCityOfficeSpace` (the 32-bit ELF binary)
- `libc.so.6`
- `ld-linux.so.2`
- `flag.txt` (local placeholder)

Then I checked protections (`checksec`) and saw:
- 32-bit, non-PIE
- NX enabled
- **No RELRO**
- No canary

The important part is **No RELRO**, because that means GOT entries are writable.

---

## Finding the Bug
I disassembled the binary and focused on the function that handles user input (`vuln`).

The key vulnerable line pattern was:
- read user input into buffer with `fgets`
- then call `printf(buffer)` directly

So user input is used as the **format string**.

That is a classic **format string vulnerability**.

---

## Why This Is Exploitable
With format strings, I can:
1. Leak memory values using `%p`
2. Write memory using `%n` / `%hn`

Since GOT is writable (no RELRO), I can overwrite a GOT function pointer to redirect execution.

Plan:
1. Leak libc address
2. Compute libc base
3. Compute `system` address
4. Overwrite `printf@GOT` with `system`
5. Send `cat flag.txt` so program calls `system("cat flag.txt")`

---

## Step 1 - Leak libc
I sent `%2$p` and `%3$p` and got values like:
- `0xf7ed2620`
- `0xf7ed2da0`

These matched offsets for `_IO_2_1_stdin_` and `_IO_2_1_stdout_` in provided libc.

From this, I used:
- `libc_base = leak_stdin - 0x22a620`
- `system = libc_base + 0x48170`

---

## Step 2 - Choose Overwrite Target
I used `printf@GOT` as target:
- `printf@GOT = 0x0804bbb0`

Reason:
- program calls `printf` multiple times with user-controlled strings
- if `printf` points to `system`, I can execute commands

---

## Step 3 - Write `system` into `printf@GOT`
On 32-bit, I split `system` address into two 16-bit chunks:
- lower 2 bytes -> write to `printf@GOT`
- upper 2 bytes -> write to `printf@GOT + 2`

I used `%4$hn` and `%5$hn` with calculated padding to write exact halfwords.

Payload structure looked like:
- first 8 bytes: packed addresses (`printf@GOT`, `printf@GOT+2`)
- then padding like `%<num>c`
- then `%4$hn`
- more padding
- then `%5$hn`

---

## Step 4 - Trigger Command Execution
After overwrite, the next `printf(user_input)` becomes `system(user_input)`.

So I sent:
- `cat flag.txt`

Program treated it as command, executed it, and printed flag.

---

## Final Flag
`UMASS{th3-f0rm4t_15-0ff-th3-ch4rt5}`

---

## Exploit Script I Used
Saved at:
- `/home/prasanna/umass/brick-city-office-space/solve.py`

It does:
1. Connect remote
2. Leak libc with `%2$p`
3. Compute `system`
4. Format-write `printf@GOT`
5. Send `cat flag.txt`
6. Print response

---

## Simple Takeaway
This challenge is a textbook format-string exploit:
- `printf(buffer)` is unsafe
- no RELRO makes GOT overwrite possible
- one libc leak + one GOT overwrite = command execution

If `printf` had been `printf("%s", buffer)` and full RELRO was enabled, this path would be blocked.
