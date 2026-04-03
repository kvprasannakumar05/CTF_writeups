# Ragebait Writeup 



The path in the prompt (`/texsaw/Ragebait`) did not exist, so I used the actual file I found.

## 1) Initial Recon

I fingerprinted the file with `file` and `checksec`:

- 64-bit ELF, stripped
- NX enabled, canary present, No PIE
- Large `.text` section, which suggested lots of obfuscated checker branches

I ran `strings` and saw many suspicious gibberish-like constants and multiple fake-looking flag messages. That already hinted this challenge was meant to waste time via decoys.

## 2) Entry Analysis

I disassembled `main` and recovered the high-level flow:

1. It requires exactly 32-byte input (`strlen == 0x20`).
2. It computes FNV-1a on only the first 9 bytes.
3. It computes `% 1009`.
4. It dispatches to a function pointer table in `.data` (`0x44e080`) and calls one of many branch handlers.

So the first 9 chars select which checker runs.

## 3) Why Dynamic Testing Was Needed

The binary had a huge number of branch functions. Many branches:

- print fake success flags,
- print joke errors,
- exit immediately,
- or intentionally waste time / crash.

This is the “ragebait” part.

I ran broad testing over prefix pairs in `texsaw{??...` and confirmed multiple fake success outputs like:

- `texsaw{n0t_th3_fl4g_lol}`
- `texsaw{fake_flag_do_not_submit}`
- `texsaw{maybe_the_real_fake_flag_was_the_friends_we_made}`

So success output alone was not enough.

## 4) Isolating the Real Checker

I mapped branch behavior and statically triaged dispatched functions, looking for a branch that:

- reads many bytes from user input,
- performs deterministic comparisons,
- and only then calls the success print format.

That isolated one real validator function at `0x42fe9c`.

## 5) Reversing the Real Validator (`0x42fe9c`)

This function uses 4 accumulators and processes all 32 chars:

- for each position `i`, it updates accumulator `i % 4`
- update rule is:
  `acc = acc * 131 + input_byte`

After processing, it compares the 4 accumulators against hardcoded 64-bit constants.

This means each residue class (positions `0,4,8,...`; `1,5,9,...`; etc.) forms a base-131 number.
Instead of brute forcing, I inverted each constant by repeated `% 131` and `// 131` to recover exact bytes directly.

Recovered groups decoded to:

- group0: `taV_4m06`
- group1: `ewhUkE_r`
- group2: `x{Y_3_4y`
- group3: `sVdM_sn}`

Interleaving by position reconstructs the 32-byte input:

`texsaw{VVhYd_U_M4k3_mE_s0_4n6ry}`

## 6) Verification

I executed:

```bash
./ragebait 'texsaw{VVhYd_U_M4k3_mE_s0_4n6ry}'
```

and got:

`[SUCCESS] Flag: texsaw{VVhYd_U_M4k3_mE_s0_4n6ry}`

So the real flag is:

`texsaw{VVhYd_U_M4k3_mE_s0_4n6ry}`

