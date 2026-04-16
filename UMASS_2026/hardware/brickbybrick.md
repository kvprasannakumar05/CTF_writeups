# Brick by Brick - Hardware CTF Writeup 

## Challenge
**Name:** Brick by Brick  
**Category:** Hardware / Signal Analysis  
**Description:** We intercepted raw data from a custom LEGO controller. Recover the message.  
**Hints:**
- Communication type is common for older I/O connectors
- Every message starts with a falling edge
- UART, 8N1 (need baud rate)

Flag format: `UMASS{...}`

---

## My Goal
Take the provided raw capture (`code.csv`), decode the serial communication, and recover the flag.

---

## Files I Had
In the challenge folder, I found:
- `code.csv`

No logic analyzer project file, no screenshots, no scripts. So I decoded straight from the CSV.

---

## Step 1: Understand the CSV Format
I checked the first rows and found two columns:
- `timestamp`
- `logic_level`

So each row is one sampled digital value at a specific time.

I measured sample interval from timestamps:
- `dt ~= 2.996075e-05 s`
- sample rate `fs ~= 33377 Hz`

So this is a uniformly sampled digital waveform.

---

## Step 2: Look for Timing Structure
At first, I tried direct UART decoding by testing common baud rates (9600, 19200, etc.) and framing assumptions. That produced noisy/semi-readable output but not a clean flag.

Then I looked at bit-level periodicity and run lengths. A strong periodic pattern appeared every **15 samples**.

When I printed raw bits in windows, I saw repeated blocks like:
- `111110xxxxxxxxx`

This looked highly structured and stable.

---

## Step 3: Identify the True Framing
I tested chunking the full waveform into 15-bit symbols with different phase offsets.

Result:
- At **phase 0**, every block matched prefix `111110`.
- This held for all 3799 blocks.

So each frame is exactly 15 bits:
1. 6-bit prefix: `111110`
2. 8-bit payload
3. 1-bit stop bit (`1`)

That means the effective symbol arrangement is UART-like payload transport already packed into fixed-size 15-bit blocks.

---

## Step 4: Extract Bytes
For each 15-bit block, I:
- ignored prefix `111110`
- took bits 6..13 as data bits
- interpreted those 8 bits as little-endian UART data (`LSB first`)
- verified last bit was stop bit `1`

This produced 3799 bytes of clean plaintext immediately.

Decoded output started with Linux boot logs, e.g.:
- `Linux version ...`
- `BIOS-e820 ...`

So decode logic was definitely correct.

---

## Step 5: Hunt the Flag in Decoded Text
I searched decoded lines for keywords like `flag`, `umass`, `ctf`.

I found this line:

`[    0.037500] secretflag: 554d4153537b553452375f31355f3768335f623335372c5f72316768373f7d`

That value is hex.

I converted hex to ASCII:

`UMASS{U4R7_15_7h3_b357,_r1gh7?}`

---

## Final Flag
`UMASS{U4R7_15_7h3_b357,_r1gh7?}`

---

## Why This Worked
The key insight was that trying random UART baud directly on the sampled waveform was misleading. The signal had a stronger fixed framing layer: 15-bit packets with a constant 6-bit marker.

Once I treated it as framed symbols and extracted the payload bytes, plaintext appeared cleanly.

---

## Repro Script (What I Effectively Used)
```python
import csv

lv = [int(r['logic_level']) for r in csv.DictReader(open('code.csv'))]
vals = []

for i in range(0, len(lv)-15, 15):
    blk = lv[i:i+15]
    if blk[:6] != [1,1,1,1,1,0]:
        continue
    if blk[14] != 1:
        continue

    data = blk[6:14]  # 8 bits
    val = sum(bit << k for k, bit in enumerate(data))  # LSB-first
    vals.append(val)

text = bytes(vals).decode('utf-8', 'replace')
print(text)
```

Then grep for `secretflag`, convert hex to bytes, done.

---

## Takeaway
Pattern recognition on raw logic captures matters more than forcing a decoder early. Once I found the fixed start marker and exact frame length, extraction became straightforward.
