# RootAccess CLI - CTF Writeup

**Category:** Reverse Engineering  
**Difficulty:** Medium

---

## Challenge Overview

This challenge presents an npm package called `root-access` that contains a heavily obfuscated command-line interface (CLI) tool. The objective is to extract a flag that has been split into 10 encrypted fragments hidden throughout the package's JavaScript modules.

The key insight here is that while the code is heavily obfuscated, we don't need to manually reverse it — we can leverage Node.js's dynamic runtime to interact with the code directly.

---

## Approach

Instead of spending hours manually deobfuscating RC4-encoded, control-flow-flattened JavaScript, we took a **dynamic analysis** approach: load the obfuscated modules in Node.js and call their exported methods directly to extract the flag fragments.

---

## Step 1: Initial Reconnaissance

### Installing the Package

First, we installed the npm package globally to examine its contents:

```bash
npm install -g root-access
```

### Locating the Source Files

To find where npm installed the package:

```bash
npm root -g
```

This led us to the source files at:

```
node_modules/root-access/dist/lib/
```

### Examining package.json

The `package.json` file revealed crucial information about how the code was obfuscated. The build script showed the use of `javascript-obfuscator` with several aggressive options:

- **RC4 string encoding** — All strings are encrypted with RC4 and stored in lookup arrays
- **Control flow flattening** — Code execution flow is scrambled with switch statements
- **Dead code injection** — Fake code paths added to confuse analysis
- **Unicode escapes** — Variable names converted to unicode sequences

### Package Structure

The package contained 5 obfuscated JavaScript modules:

1. `index.js` — Entry point
2. `crypto.js` — Cryptographic operations
3. `keyforge.js` — Key generation
4. `network.js` — Network utilities
5. `vault.js` — Data storage

**Critical Discovery:** All modules use `module.exports` to expose their classes. This means we can `require()` them in Node.js and call their methods directly without needing to deobfuscate anything!

---

## Step 2: Understanding the Technique

### Why Dynamic Analysis?

When facing heavily obfuscated code, you have two options:

1. **Static analysis** — Manually reverse the obfuscation
2. **Dynamic analysis** — Run the code and observe its behavior

For this challenge, static analysis would require:
- Decoding RC4 string arrays
- Unflattening control flow
- Removing dead code
- Understanding unicode-escaped identifiers

This could take many hours. However, dynamic analysis allows us to:
- Load the modules in Node.js
- Enumerate exported classes and methods
- Call functions directly to extract data

### Runtime Introspection

JavaScript provides powerful reflection capabilities. The key method we used was:

```javascript
Object.getOwnPropertyNames(className)
```

This returns all property names (including methods) of a class, allowing us to discover hidden functionality even in obfuscated code.

---

## Step 3: Exploring the Modules

### Loading the Crypto Module

We started by loading the crypto module:

```javascript
const cryptoMod = require('root-access/dist/lib/crypto.js');
const CryptoEngine = cryptoMod.CryptoEngine;
```

### Discovering Methods

Using runtime introspection, we enumerated all methods:

```javascript
console.log(Object.getOwnPropertyNames(CryptoEngine));
console.log(Object.getOwnPropertyNames(CryptoEngine.prototype));
```

This revealed several interesting methods:

- `getTotalFragments()` — Returns the number of flag fragments
- `getFragment(n)` — Retrieves a specific fragment
- `__getFullFlag__()` — Hidden method that returns the complete flag!

### Understanding Fragment Decryption

Each flag fragment goes through a decryption process:

1. **Base64 decode** — The stored fragment is base64-encoded
2. **XOR decryption** — Decoded bytes are XORed with the key `SHADOWGATE`

The obfuscated code handles this automatically when we call `getFragment()`.

---

## Step 4: Extracting the Flag

### Method 1: Manual Fragment Assembly

We created a Node.js script to extract all fragments individually:

```javascript
const cryptoMod = require('root-access/dist/lib/crypto.js');
const CryptoEngine = cryptoMod.CryptoEngine;

// Get total number of fragments
const totalFragments = CryptoEngine.getTotalFragments();
console.log(`Total fragments: ${totalFragments}`);

// Extract each fragment
let fullFlag = '';
for (let i = 1; i <= totalFragments; i++) {
    const fragment = CryptoEngine.getFragment(i);
    console.log(`Fragment ${i}: ${fragment}`);
    fullFlag += fragment;
}

console.log(`\nFull flag: ${fullFlag}`);
```

### Fragment Breakdown

Running the script revealed the 10 fragments:

| Fragment # | Value |
|------------|-------|
| 1 | `root{N0` |
| 2 | `d3_Cl1_` |
| 3 | `R3v3rs3` |
| 4 | `_3ng1n3` |
| 5 | `3r1ng_` |
| 6 | `M4st3r` |
| 7 | `_0f_` |
| 8 | `Sh4d` |
| 9 | `0ws` |
| 10 | `}` |

### Method 2: Using the Hidden Method

Even simpler — just call the hidden `__getFullFlag__()` method:

```javascript
const cryptoMod = require('root-access/dist/lib/crypto.js');
const CryptoEngine = cryptoMod.CryptoEngine;

console.log(CryptoEngine.__getFullFlag__());
```

### Verification

We also discovered the `DataVault` module had an `assembleFlag()` method that not only returns the flag but also provides a SHA256 hash for verification:

```javascript
const vaultMod = require('root-access/dist/lib/vault.js');
const DataVault = vaultMod.DataVault;

console.log(DataVault.assembleFlag());
```

**SHA256 Hash:**
```
0c950c774d2f7bbbb74c06d8abeba59e9679ee543d7928c320e90ebe6368aee6
```

---

## Final Flag

```
root{N0d3_Cl1_R3v3rs3_3ng1n33r1ng_M4st3r_0f_Sh4d0ws}
```

---



## Tools Used

- **Node.js** — JavaScript runtime for executing obfuscated code
- **npm** — Package manager for installation and source location
- **JavaScript Reflection API** — `Object.getOwnPropertyNames()` for method discovery
- **Terminal** — Command-line interface for running scripts

---

## Complete Solution Script

```javascript
#!/usr/bin/env node

const cryptoMod = require('root-access/dist/lib/crypto.js');
const vaultMod = require('root-access/dist/lib/vault.js');

const CryptoEngine = cryptoMod.CryptoEngine;
const DataVault = vaultMod.DataVault;

console.log('='.repeat(60));
console.log('RootAccess CLI - Flag Extraction');
console.log('='.repeat(60));

// Method 1: Extract fragments individually
console.log('\n[Method 1] Extracting fragments...\n');
const totalFragments = CryptoEngine.getTotalFragments();
let flag = '';

for (let i = 1; i <= totalFragments; i++) {
    const fragment = CryptoEngine.getFragment(i);
    console.log(`Fragment ${i.toString().padStart(2)}: ${fragment}`);
    flag += fragment;
}

console.log('\n' + '='.repeat(60));
console.log(`Assembled Flag: ${flag}`);
console.log('='.repeat(60));

// Method 2: Use hidden method
console.log('\n[Method 2] Using hidden method...\n');
const quickFlag = CryptoEngine.__getFullFlag__();
console.log(`Quick Flag: ${quickFlag}`);

// Verification
console.log('\n[Verification] Using DataVault...\n');
const verified = DataVault.assembleFlag();
console.log(`Verified Flag: ${verified}`);

console.log('\n' + '='.repeat(60));
console.log('Flag extraction complete!');
console.log('='.repeat(60));
```

---

## Alternative Approaches Considered

### 1. Manual Deobfuscation

We could have:
- Extracted RC4 string arrays
- Rebuilt the control flow graph
- Removed dead code

**Why we didn't:** Too time-consuming for this challenge.

### 2. Automated Deobfuscation Tools

Tools like:
- `webcrack` 
- `synchrony`
- Custom AST transformers

**Why we didn't:** Dynamic analysis was faster and more reliable.

### 3. Debugging with Chrome DevTools

We could have:
- Run the CLI with `node --inspect-brk`
- Set breakpoints in Chrome DevTools
- Step through execution

**Why I didn't:** Direct method calls were simpler.

---
