# switcher Writeup

I approached this as a local reversing challenge. The prompt said:

> Whoopsie, some wild functions started switching my string. Please determine a string to fit their confusion.



The expected format of the flag was:

`texsaw{string}`

## Initial Recon

I started by listing the files in the directory and checking the binary type.

```bash
ls -la
file switcheroo
strings -a -n 4 switcheroo
```

That showed a single stripped ELF:

- `switcheroo`
- 64-bit ELF
- dynamically linked
- stripped

The strings output immediately gave me a few useful clues:

- `Please make a compatible password:`
- `You have entered the flag`
- `README.txt`
- `%27[^\n]`

From that, I knew the binary:

1. Reads up to 27 characters from stdin.
2. Probably expects an input length of exactly 27.
3. Builds or checks something involving `README.txt`.
4. Prints success only if a full chain of conditions is satisfied.

## Static Analysis

Since the binary was stripped, I used `objdump` and `readelf` to map the important functions:

```bash
objdump -d switcheroo
readelf -a switcheroo
objdump -s -j .rodata -j .data switcheroo
```

### Main Logic

The main function does the following:

1. Prints `Please make a compatible password:`
2. Reads input with `scanf("%27[^\n]")`
3. Checks `strlen(input) == 0x1b`
4. If length is 27, it passes the buffer into a validation chain

So I knew immediately that the string inside `texsaw{...}` had to be exactly 27 bytes total, not counting a newline.

## The Core Reordering Function

One helper function performs the "switching" mentioned in the challenge description.

It works on a 27-byte string and takes an integer parameter `n`.

The logic is:

- If `n` is even:
  - For `i` from `0` to `n-1`, it modifies byte `(i * n) % 27` by adding `n`
  - Then it rotates the whole 27-byte string
- If `n` is odd:
  - It rotates first
  - Then for `i` from `0` to `n-1`, it modifies byte `(i + n) % 27` by subtracting `n`

This function is called repeatedly with different values:

- `5`
- `6`
- `13`
- `3`
- `24`
- `10`
- `7`

At each stage, the program checks specific byte positions against constants.

## Early Constraint Checks

After the first two transformations, the binary checks:

- byte `11 == 0x6f` (`'o'`)

After the next transformation with `13`, it checks:

- byte `14 == 0x52` (`'R'`)

After `3` and `24`, it checks:

- byte `0 == 0x9b`
- byte `26` must be in the range `0x73..0x77`

After `10`, it checks:

- byte `8 == 0x59`
- byte `11 == 0x59`
- byte `12` must be in the range `0x74..0x77`

After `7`, it checks:

- byte `20 == 0xb5`
- byte `13 == 0x73`

At this point it was clear that the program does not compare the input directly. Instead, it repeatedly scrambles the string and checks properties of the scrambled state.

## README.txt Reconstruction

The most useful part came later.

After the final transformation stage, the program derives a filename from the transformed bytes and compares it against the static string stored in `.data`:

`README.txt`

This was the key observation because it let me translate a bunch of transformed byte positions into exact required values.

The generated filename is built like this:

- `name[0] = b[0] - 0x21`
- `name[1] = b[1] - 0x20`
- `name[2] = b[2] - 0x28`
- `name[3] = 2 * (b[3] + 4)`
- `name[4] = b[12] + 0x1c`
- `name[5] = b[11] - 0x66`
- `name[6] = b[10] + 8`
- `name[7] = b[9] + 0x14`
- `name[8] = b[8] - 7`
- `name[9] = -2 * (b[26] + 6)`

Those ten bytes must equal:

`README.txt`

That yields direct constraints on the transformed buffer.

For example:

- `b[0] = 'R' + 0x21 = 0x73`
- `b[1] = 'E' + 0x20 = 0x65`
- `b[2] = 'A' + 0x28 = 0x69`
- `2 * (b[3] + 4) = 'D' = 0x44`
- `b[12] = '.' - 0x1c = 0x12`
- `b[11] = 'E' + 0x66 = 0xab`

This confirmed that the transformed bytes were supposed to land on a very specific structure.

## The Hex Checks

After building `README.txt`, the program opens that file in mode `rb` and reads one byte at a time. There is no actual `README.txt` provided with the challenge, but that turns out not to matter.

The code derives four 2-character hexadecimal strings from transformed input bytes, then parses them using `strtol(..., 16)` and compares the resulting integers to fixed constants:

- `0x57`
- `0x34`
- `0x61`
- `0x29`

So the program is effectively forcing four hex pairs:

- `"57"`
- `"34"`
- `"61"`
- `"29"`

The helper used for some of those hex digits only accepts characters valid in hex:

- `0-9`
- `a-f`
- `A-F`

That narrowed the search space further.

## Why I Solved It Symbolically

At this point I could have tried to reverse every transformation manually, but that would have been slow and error-prone because:

- there are 27 positions
- the permutation is stateful
- some positions are modified before rotation and some after
- different stages stack on top of one another

So instead of brute forcing or hand-tracing, I modeled the transform symbolically in Python.

Each byte was represented as:

- original index
- additive offset modulo 256

That let me track exactly which original input position ended up at each checked location after every transformation stage.

## My Local Emulator

I wrote a small Python script that reproduced the binary logic:

- the 27-byte rotate/permutation function
- the add/subtract stages
- the `README.txt` derivation
- the hex-digit constraints

Then I used the symbolic mapping to determine which original input positions had to hold which characters.

The important recovered relationships were:

- the final flag had to start with `texsaw{`
- several middle positions were tightly constrained by the rotate/add/subtract chain
- the last bytes had to satisfy the `README.txt` and hex-derived checks

## Recovered Valid Input

One valid 27-byte input that satisfies all checks is:

```text
texsaw{pAt1ence!!_W0rKn0w?}
```

This is exactly 27 characters long.

## Validation

The provided file did not have the executable bit set in the challenge directory, so running `./switcheroo` directly returned:

```text
Permission denied
```

Because of that, I validated the result by emulating the binary from the disassembly rather than relying on direct execution.

My emulator confirmed:

- all byte-position checks pass
- the reconstructed filename becomes `README.txt`
- the parsed hex values are `[0x57, 0x34, 0x61, 0x29]`

## Final Flag

```text
texsaw{pAt1ence!!_W0rKn0w?}
```

## Short Version

If I had to summarize the solve path in one sentence:

I reversed the repeated 27-byte rotation/modification routine, modeled the transformations symbolically, used the generated `README.txt` constraint plus the forced hex-pair comparisons to pin down the original input, and recovered the valid flag:

`texsaw{pAt1ence!!_W0rKn0w?}`
