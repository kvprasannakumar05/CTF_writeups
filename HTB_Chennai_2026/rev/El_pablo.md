# El Pablo - rev writeup 

I solved this challenge as a 3-stage shard recovery, then stitched the final flag from the 3 binaries.

Challenge files:
- `c1` (ELF64, Linux)
- `c2.exe` (PE32 x86)
- `c3.exe` (PE32+ x64 Go)

Target format: `HTB{...}`

---

## 1) Solving `c1`

I started with `c1` because it is the cleanest binary.

### Quick triage
- `file c1` -> stripped 64-bit ELF.
- I loaded it in radare2 and checked `main` logic.

### Core check logic
`c1` asks for an 8-byte input (`Enter shard key:`) and checks each byte with:

```c
input[i] ^ (0x5a + i) == const[i]
```

The constant bytes in `.rodata` (first 8 relevant bytes) were:

```text
39 6f 2e 02 35 6c 19 12
```

So I inverted byte-by-byte:

```text
input[i] = const[i] ^ (0x5a + i)
```

This gives:

```text
c4r_k3ys
```

On success, the program prints shard 1 as:

```text
HTB{c4r_k3ys
```

So I kept:

- **Shard 1** = `HTB{c4r_k3ys`

(Length 12, including `HTB{`.)

---

## 2) Solving `c2.exe`

Then I moved to `c2.exe`.

### What I found in main
The binary asks:
- `Enter shard 1:` (length must be 12)
- `Enter shard 2:` (length must be 9)

Then:
1. It computes FNV-1a hash over shard1 (seeded with `0x811C9DC5`).
2. Uses that as seed for a second hash on shard2.
3. Compares result to `0xD3C5431D`.

At first glance this looks like I must solve a hash preimage for shard2 input.

### The useful shortcut
I reversed the success print helper (`fcn.401180`) directly.
It does **not** print user shard2 input. It decodes a fixed embedded 9-byte blob via:

```c
decoded[i] = enc[i] ^ (0x42 + i)
```

Decoded string:

```text
5c4tt3r3d
```

Success format string in `c2` is:

```text
Correct! Shard 2: _%s
```

So shard 2 output fragment is:

- **Shard 2** = `_5c4tt3r3d`

(Length 10 including the underscore.)

---

## 3) Solving `c3.exe`

`c3.exe` is a Go binary, so disassembly is noisy. I solved it by isolating the meaningful validator functions near:

- `0x1400b36c0` (main flow)
- `0x1400b3e40`, `0x1400b3f20`, `0x1400b3fe0`, `0x1400b4020`, `0x1400b33a0`

### Input shape
`c3` asks:

```text
Enter all 3 shards (space separated):
```

It checks lengths:
- shard1 length = 12
- shard2 length = 10
- shard3 length = 8

This matches exactly:
- `HTB{c4r_k3ys` (12)
- `_5c4tt3r3d` (10)
- unknown 8-byte shard3

### Recovering shard3 transform
I reconstructed these helper operations:

1. One function XORs stream bytes with repeating key bytes from constant `0x63347363`, i.e.:
   - key stream = `63 73 34 63 63 73 34 63`
   - ASCII = `c s 4 c c s 4 c`

2. Another function validates transformed bytes against fixed 8-byte constant from `0x110705533c783f57` (little-endian bytes):
   - target = `57 3f 78 3c 53 05 07 11`

So for shard3 bytes `x[i]`:

```text
x[i] ^ key[i] = target[i]
=> x[i] = target[i] ^ key[i]
```

Byte-wise result:
- `57 ^ 63 = 34` -> `4`
- `3f ^ 73 = 4c` -> `L`
- `78 ^ 34 = 4c` -> `L`
- `3c ^ 63 = 5f` -> `_`
- `53 ^ 63 = 30` -> `0`
- `05 ^ 73 = 76` -> `v`
- `07 ^ 34 = 33` -> `3`
- `11 ^ 63 = 72` -> `r`

So:

- **Shard 3** = `4LL_0v3r`

---

## 4) Final flag reconstruction

Now I combined all shard outputs in flag format order:

- `HTB{c4r_k3ys`
- `_5c4tt3r3d`
- `_4LL_0v3r`
- `}`

Final flag:

## `HTB{c4r_k3ys_5c4tt3r3d_4LL_0v3r}`

---

## Notes

- I solved this statically (no Wine needed).
- For `c2`, I bypassed preimage grinding by reversing the success formatter directly.
- For `c3`, I ignored Go runtime noise and focused only on custom transform/compare routines.
