# Abyss Writeup

Challenge: `the abyss`  
Category: `pwn`  
Author: `shura356`  
Flag: `apoorvctf{th1s_4by55_truly_d03s_5t4r3_b4ck}`

## Summary

This binary exposes a command interface around two slab-managed object types:

- `dive` objects
- `leviathan` objects created by `BEACON`

The intended gimmick is that the program has:

- a main command loop
- a mesopelagic worker thread that processes `FLUSH`
- a benthic worker thread that processes `ABYSS`
- internal `io_uring` usage to open/read the flag

The bug is a stale-pointer / use-after-free style issue:

- `FLUSH` frees all existing `dive` objects asynchronously
- the registry entries in `g_dive_reg` are not cleared
- later `BEACON` allocations can reuse the same slab chunks
- the old `dive` ids still point to memory that is now a live `leviathan`

That gives us an overlap primitive:

- old `WRITE <dive_id> ...` writes into a reused chunk
- if that chunk now holds a `leviathan`, we control its note buffer
- `ABYSS <levi_id>` sends that note to the benthic thread as a raw `io_uring_sqe`

So the exploit is:

1. create two dives and leak one address
2. `FLUSH` them
3. wait for the worker to actually free them
4. create enough leviathans so allocation falls back into the freed dive slab
5. use stale dive ids to overwrite live leviathan note contents
6. forge an `IORING_OP_OPENAT` SQE targeting `/flag.txt`
7. call `ABYSS` on the overlapped leviathan
8. the benthic thread opens the flag and prints it

## Initial Recon

The binary is:

- 64-bit ELF
- PIE
- NX
- Canary
- Full RELRO
- not stripped

Useful strings immediately reveal the command surface:

- `DIVE`
- `DESCEND`
- `WRITE`
- `POP`
- `FLUSH`
- `STATUS`
- `BEACON`
- `ABYSS`
- `ECHO`
- `HELP`
- `QUIT`

The binary also contains the literal string `/flag.txt` in `.data`.

## Important Data Structures

From static analysis, the relevant object layouts are effectively:

### Dive

Size: `0x60`

Fields used by the program:

- `+0x00` depth/id-related data
- `+0x04` flags/depth field
- `+0x08` 64-byte note buffer
- `+0x48` timestamp
- `+0x50` encoded next pointer for free/request stacks
- `+0x58` tag/canary-like marker

### Leviathan

Also slab-sized so it can overlap with freed dive chunks.

Its note buffer is what eventually gets handed to the benthic thread and submitted as an `io_uring_sqe`.

## Command Semantics

### `DIVE <depth>`

Allocates a dive from the dive slab and stores it in `g_dive_reg[0..15]`.

It prints:

```text
DIVING id=%d addr=0x%lx depth=%u
```

That leak is extremely useful because the binary is PIE.

The first dive chunk is at a fixed offset from PIE base:

- `dive_slab` is at PIE offset `0x6920`

So:

```text
pie_base = leaked_dive_addr - 0x6920
```

### `WRITE <id> <len> <hex>`

Decodes hex and copies bytes into the dive note buffer at `obj + 0x8`.

The maximum range allows writing across the 64-byte note region.

This command becomes powerful once a stale dive id points to memory that has been recycled as a leviathan.

### `FLUSH`

This is the core bug trigger.

The main thread sends a request to the mesopelagic worker. That worker:

- pops entries from `g_request_stack`
- frees those dive chunks back into the dive free list

But the program does **not** clear the corresponding entries in `g_dive_reg`.

That means the registry still contains dangling pointers.

There is also a worker-side delay before reclamation fully happens, so the exploit must wait before reallocation.

### `BEACON <id>`

Allocates a leviathan and stores it in `g_levi_reg[0..23]`.

It normally allocates from `levi_slab`, but once that slab is exhausted, it falls back into the dive slab.

That is the overlap trigger.

Observed behavior:

- `BEACON 0..15` use the dedicated leviathan slab
- `BEACON 16` and `BEACON 17` land exactly on the freed dive chunks after a successful `FLUSH`

### `ABYSS <id>`

This sends the selected leviathan to the benthic thread.

The benthic thread:

- reads the 64-byte note buffer as raw bytes
- copies it directly into an `io_uring_sqe`
- submits it
- if the opcode is `0x12` (`IORING_OP_OPENAT`), it then attempts to read from the returned fd and prints:

```text
FLAG: ...
```

So `ABYSS` is effectively an `io_uring_sqe` execution gadget.

## Root Cause

The vulnerability is not a classic stack overflow. It is a lifetime bug:

- `FLUSH` returns dive chunks to the allocator
- stale pointers remain in `g_dive_reg`
- later `BEACON` reuses those same chunks for a different type
- stale `WRITE` operations mutate the new object

This is a type-confusion / stale-handle reuse issue on top of slab recycling.

## Why the Overlap Works

After:

```text
DIVE 1
DIVE 1
FLUSH
```

and waiting for the worker to finish, the two original dive chunks are free.

Then:

```text
BEACON 0
...
BEACON 17
```

reuses those exact freed addresses for:

- leviathan 16
- leviathan 17

That was verified during exploitation:

- original dive0 leaked at `...3920`
- original dive1 leaked at `...3980`
- later `BEACON 16` reused `...3920`
- later `BEACON 17` reused `...3980`

So stale `WRITE 1 ...` no longer writes to a dive. It writes into live leviathan 17 memory.

## Building the Final Primitive

We need a valid 64-byte `io_uring_sqe` for:

- opcode: `IORING_OP_OPENAT` (`0x12`)
- fd: `AT_FDCWD` (`-100`)
- addr: pointer to `/flag.txt`
- open flags: `O_RDONLY`

The nice detail is that `/flag.txt` is already stored in `.data` at PIE offset `0x6010`.

So from the first dive leak:

```text
flag_path = pie_base + 0x6010
```

That means we do not need a second overlapped chunk to store the path string manually.

We only need to overwrite one live leviathan note with a forged SQE.

## SQE Layout

The program copies a full 64-byte SQE buffer.

The working packed layout used was:

```python
struct.pack(
    "<BBHiQQIIQHHIQQ",
    0x12,      # opcode = IORING_OP_OPENAT
    0,         # flags
    0,         # ioprio
    -100,      # fd = AT_FDCWD
    0,         # off/addr2
    flag_path, # addr = pathname
    0,         # len = mode
    0,         # open_flags = O_RDONLY
    0,         # user_data
    0,         # buf_index
    0,         # personality
    0,         # splice_fd_in
    0,         # addr3
    0,         # pad
)
```

## Exploit Steps

### 1. Allocate two dives

```text
DIVE 1
DIVE 1
```

Use the first leaked address to recover PIE base:

```text
pie = dive0_addr - 0x6920
flag_path = pie + 0x6010
```

### 2. Free them with `FLUSH`

```text
FLUSH
```

Then wait long enough for the worker thread to finish reclaiming the requests.

This delay matters. If you reallocate too early, the overlap will not happen.

### 3. Allocate 18 leviathans

```text
BEACON 0
...
BEACON 17
```

Now:

- `BEACON 16` reuses old dive0
- `BEACON 17` reuses old dive1

### 4. Overwrite live leviathan 17 through stale dive id 1

Using the stale registry entry for dive id `1`:

```text
WRITE 1 64 <forged_sqe_hex>
```

That writes directly into leviathan 17’s note buffer.

### 5. Trigger execution

```text
ABYSS 17
```

The benthic thread submits our forged `OPENAT` SQE, opens `/flag.txt`, reads from the returned fd, and prints the flag.

## Remote Solve Transcript

The remote exploit produced:

```text
FLAG: apoorvctf{th1s_4by55_truly_d03s_5t4r3_b4ck}
```

## Final Flag

```text
apoorvctf{th1s_4by55_truly_d03s_5t4r3_b4ck}
```

## Solver

A solver script was saved as:

- `solve.py`

Core logic:

- leak PIE from first `DIVE`
- compute `/flag.txt` pointer from PIE
- `FLUSH` and wait
- create 18 `BEACON`s
- overwrite overlapped leviathan 17 via stale dive id 1
- trigger `ABYSS 17`

## Takeaways

This challenge is a good example of a modern pwnable where the bug is not:

- stack smashing
- GOT overwrite
- trivial heap corruption

Instead, the exploitation path is:

- asynchronous lifetime bug
- stale registry entries
- slab reuse across object types
- repurposing an internal subsystem (`io_uring`) as the code execution primitive

The interesting part is that the binary already contains the logic needed to read the flag. We do not need shellcode, ROP, or libc control. We only need to steer the program into submitting the `io_uring` request we want.
