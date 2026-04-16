# Batcave Bitflips Writeup

I started with the challenge description and the three hints:

- there are 3 bugs
- "Rotation rotation rotation!"
- "Something about that SBOX seems off..."

The challenge directory only contained one file, `batcave_license_checker`, so this was clearly a reverse/recover-the-intended-logic task rather than something involving multiple assets or network interaction.

## Initial Recon

First I checked what kind of binary I was dealing with. It was a 64-bit ELF, dynamically linked, not stripped. That was immediately useful because function names and global names were still present.

Running `strings` and looking at the symbol table gave me the important names:

- `rotate`
- `expand_state`
- `substitute`
- `mix`
- `derive_final`
- `hash`
- `verify`
- `decrypt_flag`
- globals named `EXPECTED`, `FLAG`, `SBOX`, and `LICENSE_KEY`

That already suggested the structure:

1. read a license key
2. hash it through a custom transformation
3. compare the result against `EXPECTED`
4. if it matches, decrypt the flag with the hash

## Looking at the Binary

I disassembled the interesting functions with `objdump -d -Mintel`.

### `expand_state`

This function builds a 64-byte state from the 32-byte input:

```c
state[i] = input[i % 32] ^ i;
```

So the license key length is effectively 32 bytes.

### `substitute`

This is a straight byte substitution:

```c
state[i] = SBOX[state[i]];
```

### `mix`

This mutates each byte using neighbors:

```c
state[i] ^= state[(i + 1) % 64] ^ state[63 - i];
```

### `derive_final`

This collapses the 64-byte state into a 32-byte final hash:

```c
out[i] = state[i] ^ state[i + 32];
```

### `hash`

The main hash loop repeatedly calls:

1. `substitute`
2. `mix`
3. `rotate`

for a huge number of rounds, then calls `derive_final`.

## Bug 1: Broken Rotate

The `rotate` function was the first obvious bitflip candidate. In assembly it effectively did:

```c
out = (x << 3) | (x >> 6);
```

That makes no sense as a normal 8-bit rotate. If the left side is `x << 3`, the wraparound part should be `x >> 5`, not `x >> 6`.

So this line was almost certainly corrupted by a bitflip:

```c
(x << 3) | (x >> 5)
```

That matched the second hint exactly.

## Bug 2: Broken S-Box

Next I dumped the S-box and checked whether it was a valid permutation of 0..255. It was not.

The S-box had:

- `0x43` twice
- `0x44` missing

That is a classic sign of a one-byte data corruption. Since the hint explicitly called out the S-box, this was clearly the second intended bug.

So the duplicated `0x43` should be restored to `0x44`.

## Checking the Embedded License

There was also a 32-byte string embedded in `.data`:

```text
!_batman-robin-alfred_((67||67))
```

At first this looked like it might be the right license key, or at least very close to it. I tested a few obvious variants, including changing `67||67` to `67||68`, and also the versions with and without a leading `!`.

None of those worked against the broken binary, and even patching the obvious rotate/S-box issues still did not immediately produce a successful verification path.

That told me the third bug was likely somewhere else in the decryption path rather than in the hash input alone.

## Bug 3: Broken Flag Decryption

Then I looked at `decrypt_flag`.

This function loops over the 32-byte flag buffer and combines each flag byte with the final hash byte. But the operation in the binary was:

```c
FLAG[i] |= hash[i % 32];
```

Using OR for flag decryption is extremely suspicious. OR is lossy and is a bad choice for reversible encryption. In CTF binaries like this, XOR is the usual intended operation.

So the third bug was that `decrypt_flag` used OR instead of XOR.

The intended logic should be:

```c
FLAG[i] ^= hash[i % 32];
```

## Recovering the Flag

At that point I realized I did not actually need the exact valid license key to recover the flag.

The binary already stores the correct target hash in `EXPECTED`, and if the intended decryption uses XOR, then the plaintext flag is just:

```c
plaintext = FLAG ^ EXPECTED
```

I extracted both 32-byte arrays from the binary:

### `EXPECTED`

```text
3b54751a2406af05778047c5e483d348cb8730de1a9145ab15c79b2204022bee
```

### encrypted `FLAG`

```text
6e193449777df05a07b433a68ce6e617fbe96fae2ee526c370e3c47d277f2b00
```

Then I XORed them byte-for-byte.

That produced:

```text
UMASS{__p4tche5_0n_p4tche$__#}\x00\xee
```

The `\x00` is the string terminator, so the real flag ends at the closing brace:

```text
UMASS{__p4tche5_0n_p4tche$__#}
```

## Final Answer

```text
UMASS{__p4tche5_0n_p4tche$__#}
```

## Short Solve Summary

The three intended bugs were:

1. `rotate` used `x >> 6` instead of `x >> 5`
2. the S-box had a duplicated `0x43` and was missing `0x44`
3. `decrypt_flag` used `OR` instead of `XOR`

The cleanest solve path was to identify the broken decryption op and compute:

```text
FLAG ^ EXPECTED
```

which directly revealed the plaintext flag.
