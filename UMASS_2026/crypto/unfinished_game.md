# UMASS CTF `crypto_hard` Writeup

## Summary

I solved this by turning the service into a linear oracle over `GF(101)`.

Each connection gives me:

1. Eight 64-bit integers equal to `flag_word[i] XOR state[i]`.
2. A one-byte oracle from the `m` command equal to `sum(state[i] mod 101) mod 101`.

Because XOR with a known observed word can be expanded bit-by-bit, each sample becomes one linear equation in the unknown flag bits modulo `101`. With the known prefix `UMASS{`, that leaves `464` unknown bits, so after collecting enough independent sessions I can solve the whole system with modular Gaussian elimination and recover the flag.

Flag:

```text
UMASS{sparse_fourier_transforms_are_so_much_fun!fhwtftw!yayayay}
```
 files:

- `main`: the ELF binary
- `PRNG.h`: custom xoshiro-like PRNG plus many jump functions
- `solve.py`: my solver
- `samples.txt`: collected live samples

## First look

I started by identifying what the binary exposed:

```bash
file main
nm -C main
strings -n 4 main
objdump -d -Mintel main
```

The interesting symbols were:

- `load`
- `explore_right`
- `explore_left`
- `explore_middle`
- `next`
- `jump0`, `jump16`, ..., `jump496`

That immediately suggested:

- there is a PRNG state in global memory
- the game commands can move around the state space
- one of the commands probably leaks something about that state

## Reversing the binary

### `main`

`main` does three important things:

1. Fills the global `s[8]` PRNG state using `getrandom(0x40)`.
2. Calls `load()`.
3. Calls `explore_right()`.

So every connection starts with a fresh random 512-bit state.

### `load()`

`load()` is where the first leak happens.

It:

1. Creates a local 64-byte buffer.
2. Fills that buffer with random bytes first.
3. Opens `flag.txt` and reads `0x40` bytes into the same buffer.
4. For `i = 0..7`, prints:

```c
((uint64_t *)flagbuf)[i] ^ s[i]
```

as `%llu`.

So the service prints eight 64-bit words:

```text
x_i = f_i XOR s_i
```

where:

- `f_i` is the `i`-th 64-bit word of the 64-byte flag buffer
- `s_i` is the `i`-th 64-bit PRNG word
- `x_i` is what I observe on the wire

After printing them, the program writes a newline and removes `flag.txt`.

That means the only way to recover the flag is from those masked outputs plus whatever the game commands leak later.

### `explore_right()` and `explore_left()`

These are command loops. They repeatedly read one byte until they see `'e'`.

The useful commands are:

- `a`: apply a random jump to the PRNG state
- `m`: call `explore_middle()`
- `l`, `r`: recurse into the other branch
- `s`: store a pointer used for a weird one-byte write gadget

The random jumps differ by branch:

- right side: `jump0`, `jump16`, ..., `jump240`
- left side: `jump256`, `jump272`, ..., `jump496`

I checked the write gadget too because it smells like a pwn challenge at first glance:

- `s` stores a pointer to a local stack area in global `d`
- later `w` in the left branch writes one byte to `d + 8`

But in practice this did not turn into a useful control-flow primitive. The overwritten slot was not enough to get code execution cleanly, and the crypto oracle path was much stronger and much cleaner.

### `explore_middle()`

This was the real leak.

`explore_middle()` computes:

```text
(s[0] mod 101 + s[1] mod 101 + ... + s[7] mod 101) mod 101
```

and outputs that value as a raw single byte with `putc`.

So if I send `m` and then exit with `e`, I get one byte:

```text
y = sum_i (s_i mod 101) mod 101
```

Since everything is already modulo `101`, that is equivalent to:

```text
y = (s_0 + s_1 + ... + s_7) mod 101
```

## The key observation

The service lets me do this on each fresh connection:

```text
receive x_i = f_i XOR s_i   for i = 0..7
receive y   = (s_0 + ... + s_7) mod 101
```

And crucially, I can get `y` without mutating the state first by sending:

```text
me
```

Why this works:

- the program starts in `explore_right()`
- the first byte `'m'` triggers `explore_middle()`
- the next byte `'e'` causes the loop to exit

So each connection gives me one clean independent sample from a fresh random `s`.

## Turning XOR into a linear equation

At first glance,

```text
s_i = x_i XOR f_i
```

looks nonlinear in the unknown flag words `f_i`.

But bitwise XOR with a known word can be linearized bit-by-bit.

Write one 64-bit word as:

```text
f = sum_b 2^b f_b
x = sum_b 2^b x_b
```

where `f_b, x_b in {0,1}`.

Then:

```text
x_b XOR f_b = x_b + f_b - 2 x_b f_b
```

Since `x_b` is known, the coefficient of `f_b` is known too:

- if `x_b = 0`, the term is just `f_b`
- if `x_b = 1`, the term is `1 - f_b`

So modulo `101`:

```text
x XOR f
= sum_b 2^b (x_b XOR f_b)
= sum_b 2^b x_b + sum_b 2^b (1 - 2x_b) f_b
```

That is affine in the unknown bits `f_b`.

Now sum over the 8 words:

```text
y = sum_i (x_i XOR f_i) mod 101
```

Move the known constant part to the other side:

```text
y - sum_i sum_b 2^b x_{i,b}
= sum_i sum_b 2^b (1 - 2x_{i,b}) f_{i,b}
mod 101
```

This is one linear equation modulo `101` in the 512 flag bits.

## Reducing the unknowns

The flag format is known:

```text
UMASS{...}
```

I used the prefix:

```text
UMASS{
```

That gives me 6 known bytes = 48 known bits.

So instead of solving for all `512` bits, I only need to solve for:

```text
512 - 48 = 464
```

unknown bits.

Each fresh connection gives me one equation. In the ideal case I therefore need at least `464` independent samples. In practice I collected `560` so the matrix rank stabilized at full rank.

## Solver construction

I wrote `solve.py` to automate the whole attack.

### Step 1: collect live samples

For each connection, the script sends `me`, parses the eight masked words, and grabs the trailing raw oracle byte.

Conceptually each sample is:

```python
(oracle_byte, [x0, x1, ..., x7])
```

### Step 2: build the modular linear system

For each sample, for each observed bit:

- weight = `2^bit mod 101`
- coefficient is `weight` if observed bit is `0`
- coefficient is `-weight mod 101` if observed bit is `1`

Known bits from the prefix are folded directly into the right-hand side.

That produces:

```text
A * v = b mod 101
```

where:

- `A` is `num_samples x 464`
- `v` is the vector of unknown flag bits
- `b` is the right-hand side vector

### Step 3: solve with Gaussian elimination mod 101

Because `101` is prime, every nonzero pivot is invertible. So ordinary Gaussian elimination works cleanly modulo `101`.

I implemented elimination directly in the solver using NumPy arrays and `pow(pivot, -1, 101)` for modular inverses.

### Step 4: rebuild the bytes

After solving:

- each variable should be either `0` or `1`
- I insert the known prefix bits back in
- reassemble 64 bytes
- search for a `UMASS{...}` substring

## Running it

The command that recovered the flag was:

```bash
python3 /home/prasanna/umass/crypto_hard/solve.py -n 560 -j 24
```

It produced:

```text
rank=464 unknowns=464
b'UMASS{sparse_fourier_transforms_are_so_much_fun!fhwtftw!yayayay}'
UMASS{sparse_fourier_transforms_are_so_much_fun!fhwtftw!yayayay}
```

The collected samples were also saved to:

```text
/home/prasanna/umass/crypto_hard/samples.txt
```

## Why this works

The challenge tries to hide the flag with a fresh random 512-bit mask each run:

```text
masked = flag XOR state
```

Normally that would be information-theoretically fine if the state were never leaked.

But the `m` command leaks a modular sum of the same state:

```text
sum(state_i) mod 101
```

That seems tiny, but across many fresh sessions it becomes devastating because:

1. The mask is different every time.
2. The masked words are printed every time.
3. The oracle value is derived from the same hidden mask every time.
4. The relation between the masked words and the oracle can be written as a linear equation in the flag bits.

So instead of brute forcing anything, I just collect enough equations and solve.

## Dead end I checked

I did spend some time looking at the weird `s` / `w` behavior in the recursive left/right handlers because it looked like a stack write primitive:

- `s` stores a pointer into a stack frame in global memory
- `w` writes one byte to `pointer + 8`

That is exactly the sort of thing that can become a pwn bug if the stack layout lines up well.

But after checking the disassembly and where those locals were actually used, it was not the clean path. The crypto leak was much more direct and did not require fragile stack abuse.

## Final answer

```text
UMASS{sparse_fourier_transforms_are_so_much_fun!fhwtftw!yayayay}
```
