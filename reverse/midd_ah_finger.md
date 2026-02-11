# mid_aah_finguh

**Category:** Forensics / Reverse Engineering  
**Difficulty:** Medium

---

## Overview

The challenge provides a PNG image of Sukuna's cursed finger from *Jujutsu Kaisen*. Although the image resolution is only **100×159 pixels**, the file size is around **63 KB**, which is unusually large for such a small PNG. This strongly suggests that additional data is embedded inside the image.

---

## 1. Initial Recon

### Strings Analysis

The first step was to check for readable strings inside the image.
```bash
strings middle_fingu.png
```

A suspicious Base64 string appeared:
```
WU9VX0NBTl9PTkxZX1VOTE9DS19NRV9XSEVOX0lfQU1fQVRfTVlfRlVMTF9QT1dFUg==
```

Decoding it revealed the following message:
```
YOU_CAN_ONLY_UNLOCK_ME_WHEN_I_AM_AT_MY_FULL_POWER
```

This looked like a deliberate hint rather than random data.

### File Structure Analysis

Next, the image was analyzed using `binwalk`.
```bash
binwalk middle_fingu.png
```

Output:
```
DECIMAL       HEXADECIMAL     DESCRIPTION
-------------------------------------------------------
0             0x0             PNG image
27916         0x6D0C          ELF 64-bit LSB executable
51868         0xCA9C          PDF document, version 1.7
```

This confirmed that the PNG contains two embedded files:
- A 64-bit Linux ELF executable
- A PDF document

---

## 2. ELF Binary Analysis (Red Herring)

The embedded ELF binary was extracted first.
```bash
dd if=middle_fingu.png of=hidden_elf bs=1 skip=$((0x6D0C))
```

Upon reversing the binary, functions such as the following were observed:
- `solve()`
- `kitne_fingers()`

Multiple XOR keys and decoding approaches were tested, but none produced meaningful results. After spending significant time on it, it became clear that this binary was intentionally designed as a **red herring** to mislead the solver.

---

## 3. Extracting the Real Target (PDF)

After realizing the ELF binary was bait, focus was shifted to the embedded PDF.
```bash
dd if=middle_fingu.png of=hidden.pdf bs=1 skip=$((0xCA9C))
```

The extracted PDF was **password-protected**.

### Determining the PDF Password

Revisiting the earlier Base64 hint:
```
YOU_CAN_ONLY_UNLOCK_ME_WHEN_I_AM_AT_MY_FULL_POWER
```

In *Jujutsu Kaisen* lore, Sukuna reaches full power only after consuming all **20 cursed fingers**.

Using `20` as the password successfully unlocked the PDF.

---

## 4. Flag Recovery

Inside the PDF, the content was not readable plaintext. Instead, the following encoded string was found:
```
7**1>//.7v#v7v+&vpq7v#0+8
```

This suggested a simple encoding technique.

### XOR Key Identification

CTF flags typically begin with `root{`. XORing the first few characters of the encoded string with `root{` revealed a constant key:
```
'7' XOR 'r' = 0x45
'*' XOR 'o' = 0x45
'*' XOR 'o' = 0x45
'1' XOR 't' = 0x45
```

This confirmed the encoding used a **single-byte XOR key: 0x45**.

### Decoding the Flag

The full string was decoded using a short Python script.
```python
encoded = "7**1>//.7v#v7v+&vpq7v#0+8"
print(''.join(chr(ord(c) ^ 0x45) for c in encoded))
```

**Final Flag:**
```
root{jjkr3f3r3nc354r3fun}
```

---

## Summary

1. **Initial reconnaissance** revealed embedded files within the PNG
2. **Extracted ELF binary** turned out to be a red herring
3. **Extracted PDF** was password-protected with the password `20` (reference to Sukuna's 20 fingers)
4. **Flag was XOR-encoded** with key `0x45` inside the PDF
5. **Final flag:** `root{jjkr3f3r3nc354r3fun}`

---

