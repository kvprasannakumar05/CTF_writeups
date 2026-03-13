# Rude Guard - Writeup

## Challenge Description
"There's a guard that's protecting the flag! How do I sneak past him?"
Binary: `pwnable`

## Initial Recon
I started by checking the binary's protections and characteristics:
```bash
$ file pwnable
pwnable: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked ...
$ checksec pwnable
[*] '/home/prasanna/utctf/pwn_rude_guard/pwnable'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX unknown - GNU_STACK missing (Executable Stack)
    PIE:        No PIE (0x400000)
```

Key observations:
1. **NX is disabled**: The stack is executable.
2. **No Canary**: Classic buffer overflows are possible without worrying about stack cookies.
3. **No PIE**: The binary base address is fixed, which makes function pointers and ROP gadgets stable across runs.

## Vulnerability Discovery
I dumped the binary's code using `objdump` and analyzed the `main` and `read_input` functions.

### The "Gatekeeper" Check
In `main`, the program expects a command-line argument. It uses `atoi()` on the first argument and checks if it matches `0x656c6c6f` (hex for "hello" in little-endian ASCII, decimal `1701604463`). If it doesn't match, the guard is rude and the program exits.

### The Overflow
Once past the check, `main` calls `read_input()`. Inside `read_input()`, there's a stack-based buffer of 32 bytes, but the program uses `read(0, buffer, 100)`, which allows us to write well past the buffer's bounds.

```c
void read_input(int fd) {
    char buffer[32]; // [rbp-0x20]
    read(0, buffer, 0x64); // Reads up to 100 bytes
}
```

## Exploitation
The goal was to redirect execution to `secret_function` at `0x40124f`.

### Calculating the Offset
The buffer is at `rbp-0x20` (32 bytes). To reach the return address, I need to cover:
- 32 bytes of the buffer.
- 8 bytes of the saved `rbp`.
- **Total Offset**: 40 bytes.

### The Payload
`[ "A" * 40 ] + [ 0x40124f (secret_function address) ]`

## Flag Decryption logic
The `secret_function` doesn't just print a flag; it decrypts it using an XOR algorithm. It uses a series of hardcoded 64-bit values and XORs each byte with `0x32`.

To ensure I got the flag despite some local stack alignment issues during execution, I reversed the XOR logic directly:

```python
# Extraction from secret_function assembly:
xor_key = 0x32
# Constants extracted: 0x554955535e544647, 0x4106456d56400647, ...
# XOR logic applied to get the flag.
```

## Flag
`utflag{gu4rd_w4s_w34ker_th4n_i_th0ught}`
