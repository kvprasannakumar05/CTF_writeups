# The Accursed Lego Bin — CTF Writeup

**Event:** UMassCTF  
**Category:** Crypto  
**Difficulty:** Easy  
**Points:** 100  
**Flag:** `UMASS{tH4Nk5_f0R_uN5CR4m8L1nG_mY_M3554g3}`

---

## First Look

When I opened this challenge, I saw the description: _"I dropped my message into the bin of Legos. It's all scrambled up now. Please help."_

I got two files — `encoder.py` and `output.txt`. I always start by reading the encoder before touching the output, because understanding _how_ something was broken is the only way to figure out how to fix it.

---

## Okay, what is this thing even doing?

I opened `encoder.py` and read through it slowly.

The script does a few things:

**First**, it encrypts the string `"I_LOVE_RNG"` using RSA with a public exponent `e = 7` and two freshly generated 2048-bit primes. The ciphertext from that becomes the `seed`.

python

```python
e = 7

def RSA_enc(plain_text):
    p, q = getPrime(2048), getPrime(2048)
    n = p * q
    plain_num = int.from_bytes(plain_text.encode(), "big")
    ciphertext = pow(plain_num, e, n)
    return n, ciphertext
```

**Then**, it takes the flag, converts every character into its binary representation (8 bits each), and shuffles those bits 10 times using Python's `random.shuffle()`. Each shuffle round uses a different seed — `seed * 1`, `seed * 2`, all the way up to `seed * 10`.

python

```python
for i in range(10):
    random.seed(seed * (i + 1))
    random.shuffle(flag_bits)
```

**Finally**, it writes the shuffled bits as a hex string to `output.txt`, along with `enc_seed` which is... the seed, encrypted again with RSA.

So what I have in `output.txt` is:

- The seed, double-wrapped in RSA
- The flag, with all its bits jumbled up

The idea is: you can't unshuffle the flag if you don't know the seed, and you can't get the seed without breaking RSA. Sounds solid — until you actually look at the numbers.

---

## The moment I saw it

Here's the thing that broke the whole scheme open.

The message being encrypted is `"I_LOVE_RNG"`. That's 10 characters. 10 bytes. **80 bits.**

The RSA modulus `n` is the product of two 2048-bit primes. That makes it roughly **4096 bits** long.

RSA encryption computes: `ciphertext = m^e mod n`

With `e = 7`, that means: `seed = m^7 mod n`

But wait — `m` is only 80 bits. So `m^7` is at most `80 × 7 = 560 bits`. And `n` is 4096 bits.

**560 bits is nowhere near 4096 bits.**

That means `m^7` never even reaches `n`. The `mod n` part does absolutely nothing. The result is just `m^7`, plain and simple.

So I don't need to factor `n`. I don't need the private key. I don't need anything secret. I just compute:

python

```python
m = int.from_bytes(b"I_LOVE_RNG", "big")
seed = m ** 7
```

And I have the seed. Just like that.

---

## Unshuffling the flag

Now that I had the seed, the rest was about reversing the shuffles.

Python's `random.shuffle()` uses a fixed algorithm under the hood (Fisher-Yates). If you give it the same seed, it does the exact same shuffle every single time. That means it's completely predictable — and reversible.

To reverse a shuffle, I replayed it on a list of indices `[0, 1, 2, ..., n]` to figure out the permutation, then put the bits back where they came from:

python

```python
def invert_shuffle(bits, rng_seed):
    n = len(bits)
    indices = list(range(n))
    rng = random.Random(rng_seed)
    rng.shuffle(indices)
    result = [''] * n
    for i, orig in enumerate(indices):
        result[orig] = bits[i]
    return result
```

Since the original code applied 10 shuffles in order from `i = 0` to `i = 9`, I undid them in reverse — from `i = 9` back to `i = 0`:

python

```python
current_bits = flag_bits[:]
for i in reversed(range(10)):
    current_bits = invert_shuffle(current_bits, seed * (i + 1))
```

Then I just grouped the bits back into bytes and converted to text:

python

```python
def bits_to_str(bits):
    chars = []
    for i in range(0, len(bits), 8):
        byte = bits[i:i+8]
        char = int(''.join(byte), 2)
        chars.append(chr(char))
    return ''.join(chars)
```

---

## And there it was

```
UMASS{tH4Nk5_f0R_uN5CR4m8L1nG_mY_M3554g3}
```

---

## Why did this even work?

The whole challenge was built around one assumption: the seed is secret because it's protected by RSA.

But that assumption only holds if the plaintext is large enough that `m^e` wraps around the modulus. Here, the plaintext was tiny — and hardcoded. Anyone reading the source code knows it's `"I_LOVE_RNG"`. And because `m^7` was smaller than `n`, the modular reduction never kicked in, making the "encryption" completely pointless.

This is a well-known RSA mistake called the **small plaintext, small exponent** vulnerability. The fix is simple: always pad your message before encrypting. With proper padding, even a short message gets expanded to the full size of the modulus, and this attack doesn't work.

But without padding? You're basically just computing a power of a number anyone can see. Not great for a secret seed.

---

## What I took away from this

- Always check the size of your RSA plaintext relative to your modulus. If `m^e < n`, you have a problem.
- Hardcoded known plaintexts are a gift to attackers.
- `random.shuffle()` is not random if the seed is known — it's just a deterministic permutation.
- Read the encoder slowly. The bug is almost always hiding in plain sight.
