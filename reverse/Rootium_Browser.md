# Rootium Browser Write-up
### **RootAccess2026**
- **Challenge:** Rootium Browser
- **Category:** Reverse Engineering / Electron

## Overview
We were given a custom Electron-based browser with a "Secret Vault" feature locked by a master password. The goal was to reverse the authentication mechanism to unlock the vault and retrieve the root flag.

## 1. Recon & Extraction
Since the challenge provided a `.deb` Linux installer, I started by extracting the contents.
* **Unpacking:** Extracted `data.tar.xz` from the `.deb` file.
* **Electron Source:** Used `@electron/asar` to unpack `opt/rootium_browser/resources/app.asar`.

Checking `main.js`, I noticed the app wasn't handling auth in JavaScript. Instead, it spawned a native binary (`bin/rootium_vault`) and communicated via stdin/stdout. This confirmed the target was the native ELF binary, not the JS code.

## 2. Binary Analysis
I loaded the `rootium_vault` binary into a disassembler. The logic was straightforward:
1.  **Password Check:** Validates input against a stored, encoded master password.
2.  **Flag Retrieval:** If authorized, it decodes the flag.

I identified two distinct custom decoding functions. Both used a mix of XOR and Bitwise Rotation (ROR), but with different keys.

* **Function 0x125c (Password Decoder):** Uses a static 7-byte key.
* **Function 0x130A (Flag Decoder):** Uses the *decoded password* as the key.

## 3. The Solution
I wrote a Python script to emulate the decryption logic found in the binary.

### Step 1: Recover the Password
The password was encrypted using a static key (`[0xD8, ...]` which XORs to "rootium").
* *Algorithm:* `((byte ^ 0x42) >>> 3) ^ key_byte`
* *Result:* `v4ult_m4st3r_p4ss_2026`

### Step 2: Decrypt the Flag
With the password recovered, I applied the second algorithm to the encrypted flag blob found in the `.data` section.
* *Algorithm:* `((byte ^ 0x7E) >>> (i % 8)) ^ password_byte`

## Final Flag
```text
root{n0_m0r3_34sy_v4ult_3xtr4ct10n_99}
