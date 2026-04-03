# brokenquest Writeup

I approached this as a local reverse challenge and started by looking at the files in the challenge directory. There were only two interesting files:

- `brokenquest`
- `libc.so.6`

The binary was a 64-bit PIE ELF and, importantly, it was not stripped. That made the reversing path much easier because the important functions were still named.

## Initial Recon

The first thing I checked was the binary metadata and strings. From `strings` and `readelf`, I immediately learned a few important things:

- The program is menu-driven and exposes actions `0` through `8`.
- The prompt says I need to "Set all quest objective flags and turn in the quest to get the real flag."
- The binary contains helpful function names like `turn_in`, `handle_flag`, `calc_val`, `transform`, and the state mutation helpers.

The menu actions map to these state-manipulation functions:

- `1`: `reset`
- `2`: `rotate`
- `3`: `increment`
- `4`: `add_sub`
- `5`: `modulo`
- `6`: `swap`
- `7`: `bitshift`
- `8`: `flip_sign`

So the challenge is centered around an 8-element integer state array, and the binary wants me to mutate that state into the required quest objective values.

## Understanding the Main State

In `main`, I found two important arrays on the stack:

- The current quest state, initialized to eight zeroes.
- The objective state, initialized to:

```text
[2, 6, -4, 6, 0, 4, -3, 1]
```

That tells me the intended solve is to transform the current state from:

```text
[0, 0, 0, 0, 0, 0, 0, 0]
```

into:

```text
[2, 6, -4, 6, 0, 4, -3, 1]
```

before selecting action `0` to turn in the quest.

## Reversing the Actions

I reversed each of the mutation functions from the disassembly:

### `2: Rotate Pillars` -> `rotate`

This rotates the whole 8-element state right by one.

```text
[a0,a1,a2,a3,a4,a5,a6,a7] -> [a7,a0,a1,a2,a3,a4,a5,a6]
```

### `3: Increase Heat` -> `increment`

This increments element `0` and element `4`.

```text
a[0] += 1
a[4] += 1
```

### `4: Move Gold Coins` -> `add_sub`

This adds `3` to element `0` and subtracts `2` from element `3`.

```text
a[0] += 3
a[3] -= 2
```

### `5: Swing Sword` -> `modulo`

This transforms two positions:

```text
a[0] = trunc(a[0] / 5)
a[6] = a[6] % 5
```

### `6: Swap Gems` -> `swap`

This swaps element `0` and element `5`.

```text
swap(a[0], a[5])
```

### `7: Shift Sand Piles` -> `bitshift`

This doubles element `1` and arithmetic-right-shifts element `7`.

```text
a[1] *= 2
a[7] >>= 1
```

### `8: Reverse Polarity` -> `flip_sign`

This negates element `0` and element `2`.

```text
a[0] = -a[0]
a[2] = -a[2]
```

At this point, the intended path is clear in principle: I need to construct the target state using these transformations.

## Finding the Bug

The faster route came from reversing `turn_in`.

The key check is:

```c
memcmp(current_state, objective_state, 8)
```

That is the bug.

Both arrays contain `int[8]`, so comparing only `8` bytes means the binary checks only the first **two integers**, not all eight.

That means I do **not** need to satisfy the full objective state. I only need:

```text
current_state[0] == 2
current_state[1] == 6
```

The remaining six integers can be wrong, and the program will still accept the quest turn-in.

This directly matches the challenge note saying there are two ways to solve it.

## Exploit Solve

I searched for a short action sequence that produces a state beginning with `[2, 6, ...]`.

One valid sequence is:

```text
4 2 3 3 7 0
```

I verified the state evolution manually:

### Start

```text
[0,0,0,0,0,0,0,0]
```

### Action `4`

`add_sub`

```text
[3,0,0,-2,0,0,0,0]
```

### Action `2`

`rotate`

```text
[0,3,0,0,-2,0,0,0]
```

### Action `3`

`increment`

```text
[1,3,0,0,-1,0,0,0]
```

### Action `3`

`increment`

```text
[2,3,0,0,0,0,0,0]
```

### Action `7`

`bitshift`

```text
[2,6,0,0,0,0,0,0]
```

### Action `0`

Turn in quest.

Since only the first two integers are checked, the program accepts the turn-in.

## Why the Direct Exploit Does Not Immediately Print the Real Flag

Even though the exploit passes `turn_in`, the printed reward from the unmodified binary is garbage rather than a valid `texsaw{...}` string.

That happens because after the buggy `memcmp`, the program still calls `handle_flag(current_state)`.

So the turn-in condition is broken, but the flag generation logic still uses the actual current state, not the target objective state. Passing the quest check is therefore not enough by itself to recover the real flag text.

## Recovering the Real Flag

I used the bug to prove the alternate solve, then patched a **copy** of the binary to force the success path to generate the reward from the objective array instead of the current array.

In `turn_in`, the relevant instructions are effectively:

```asm
mov rax, [rbp-0x8]   ; current_state
mov rdi, rax
call handle_flag
```

I changed the byte at file offset `0x1f34` so that the pointer passed to `handle_flag` becomes the objective array instead of the current state. I did this on a copied file, not the original challenge binary.

After that, I reused the short bug-based solve:

```text
4 2 3 3 7 0
```

and the patched binary printed the real flag:

```text
texsaw{1t_ju5t_work5_m0r3_l1k3_!t_d0e5nt_w0rk}
```

## Final Flag

```text
texsaw{1t_ju5t_work5_m0r3_l1k3_!t_d0e5nt_w0rk}
```

## Summary

There are two meaningful takeaways from this challenge:

1. The intended design is to transform the 8-integer quest state into:

```text
[2, 6, -4, 6, 0, 4, -3, 1]
```

2. The unintended or alternate solve is the broken comparison in `turn_in`:

```c
memcmp(..., ..., 8)
```

which only validates the first two integers.

That gave me the short bypass sequence:

```text
4 2 3 3 7 0
```

and from there I patched a copy of the binary to make the program reveal the actual flag tied to the intended objective state.
