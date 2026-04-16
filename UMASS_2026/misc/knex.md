## Bespoke Puzzle Chunk Splice: how I got baited, bullied my CPU, and finally got the flag

I opened the challenge expecting some cute misc trick and immediately got humbled.

The challenge gave me three things:
- a description: “You've never seen a Bespoke Puzzle Chunk Splice (BPCS) challenge like this before!”
- `alpha = 0.45`
- one file: `lush.png`

That one line about BPCS was already suspicious. It was either a friendly hint or the author laughing at me in advance. It turned out to be both.

### First look: one PNG and a dream

I started with the obvious recon. The challenge directory was almost empty: just a giant `lush.png` around 33 MB and a Windows zone identifier sidecar. `file` said it was a normal PNG:

```bash
file lush.png
# PNG image data, 4284 x 5712, 8-bit/color RGB, non-interlaced
```

So far, so normal.

Then I tried to inspect it with ImageMagick and it immediately complained about a bad PNG header. That was the first sign this file was not behaving like a normal image. ExifTool still parsed it fine. Great. One tool said “looks good,” another said “absolutely not,” and I was already getting that CTF feeling where every object in the room is lying to me.

### The first wrong road: “maybe it’s just regular stego”

I ran the usual things:
- `strings`
- `binwalk`
- metadata checks
- chunk/header inspection

`binwalk` reported a JPEG signature somewhere inside the PNG. That smelled useful, but I’ve been baited by false-positive signatures inside compressed streams before, so I didn’t trust it yet.

At this point I still thought maybe this was:
- a normal PNG with appended data
- a malformed PNG with a hidden payload
- or a standard BPCS challenge where I just needed to decode with `alpha = 0.45`

That optimism lasted about five minutes.

### The moment the PNG started looking cursed

I wrote a quick parser to walk the PNG chunks by hand.

That’s when the file stopped pretending.

Instead of a normal stream of `IHDR`, `IDAT`, and `IEND`, the image had a bunch of invalid custom chunks named:

```text
l0l4
```

There were exactly 64 of them, each 8192 bytes long, and they were interleaved between legitimate `IDAT` chunks. That’s not an accident. That’s the author standing behind you whispering “no shortcuts.”

Even better: the PNG CRCs were valid. So this wasn’t corruption. This was handcrafted nonsense.

### Extracting the fake chunks

I concatenated all the `l0l4` chunk payloads and checked the first few bytes.

They started with:

```text
FF D8 FF E0 ... JFIF
```

So the fake chunks were actually a JPEG split into 64 pieces and smuggled inside the PNG.

Nice.

I rebuilt that JPEG and opened it. It was an image with the AM “hate” quote from *I Have No Mouth, and I Must Scream*. Very on-brand, because by this point the challenge and my computer both hated me.

The JPEG looked important, but it also looked like a clue rather than a final payload.

### Cleaning the PNG

Since the `l0l4` chunks were invalid and only there to distract or carry a second object, I stripped them out and rebuilt a clean PNG using only the legitimate chunks.

The cleaned image opened perfectly. It was a normal-looking photo of lush plants and pond growth. So now I had:
- the original cursed PNG
- the extracted JPEG clue
- the cleaned PNG vessel

At this point the challenge description started making more sense:
- “Chunk Splice” referred to the JPEG being spliced into fake PNG chunks
- “BPCS” probably referred to the actual hidden data in the cleaned carrier image

So I tried the obvious next step.

### Attempt 1: use the official BPCS tool like a civilized person

The repo in the challenge description pointed to `mobeets/bpcs`, so I used it directly.

I tested decode on both candidates with `alpha = 0.45`:
- the reconstructed JPEG
- the cleaned PNG

The JPEG decoded quickly, but the output was garbage. Just high-entropy nonsense. So the JPEG was a decoy/clue, not the real vessel.

The cleaned PNG was the real target. Unfortunately, the official decoder did what pure Python on a 4284x5712 image tends to do: it started slicing/graying the image and then basically turned my machine into a small indoor heating appliance.

The decoder wasn’t broken. It was just doing per-pixel Gray-code conversion and grid scanning in Python across a 24.5 megapixel image. My PC did not appreciate this. The fans spun up, the room temperature changed, and I had enough time to reconsider my life choices.

### Failure arc: “surely the output is just bad”

While the stock decoder struggled, I also chased a couple of dumb ideas:
- maybe the JPEG was the real BPCS vessel
- maybe the payload was directly visible in `strings`
- maybe there was an appended file after `IEND`
- maybe the weird chunk data was compressed or XOR’d

None of that paid off.

This was one of those challenges where every easy explanation is wrong in a slightly different way.

### The turning point: if the tool is too slow, replace the tool

At that point I stopped waiting for the reference implementation and read the BPCS code itself.

The algorithm in that repo is straightforward:
1. convert image channels into bitplanes
2. transform them into Canonical Gray Code
3. scan 8x8 blocks in a fixed order
4. keep blocks with complexity above `alpha`
5. split message blocks from conjugation-map blocks
6. undo conjugation
7. write the recovered bitstream

The logic was fine. The implementation was just slow for this image size.

So I wrote a fast vectorized decoder that reproduced the repo’s exact behavior:
- same Gray-code transform
- same block traversal order
- same complexity threshold
- same conjugation-map recovery

Before trusting it on the challenge, I validated it on the repo’s example image. It recovered the original sample message exactly, aside from the expected zero padding. Good enough.

Then I ran it on the cleaned `lush.png`.

### Finally: the real payload

The fast decoder finished in seconds instead of geological time.

It recovered a 44 KB payload from the image.

At first glance it still looked like garbage. But this time it was *structured* garbage:
- almost entirely printable ASCII
- no obvious file header
- no random binary entropy
- a weird restricted alphabet

That meant one more layer.

I checked the character set and it matched the basE91 alphabet almost perfectly. So the BPCS payload wasn’t the flag directly; it was a basE91-encoded second-stage blob.

I decoded that.

And immediately got:

```text
UMASS{0N3_D4Y_Y0U_W1LL_83_3MPL0Y3D}
```

At that point the challenge finally stopped lying to me.

## Why this challenge was cool

What I liked here is that it stacked multiple annoyances in a very deliberate way:

- The PNG was valid enough to fool some tools and invalid enough to break others.
- The fake `l0l4` chunks hid a fully reconstructable JPEG.
- The JPEG was meaningful enough to waste time on.
- The real flag path still required actual BPCS decoding with the given `alpha = 0.45`.
- Even after decoding BPCS, the payload had one more encoding layer.

So the solve path was basically:

1. Notice the PNG is structurally weird.
2. Parse chunks manually.
3. Extract and rebuild the JPEG from custom chunks.
4. Realize the JPEG is bait.
5. Remove fake chunks to recover the true vessel PNG.
6. BPCS-decode the cleaned PNG with `alpha = 0.45`.
7. Recognize the recovered printable blob as basE91.
8. Decode basE91.
9. Collect flag and regain emotional stability.

## Flag

```text
UMASS{0N3_D4Y_Y0U_W1LL_83_3MPL0Y3D}
```

## Closing thought

This challenge absolutely had the energy of “what if file format abuse and stego had a child and made it your problem.” I wasted time on decoys, my CPU filed a workplace complaint, and the official decoder moved like it was rendering the image one electron at a time.

Still solved it.
