# Lego Clicker - UMass CTF Reverse Writeup

I approached this challenge like a normal Android reverse task at first: decompile the APK, identify the interesting Activities, follow the Java logic into JNI, and then recover the real flag path. That plan was correct, but I still managed to get baited several times on the way. This binary was built to waste time if I got overconfident.

The challenge description already warned that there were fake flags, and in hindsight that warning was doing a lot of work.

## Files and Initial Setup

The challenge directory only contained the APK:

```bash
/home/prasanna/umass/lego-clicker/LegoClicker_umass.apk
```

The first thing I did was unpack and decompile it with the usual Android tooling.

## Tools I Used

I used a mix of Java-side and native-side tools:

- `jadx` to decompile Java classes and inspect app logic quickly
- `apktool` to get smali, manifest, resources, and native libraries out of the APK
- `rg` to search for flag-like strings and JNI references quickly
- `objdump` to disassemble the native library in a deterministic way
- `r2` / radare2 for function discovery and spot decompilation
- Python for scripting and emulation helpers
- `unicorn` to emulate internal native functions directly when static reasoning became slower than execution

That last step was the one that broke the challenge open.

## First Pass Through the APK

After decompiling, I found the interesting Java classes under `com.example.LegoClicker`:

- `MA` looked like the main game Activity
- `RA` was the leaderboard Activity
- `FCA`, `AA`, `SA`, `LA` were auxiliary activities
- `SessionValidator` and `FlagEngine` were clearly JNI wrappers

The Android manifest made the structure clearer:

- `MA` was the launcher
- `RA` was the leaderboard screen
- `SA` was labeled something like score sync / verification
- `AA` looked like achievements / alternate reward path

At that point the likely attack surface was obvious: high score logic crossing into native code.

## Following the Java Logic

The most useful Java file was `defpackage/n0.java`.

That class had two branches depending on a constructor argument.

### Branch 0

The first branch called into `FlagEngine` and produced a string, but it also looked suspiciously puzzle-like and a little too clean. I noted it, but I did not trust it yet.

### Branch 1

The second branch was much more interesting.

It pulled a `score` extra out of the intent, required the score to be at least `1e9`, performed some custom 64-bit mixing, and then called:

```java
SessionValidator.a(longExtra, j8)
```

Then it only accepted the result if:

- it was non-null
- non-empty
- started with `UMASS{`
- ended with `}`

This immediately told me two things:

1. The challenge author wanted me to think the answer came from `SessionValidator`
2. The real flag was probably returned as a Java string from hidden native code

That made `SessionValidator` the obvious next target.

## JNI Discovery

`SessionValidator` declared three native methods:

- `refreshTileMap(long, long)`
- `syncBrickCache(long, long)`
- `validateBrickToken(long, long)`

One of the wrappers used reflection and some obfuscated string decoding to call one of these hidden names dynamically. Early on, I decoded that reflective target as `syncBrickCache`, and I treated that like a major win.

This was my first real mistake.

I assumed that because I had recovered a real method name from the Java side, I had also found the real flag path. That assumption cost me time.

## Native Library Recon

The native library was:

```text
liblegocore.so
```

I extracted it from the APK and looked at the export table. The obvious JNI exports were only for `FCA` and `FlagEngine`. The `SessionValidator` methods were not directly exported, which meant they were dynamically registered in `JNI_OnLoad`.

So I went straight to `JNI_OnLoad`.

That confirmed there were three hidden JNI methods being registered with function pointers:

- `0x210f0`
- `0x21280`
- `0x213b0`

At this point I knew the real work was in those three functions.

## First Wrong Turn: Fake Flag Recovery

One of the hidden paths eventually called a helper at `0x21ba0`. I emulated that helper and got a string out of it:

```text
BHREV{fAk3_flAG_wr0ng_s3ss10n}
```

That was clearly fake, both because the challenge explicitly warned about fake flags and because the format was wrong.

Still, this was useful.

It told me:

- I was correctly decoding real native string-building logic
- At least one entire native path existed only to waste time
- The challenge was going to reward careful validation, not first-string-wins behavior

So I kept going.

## Second Wrong Turn: Trusting `syncBrickCache`

Because I had already decoded `syncBrickCache` from the Java-side reflective call, I kept trying to force the hidden native path around `0x213b0` to be the real solution. I spent time simplifying conditions and following that function through:

- an XXTEA-like block transformation at `0x22720`
- CRC-like logic at `0x227b0`
- various small transformation helpers
- byte-table driven branching

That analysis was not wasted, but my framing was wrong. I was still treating the `SessionValidator` path as if it had to be the real one.

The challenge designer used that assumption against me.

## Third Wrong Turn: Over-trusting Static Reasoning

I also lost time trying to statically decode several helper-generated strings by hand. One especially annoying example was the internal string decoder used in `JNI_OnLoad`. My first interpretation of it was wrong because I treated it like a fixed-key XOR, when it actually used a rolling key.

That was another good lesson from this challenge: once native helper logic becomes tedious enough, emulation is often cheaper than proving every step manually.

## The Pivot: Emulate the Internal Native Helpers

At some point the best path became obvious:

Instead of continuing to reason about every branch symbolically, I would execute the internal native functions directly.

Loading the Android library normally on the host was annoying because of Android-specific linkage issues, so instead of trying to fully host the library, I used `unicorn` to emulate internal functions directly from the ELF image.

That gave me a very clean way to do exactly what I needed:

- map the ELF into memory
- initialize the global tables the same way `JNI_OnLoad` does
- stub allocator calls like `operator new` / `delete`
- call the internal string-construction helpers directly
- read their resulting C++ string objects out of emulated memory

That was the turning point.

## What the Emulation Revealed

Once I emulated the internal builders directly, the picture became clean immediately.

I recovered three relevant outputs:

```text
21ea0 -> BHREV{f4k3_fl4g_n1c3_try_d3bugger}
21f60 -> UMASS{br1ck_by_br1ck_y0u_r3ach3d_th3_t0p}
21ba0 -> BHREV{fAk3_flAG_wr0ng_s3ss10n}
```

That resolved the whole challenge structure.

There were at least two decoy flag generators in native code, and the real flag came from a different internal builder entirely.

The correct flag was:

```text
UMASS{br1ck_by_br1ck_y0u_r3ach3d_th3_t0p}
```

## How I Knew It Was Real

I did not just accept it because it looked good.

I validated it against the challenge structure:

- It matched the required `UMASS{...}` format
- It fit the Java-side `startsWith("UMASS{") && endsWith("}")` gate exactly
- It was semantically plausible for the Lego / brick theme
- The other recovered candidates were explicitly fake in both format and wording

Once I had all three outputs side by side, there was no ambiguity left.

## What I Went Wrong On

There were three main mistakes in my solve path.

### 1. I anchored too hard on the Java reflection result

Recovering `syncBrickCache` felt like progress, but I gave that result too much authority. I treated it as proof that the reflective `SessionValidator` path had to be the real one.

It was not.

### 2. I underestimated how much decoy logic existed in native code

After finding the first fake flag, I still kept trying to “repair” that path into the real solution instead of stepping back and asking whether the path itself was intentionally fake end-to-end.

### 3. I stayed in static mode too long

Once I had enough of the helper graph mapped out, emulation should have happened sooner. The challenge was designed to punish hand-decoding every transformation. Executing the real code was simply the more efficient move.

## Why the Final Approach Worked

The final approach worked because it matched the challenge’s structure.

The binary was not hard because of deep cryptography or anti-analysis. It was hard because it mixed:

- Java-side gating
- JNI indirection
- several native helper layers
- multiple fake outputs
- enough obfuscation to make over-analysis expensive

The correct response to that design was to verify behavior directly rather than trying to mentally simplify every byte transform.

## Final Flag

```text
UMASS{br1ck_by_br1ck_y0u_r3ach3d_th3_t0p}
```

## Closing Thoughts

This was a nice reminder that “I found a flag-looking string” and “I solved the challenge” are not the same thing.

The fake flags were not subtle, but they were placed behind enough legitimate-looking reverse engineering work that it was easy to spend time on them anyway. The real progress came from switching from static confidence to executable validation.

That was the difference between almost solving it and actually solving it.
