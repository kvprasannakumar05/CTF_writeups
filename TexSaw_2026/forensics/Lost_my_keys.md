# CTF Writeup: Key Search
<img width="462" height="520" alt="image" src="https://github.com/user-attachments/assets/479d8fb6-2ea4-48d6-a916-23aeb8cdfb00" />


## Challenge Description
> I can't find my original house key anywhere! Can you help me find it? Here's a picture of my keys the nanny took before they were lost. It must be hidden somewhere!

**Goal:** Get the flag formatted as `texsaw{...}`

---

## 1. Initial Assessment
I started by inspecting the provided `Temoc_keyring.png` file. It was unusually large (around 43 MB), which immediately made me suspect that additional data was hidden inside it.

Running `exiftool` initially gave me a huge hint:
```bash
$ exiftool Temoc_keyring.png
...
Warning: [minor] Trailer data after PNG IEND chunk
```

This indicated that data was appended to the end of the image file, beyond the valid PNG data chunk. 

## 2. Extracting the Hidden Data
Based on the `exiftool` warning, I ran `unzip` against the PNG file to see if the appended data was a standard ZIP archive:

```bash
$ unzip Temoc_keyring.png
Archive:  Temoc_keyring.png
warning [Temoc_keyring.png]:  39157936 extra bytes at beginning or within zipfile
  (attempting to process anyway)
   creating: key/
  inflating: key/Temoc_keyring(orig).png  
  inflating: key/where_are_my_keys.png
```

As expected, it successfully extracted a `key/` directory containing two new images: `Temoc_keyring(orig).png` (2.1 MB) and `where_are_my_keys.png` (1.8 MB).

## 3. Image Comparison
Having both an original and a seemingly modified image strongly implies that the flag is encoded within the pixel differences between the two. Both images had the exact same dimensions: 1024 x 1024 pixels.

To visualize the differences, I wrote a quick Python script using Pillow's `ImageChops` module:

```python
from PIL import Image, ImageChops

im1 = Image.open('key/Temoc_keyring(orig).png')
im2 = Image.open('key/where_are_my_keys.png')

diff = ImageChops.difference(im1, im2)
print("Difference bounds:", diff.getbbox())
```

The bounding box returned was `(0, 0, 215, 1)`, which means that the differences between the two images were contained entirely in a single 215-pixel wide line at the very top row (y = 0) of the image!

## 4. Extracting the Payload
Knowing that the hidden data spans exactly 215 pixels, I needed to extract it. I wrote a script to iterate over those first 215 pixels and compare the RGB channels of the original image against the modified one. 

For each color channel (R, G, and B), I appended a `1` to my binary string if the channel was modified, and a `0` if it remained the same:

```python
from PIL import Image

im1 = Image.open('key/Temoc_keyring(orig).png')
im2 = Image.open('key/where_are_my_keys.png')

p1 = im1.load()
p2 = im2.load()

binary_str = ""
for x in range(215):
    r1, g1, b1 = p1[x, 0]
    r2, g2, b2 = p2[x, 0]
    
    # 1 if difference is present, 0 normally
    binary_str += "1" if r1 != r2 else "0"
    binary_str += "1" if g1 != g2 else "0"
    binary_str += "1" if b1 != b2 else "0"

print(f"Extracted {len(binary_str)} bits")
```

This produced a 645-bit binary string. I grouped these bits into 8-bit bytes to see what they represented. 

## 5. Decoding the Flag
When observing the hex representation of the first few bytes (`e8 ca f0 e6 c2...`), they didn't map to valid ASCII characters out-of-the-box. However, I noticed a distinct pattern: every valid byte started with a `1` and ended with a `0`. 

This meant the bytes were bit-shifted. By shifting each byte to the right by 1 bit (`byte >> 1`), the binary data magically aligned to valid ASCII:

- `0xe8` -> `1110 1000` >> 1 = `0111 0100` -> `t`
- `0xca` -> `1100 1010` >> 1 = `0110 0101` -> `e`
- `0xf0` -> `1111 0000` >> 1 = `0111 1000` -> `x`

I added the final transformation to my decoding logic:
```python
def bin_to_bytes(b_str):
    b = bytearray()
    for i in range(0, len(b_str)-7, 8):
        byte_val = int(b_str[i:i+8], 2)
        # Shift right by 1 to decode the ASCII characters
        b.append(byte_val >> 1)
    return b

print("Flag:", bin_to_bytes(binary_str).decode('ascii', errors='ignore'))
```

Running the final extraction script printed the flag perfectly!

**Flag:** `texsaw{you_found_me_at_key}`
