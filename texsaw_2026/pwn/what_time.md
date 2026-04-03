# what_time - Pwn Writeup 

## Challenge
I got the challenge prompt:

> "I think one of the hands of my watch broke. Can you tell me what the time is?"

Remote: `nc 143.198.163.4 3000`

Goal: recover flag in format `texsaw{...}`.

## My Setup and Tools
I used:
- `file` to identify architecture
- `checksec` / `pwn checksec` for mitigations
- `strings` for quick function/string hints
- `nm` for symbol addresses
- `objdump -d -M intel` for static reversing
- `python3` with `socket` + `struct` for exploit automation

(There was no need for dynamic local execution because the binary was 32-bit and this environment lacked the required loader.)

## Initial Recon
I first confirmed basic binary properties:
- 32-bit ELF (`i386`), dynamically linked, not stripped
- `NX enabled`
- `No PIE` (fixed code addresses)
- `No stack canary`
- `Partial RELRO`

The binary exported useful symbols:
- `read_user_input` at `0x0804925c`
- `win` at `0x080491f6`

## Reversing Notes
From disassembly of `main`:
1. It computes a time-derived value (minute-rounded epoch style transform).
2. It prints the current time string.
3. It calls `read_user_input(time_value)`.

Inside `read_user_input`:
1. `malloc(0xa0)` then `read(0, heap_buf, 0xa0)`.
2. It XOR-decodes each byte in 4-byte blocks using `(key + block_index)` where `key` is the argument from `main`.
3. It does `memcpy(stack_buf_64, heap_buf, bytes_read)` where `stack_buf_64` is only 64 bytes.
4. Then `write(1, stack_buf_64, 0x28)` prints 40 bytes.

## Vulnerability
The core bug is a **stack buffer overflow**:
- Destination buffer is 64 bytes on stack.
- Copy length is attacker-controlled (`bytes_read` up to `0xa0`).
- No bounds check before `memcpy`.

So I can overwrite saved EIP.

There is also a useful side effect: since the service prints 40 bytes from the decoded stack buffer, I can leak decoded bytes.

## Exploit Strategy
The input is XOR-decoded server-side, so I can’t just send raw ROP bytes directly. I handled this in two phases:

1. **Recover keystream/key for current minute**
- Open connection A.
- Send `40` null bytes.
- Service decodes and echoes first 40 bytes.
- Since `0x00 ^ K = K`, echoed bytes directly reveal keystream.
- First 4 leaked bytes are the base key dword.

2. **Exploit in same minute window**
- Open connection B quickly (same displayed minute).
- Build desired plaintext overflow:
  - `"A" * 68` to reach saved EIP
  - then ROP to call `system("/bin/sh")` (ret2system)
- Encode payload as: `cipher[i] = plain[i] ^ keystream[i]`.
- Send encoded payload so server decodes into my real overflow bytes.

I used ret2system chain because it was reliable:
- `system@plt = 0x080490b0`
- return address = `0x0804935f` (safe re-entry target)
- argument `/bin/sh` string = `0x0804a018`

Then I sent command: `cat flag.txt`.

## Result
The remote output returned:

`texsaw{7h4nk_u_f0r_y0ur_71m3}`

## Flag
`texsaw{7h4nk_u_f0r_y0ur_71m3}`

## Saved Artifacts
- Exploit script: `/home/prasanna/Desktop/CTF/texaw_2026/pwn/what_time/solve.py`
- This writeup: `/home/prasanna/Desktop/CTF/texaw_2026/pwn/what_time/WRITEUP.md`
