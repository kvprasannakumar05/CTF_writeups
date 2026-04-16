# Ninja Nerds Writeup

I approached this as a quick image-forensics / stego triage challenge. The description was:

> where are your little ninja-nerds?

The flag format was `UMASS{...}`, and the challenge directory contained a single PNG. That immediately suggested one of the usual image-forensics paths:

- metadata
- appended payloads
- PNG chunk abuse
- LSB steganography

## Files Given

I started by listing the challenge directory.

### Command

```bash
ls -la /home/prasanna/umass/ninja-nerds
```

### Output

```text
total 264
drwxr-xr-x 2 prasanna prasanna   4096 Apr 11 11:24 .
drwxr-xr-x 9 prasanna prasanna   4096 Apr 11 11:24 ..
-rw-r--r-- 1 prasanna prasanna 256589 Apr 11 11:24 challenge.png
-rw-r--r-- 1 prasanna prasanna     25 Apr 11 11:24 challenge.png:Zone.Identifier
```

There were only two files:

- `challenge.png`
- `challenge.png:Zone.Identifier`

The `Zone.Identifier` file was just a Windows metadata artifact and not part of the challenge logic.

## File Type Identification

I checked what kind of file I was dealing with.

### Command

```bash
file /home/prasanna/umass/ninja-nerds/*
```

### Output

```text
/home/prasanna/umass/ninja-nerds/challenge.png:                 PNG image data, 640 x 360, 8-bit/color RGB, non-interlaced
/home/prasanna/umass/ninja-nerds/challenge.png:Zone.Identifier: ASCII text, with CRLF line terminators
```

So the challenge artifact was a normal RGB PNG, `640x360`.

## First-Pass Triage

My usual first pass on images is:

1. `strings`
2. metadata extraction
3. `binwalk`

That quickly rules out lazy embedding tricks.

### Command

```bash
strings -n 6 /home/prasanna/umass/ninja-nerds/challenge.png | head -n 300
```

### Output

The output was mostly random noise-looking fragments. A few example lines:

```text
""***r
nOxq[~
Z7R77-W7q
HLHCh6
iUfh[]
pU8] Dj
...
```

Nothing in `strings` looked like a flag or clean plaintext.

### Command

```bash
exiftool /home/prasanna/umass/ninja-nerds/challenge.png
```

### Output

```text
ExifTool Version Number         : 13.50
File Name                       : challenge.png
Directory                       : /home/prasanna/umass/ninja-nerds
File Size                       : 257 kB
File Modification Date/Time     : 2026:04:11 11:24:27+05:30
File Access Date/Time           : 2026:04:11 11:25:35+05:30
File Inode Change Date/Time     : 2026:04:11 11:24:38+05:30
File Permissions                : -rw-r--r--
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 640
Image Height                    : 360
Bit Depth                       : 8
Color Type                      : RGB
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Image Size                      : 640x360
Megapixels                      : 0.230
```

No suspicious metadata. No comments, author fields, text chunks, or embedded hints.

### Command

```bash
binwalk /home/prasanna/umass/ninja-nerds/challenge.png
```

### Output

```text
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 640 x 360, 8-bit/color RGB, non-interlaced
41            0x29            Zlib compressed data, default compression
246662        0x3C386         JBOOT STAG header, image id: 3, timestamp 0xFB61A746, image size: 2642431325 bytes, image JBOOT checksum: 0xAFD9, header JBOOT checksum: 0xC76F
```

The `JBOOT STAG` hit near the end looked like a false positive rather than a meaningful appended payload. At this point, nothing obvious was exposed by metadata or file carving.

## Looking at the Image

I opened the PNG to see whether the visual content hinted at a hidden layer or a challenge theme.

### Observation

It showed a Ninjago-style scene with two LEGO ninja characters fighting in a purple crystal environment.

That did not directly reveal the flag, but it reinforced that this was probably a “hidden in the image” stego challenge rather than a metadata puzzle.

## Running `zsteg`

Since the file was a PNG, the next tool was `zsteg`.

### Command

```bash
zsteg /home/prasanna/umass/ninja-nerds/challenge.png
```

### Output

```text
imagedata           .. text: ";98?5:Xn]"
chunk:0:IHDR        .. file: Adobe Photoshop Color swatch, version 0, 640 colors; 1st RGB space (0), w 0x168, x 0x802, y 0, z 0; 2nd RGB space (0), w 0, x 0, y 0, z 0x1
b1,r,lsb,xy         ..
b1,r,msb,xy         ..
b1,g,lsb,xy         ..
b1,g,msb,xy         ..
b1,b,lsb,xy         ..
b1,b,msb,xy         ..
b1,rgb,lsb,xy       ..
b1,rgb,msb,xy       ..
b1,bgr,lsb,xy       ..
b1,bgr,msb,xy       ..
b2,r,lsb,xy         ..
b2,r,msb,xy         ..
b2,g,lsb,xy         .. text: "\"3UUUKhUU"
b2,g,msb,xy         ..
b2,b,lsb,xy         .. text: ["U" repeated 9 times]
b2,b,msb,xy         ..
b3,r,lsb,xy         ..
b3,r,msb,xy         ..
b3,g,lsb,xy         ..
b3,g,msb,xy         ..
b3,b,lsb,xy         ..
b3,b,msb,xy         ..
b3,rgb,lsb,xy       ..
b3,rgb,msb,xy       ..
b3,bgr,lsb,xy       ..
b3,bgr,msb,xy       ..
b4,r,lsb,xy         .. text: ["f" repeated 8 times]
b4,r,msb,xy         .. text: "ffffffffa"
b4,g,lsb,xy         .. text: "DDDUDUUwUUDDDDDUUwfww"
b4,g,msb,xy         .. text: ["w" repeated 8 times]
b4,b,lsb,xy         .. text: "bd\"DfffDh"
b4,b,msb,xy         .. text: "F&D\"fff\""
b4,rgb,lsb,xy       .. text: "sg4s'2sG4"
b4,rgb,msb,xy       ..
b4,bgr,lsb,xy       .. text: "ct7#r7Ct7b"
b4,bgr,msb,xy       ..
```

`zsteg` did not give me the full flag directly, but it did suggest that low-order bits, especially in the blue channel, were interesting. The repeated `U` values were a good hint that a structured payload might be sitting in one channel’s LSB stream.

## Bit-Plane Inspection

I then generated bit-plane images for each RGB channel to see whether any single bit plane showed hidden text or shapes.

### Command

```bash
python3 - <<'PY'
from PIL import Image
img=Image.open('/home/prasanna/umass/ninja-nerds/challenge.png').convert('RGB')
for plane in range(8):
    for ci,name in enumerate('rgb'):
        out=Image.new('L', img.size)
        pix=img.load(); op=out.load()
        for y in range(img.height):
            for x in range(img.width):
                v=(pix[x,y][ci]>>plane)&1
                op[x,y]=255*v
        path=f'/tmp/{name}_bit{plane}.png'
        out.save(path)
        print(path)
PY
```

### Output

```text
/tmp/r_bit0.png
/tmp/g_bit0.png
/tmp/b_bit0.png
/tmp/r_bit1.png
/tmp/g_bit1.png
/tmp/b_bit1.png
/tmp/r_bit2.png
/tmp/g_bit2.png
/tmp/b_bit2.png
/tmp/r_bit3.png
/tmp/g_bit3.png
/tmp/b_bit3.png
/tmp/r_bit4.png
/tmp/g_bit4.png
/tmp/b_bit4.png
/tmp/r_bit5.png
/tmp/g_bit5.png
/tmp/b_bit5.png
/tmp/r_bit6.png
/tmp/g_bit6.png
/tmp/b_bit6.png
/tmp/r_bit7.png
/tmp/g_bit7.png
/tmp/b_bit7.png
```

I also built low-bit composite images:

### Command

```bash
python3 - <<'PY'
from PIL import Image
img=Image.open('/home/prasanna/umass/ninja-nerds/challenge.png').convert('RGB')
combos=[('rgb_lsb1',1),('rgb_lsb2',2)]
for name,bits in combos:
    out=Image.new('RGB', img.size)
    p=img.load(); o=out.load()
    for y in range(img.height):
        for x in range(img.width):
            vals=[]
            for c in range(3):
                v=p[x,y][c] & ((1<<bits)-1)
                vals.append(int(v*255/((1<<bits)-1)))
            o[x,y]=tuple(vals)
    path=f'/tmp/{name}.png'
    out.save(path)
    print(path)
PY
```

### Output

```text
/tmp/rgb_lsb1.png
/tmp/rgb_lsb2.png
```

Visually, those composite images looked mostly noisy, not like a hidden QR code or a second embedded picture. That pushed me toward a direct bitstream extraction approach instead of purely visual stego.

## Extracting LSB Bitstreams Directly

At this point, the most likely explanation was:

- the payload was stored in LSBs
- not as a visible hidden image
- but as raw text serialized through one color channel

So I wrote a quick Python script to:

1. read the image row by row
2. extract the LSBs from different channel orders
3. repack them into bytes
4. search the resulting byte stream for the known prefix `UMASS`

### Command

```bash
python3 - <<'PY'
from PIL import Image
img=Image.open('/home/prasanna/umass/ninja-nerds/challenge.png').convert('RGB')
for mode in ['rgb','bgr','r','g','b']:
    bits=[]
    for y in range(img.height):
        for x in range(img.width):
            px=img.getpixel((x,y))
            chans={'r':[px[0]],'g':[px[1]],'b':[px[2]],'rgb':[px[0],px[1],px[2]],'bgr':[px[2],px[1],px[0]]}[mode]
            for v in chans:
                bits.append(str(v&1))
    data=bytearray()
    for i in range(0,len(bits)-7,8):
        data.append(int(''.join(bits[i:i+8]),2))
    s=data.decode('latin1','ignore')
    if 'UMASS' in s:
        print('found direct',mode,s[s.index('UMASS'):s.index('UMASS')+80])
    else:
        print('no direct',mode)
PY
```

### Output

```text
no direct rgb
no direct bgr
no direct r
no direct g
found direct b UMASS{perfectly-hidden-ready-to-strike}...
```

That was the solve.

The flag appeared directly in the **blue-channel LSB stream**:

`UMASS{perfectly-hidden-ready-to-strike}`

## Why This Worked

The image was carrying hidden text in the least significant bit of the **blue channel**, serialized in row-major pixel order. Most generic metadata and carving tools would not reveal that automatically, and `zsteg` only hinted that the blue low bits contained structure.

Once I repacked those blue-channel LSBs into bytes and searched for the flag prefix, the flag appeared immediately.

## Final Flag

`UMASS{perfectly-hidden-ready-to-strike}`

## Short Solve Summary

I triaged the PNG with `file`, `strings`, `exiftool`, and `binwalk`, found no obvious metadata or appended payload, used `zsteg` to confirm suspicious low-bit structure, then extracted row-wise LSB bitstreams from the image channels with Python. The flag was embedded directly in the **blue channel’s LSB data**, yielding:

`UMASS{perfectly-hidden-ready-to-strike}`
