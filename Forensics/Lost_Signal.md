# Lost Signal - CTF Writeup

**CTF:** L3m0nCTF 2025  
**Category:** Forensics  
---

## Challenge Overview

**File Provided:** `lost_signal.tar.gz`  
**Clue:** `Seed = 739391`

---
<img width="434" height="596" alt="image" src="https://github.com/user-attachments/assets/38a33ffa-65a1-4689-9515-92bebac5e206" />

## Author's Note

This challenge was authored by me for L3m0nCTF 2025. It was designed to test several key forensic analysis skills:

- Interpretation of a seed as an ordering mechanism
- Rejection of traditional steganography tools in favor of programmatic analysis
- Pixel-level manipulation rather than file-level extraction
- Reconstruction of shuffled data using deterministic randomness
- Multi-bitplane extraction guided by a controlled permutation

**Key Challenge Design:** Without applying the seed correctly, all extracted data appears as noise.

---

## Initial Reconnaissance

### File Analysis

Extracting the archive reveals an image file.

```bash
tar -xzf lost_signal.tar.gz
```

### Traditional Stego Tools

Running standard forensic tools yields nothing:

```bash
# Metadata analysis
exiftool challenge_color_random.png

# Binary analysis
binwalk challenge_color_random.png
strings challenge_color_random.png

# Visual stego analysis
stegsolve challenge_color_random.png
```

**Result:** No hidden files, no obvious patterns, no visual anomalies.

---

## Phase 1: Interpreting the Seed

### Understanding the Clue

The challenge explicitly provides: **Seed = 739391**

A seed is not encryption — it's used to:
- Initialize a pseudo-random number generator (PRNG)
- Produce a **deterministic order**

**Key Inference:** The hidden data depends on a specific order, reproducible using the seed.

Since:
- The file is an image
- No metadata or embedded files exist
- StegSolve shows no spatial patterns

**Conclusion:** The seed defines an **order of pixels**, not bytes or files.

---

## Phase 2: Pixel-Level Analysis

### Why StegSolve Fails

StegSolve works by analyzing bitplanes in their original spatial layout. In this challenge:
- Data has been **shuffled** at the pixel level
- No spatially coherent pattern exists
- Visual analysis tools are ineffective

### Programmatic Approach

Load the image as a pixel array for manipulation:

```python
from PIL import Image
import numpy as np

img = Image.open("challenge_color_random.png")
pixels = np.array(img)
print(f"Image shape: {pixels.shape}")
```

---

## Phase 3: Isolating the Luminance Channel

### Why Luminance?

RGB mixes color and brightness. For steganography:
- **Luminance (Y)** carries brightness information
- Small brightness changes are **visually imperceptible**
- Color changes are **more noticeable**

### Extracting the Y Channel

Convert to YCbCr color space and extract luminance:

```python
from PIL import Image
import numpy as np

# Load and convert to YCbCr
img = Image.open("challenge_color_random.png").convert("YCbCr")
Y, Cb, Cr = img.split()

# Extract Y channel as numpy array
Y_arr = np.array(Y, dtype=np.uint8)
np.save("Y.npy", Y_arr)

print(f"Y channel extracted: {Y_arr.shape}")
```

---

## Phase 4: Reconstructing the Permutation

### Generating the Pixel Order

The seed recreates the **exact pixel visit order** used during embedding:

```python
import numpy as np

# Configuration
seed = 739391
Y = np.load("Y.npy")
h, w = Y.shape

# Generate permutation
rng = np.random.RandomState(seed)
indices = np.arange(h * w)
rng.shuffle(indices)

# Save for extraction
np.save("perm.npy", indices)
print(f"Permutation generated: {len(indices)} pixels")
```

**Key Concept:** `indices[i]` tells us which pixel to visit at step `i`.

### Why This Matters

Without the correct seed:
- The permutation is different
- Pixels are read in the wrong order
- Extracted data is **pure noise**

---

## Phase 5: Controlled Bitplane Extraction

### Why Simple LSB Fails

A basic LSB extraction doesn't work because:
- **Multiple bits** are used per pixel
- Bits are **interleaved** across bitplanes
- The extraction must follow the **shuffled order**

### Multi-Bitplane Strategy

Extract two least significant bits (LSB0 and LSB1) and alternate between them:

```python
import numpy as np

# Load data
Y = np.load("Y.npy")
perm = np.load("perm.npy")
h, w = Y.shape

# Bitplanes to extract
LSBS = [0, 1]  # LSB0 and LSB1

# Total bits to extract
total_bits = h * w
qr_flat = np.zeros(total_bits, dtype=np.uint8)

# Extract bits in shuffled order
for i in range(total_bits):
    # Which pixel to visit (following permutation)
    pix_idx = perm[i // len(LSBS)]
    
    # Which bitplane to read
    bitplane = LSBS[i % len(LSBS)]
    
    # Convert 1D index to 2D coordinates
    y = pix_idx // w
    x = pix_idx % w
    
    # Extract bit from specific bitplane
    qr_flat[i] = (Y[y, x] >> bitplane) & 1

# Save extracted bitstream
np.save("qr_flat.npy", qr_flat)
print(f"Bitstream recovered: {len(qr_flat)} bits")
```

**Bitplane Extraction Pattern:**
```
Pixel 0: LSB0 → bit 0
Pixel 0: LSB1 → bit 1
Pixel 1: LSB0 → bit 2
Pixel 1: LSB1 → bit 3
...
```

### Critical Dependencies

Without:
- ✗ The correct seed → wrong pixel order
- ✗ The correct bitplanes → wrong data
- ✗ The correct interleaving → corrupted structure

The output is **indistinguishable from noise**.

---

## Phase 6: QR Code Reconstruction

### Reshaping the Bitstream

The flat bitstream must be reshaped back into a 2D image:

```python
from PIL import Image
import numpy as np

# Load extracted bits
qr_flat = np.load("qr_flat.npy")
Y = np.load("Y.npy")
h, w = Y.shape

# Reshape to image dimensions
qr = qr_flat.reshape((h, w))

# Invert for proper QR code colors (black/white swap)
qr_img = (255 * (1 - qr)).astype(np.uint8)

# Save as image
Image.fromarray(qr_img).save("solved_qr.png")
print("QR code saved as solved_qr.png")
```

### Result

<img width="998" height="598" alt="image" src="https://github.com/user-attachments/assets/7d62af25-205f-43fd-8a24-ea2f6c3ad541" />


---

## Scanning the QR Code

Using any QR scanner:

```bash
zbarimg solved_qr.png
```

**Output:**
```
QR-Code:L3m0nCTF{1nv1s1bl3_b1tpl4n3_x0r_qr}
```

---

## Flag

```
L3m0nCTF{1nv1s1bl3_b1tpl4n3_x0r_qr}
```

---

## Complete Solution Script

```python
#!/usr/bin/env python3
"""
L3m0nCTF 2025 - Lost Signal
Complete solution script
"""

from PIL import Image
import numpy as np

# Configuration
SEED = 739391
INPUT_IMAGE = "challenge_color_random.png"
OUTPUT_QR = "solved_qr.png"

def extract_y_channel(image_path):
    """Extract luminance channel from image"""
    img = Image.open(image_path).convert("YCbCr")
    Y, _, _ = img.split()
    return np.array(Y, dtype=np.uint8)

def generate_permutation(h, w, seed):
    """Generate pixel visit order using seed"""
    rng = np.random.RandomState(seed)
    indices = np.arange(h * w)
    rng.shuffle(indices)
    return indices

def extract_bits(Y, perm, bitplanes=[0, 1]):
    """Extract bits from shuffled pixels"""
    h, w = Y.shape
    total_bits = h * w
    bits = np.zeros(total_bits, dtype=np.uint8)
    
    for i in range(total_bits):
        pix_idx = perm[i // len(bitplanes)]
        bitplane = bitplanes[i % len(bitplanes)]
        
        y = pix_idx // w
        x = pix_idx % w
        
        bits[i] = (Y[y, x] >> bitplane) & 1
    
    return bits

def reconstruct_qr(bits, h, w):
    """Reshape bits into QR code image"""
    qr = bits.reshape((h, w))
    qr_img = (255 * (1 - qr)).astype(np.uint8)
    return Image.fromarray(qr_img)

def main():
    print("="*60)
    print("L3m0nCTF 2025 - Lost Signal Solver")
    print("="*60)
    
    # Step 1: Extract Y channel
    print("\n[1] Extracting luminance channel...")
    Y = extract_y_channel(INPUT_IMAGE)
    h, w = Y.shape
    print(f"    Image dimensions: {h}x{w}")
    
    # Step 2: Generate permutation
    print(f"\n[2] Generating permutation with seed {SEED}...")
    perm = generate_permutation(h, w, SEED)
    print(f"    Permutation length: {len(perm)}")
    
    # Step 3: Extract bits
    print("\n[3] Extracting bits from shuffled pixels...")
    bits = extract_bits(Y, perm)
    print(f"    Bits extracted: {len(bits)}")
    
    # Step 4: Reconstruct QR
    print("\n[4] Reconstructing QR code...")
    qr_img = reconstruct_qr(bits, h, w)
    qr_img.save(OUTPUT_QR)
    print(f"    QR code saved: {OUTPUT_QR}")
    
    print("\n" + "="*60)
    print("Extraction complete! Scan the QR code to get the flag.")
    print("="*60)

if __name__ == "__main__":
    main()
```

---

