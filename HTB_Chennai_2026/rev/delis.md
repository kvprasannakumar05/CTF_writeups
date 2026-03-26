# Dali's Rev (HTB Chennai) - Writeup

## Challenge Info
- Challenge: **Dali's rev**
- Category: Reverse Engineering
- Files provided:
  - `client.exe`
  - `server.dll`
- Flag format: `HTB{...}`
- Challenge prompt hint: *strict Windows architectural contract*, *hidden door*, *public lobby works*

---

## 1) First look and hypothesis

I started by checking what type of binaries I had:

```bash
ls -la /home/prasanna/Desktop/CTF/HTB_chennai/Dalis_rev
file /home/prasanna/Desktop/CTF/HTB_chennai/Dalis_rev/*
```

That confirmed both were **64-bit PE files**:
- `client.exe`: PE32+ console executable
- `server.dll`: PE32+ DLL

From the challenge text and file names alone, I suspected:
1. `client.exe` is the launcher/driver.
2. `server.dll` contains the real logic.
3. The “strict contract” likely means **COM interface contract** (IUnknown-like vtable behavior).

---

## 2) Static triage: imports/exports/strings

### `server.dll` exports
I checked exports and saw only one meaningful export:
- `DllGetClassObject`

That is a very strong COM signal. A COM server often exports `DllGetClassObject` and uses class factories + interfaces.

### `client.exe` strings
I extracted strings and found high-value lines:
- `server.dll`
- `DllGetClassObject`
- `Failed to load server.dll`
- `Failed to resolve DllGetClassObject`
- `Failed to create COM object`
- `Hidden interface not found`
- `Flag: %s`

This immediately told me the whole flow:
1. `client.exe` dynamically loads `server.dll`
2. Resolves `DllGetClassObject`
3. Creates COM object
4. Tries to query a hidden interface
5. Prints flag if hidden interface call succeeds

### `server.dll` strings
Useful strings there:
- `Vault COM Server Loaded`
- `Public interface executed.`
- `.?AUIHidden@@`
- `~HGetFlagW`

Again confirms COM class + hidden interface.

---

## 3) Reverse `client.exe` main flow

I used `radare2` (`r2`) and disassembled `main`.

### High-level behavior of `main`
`main` does exactly this:
1. `CoInitialize(NULL)`
2. `LoadLibraryA("server.dll")`
3. `GetProcAddress(..., "DllGetClassObject")`
4. Calls `DllGetClassObject(CLSID, IID_IClassFactory, &factory)`
5. Calls factory `CreateInstance` to get object
6. Calls object `QueryInterface` with a second GUID (hidden interface IID)
7. If hidden interface found:
   - Call hidden vtable method at offset `0x28`
   - Print `Flag: %s`
8. Else prints `Hidden interface not found`

So the key was recovering what that hidden interface method returns.

---

## 4) Recovering GUIDs from `client.exe`

In `main`, two GUID-like blobs are hardcoded:

- At `0x140084f20`:
  - `102a1c8f349b214fabcd998877665544`
- At `0x140084f30`:
  - `010070ca11112222aaaa010203040506`

These are used in COM calls (`DllGetClassObject` and then `CreateInstance`/`QueryInterface` chain).

Then there is a hidden IID construction loop.
`client.exe` stores 16 encoded bytes and decodes each using:

```c
decoded[i] = encoded[i] ^ (i + 0x11)
```

Encoded bytes were:

```text
d5 b5 13 16 26 25 53 52 54 5f 2b 4b 5e 5f 4b 2d
```

Decoded bytes became:

```text
c4 a7 00 02 33 33 44 4a 4d 45 30 57 43 41 54 0d
```

So the hidden IID is effectively:

```text
{0200A7C4-3333-4A44-4D45-30574341540D}
```

(standard GUID formatting with little-endian handling for first fields).

---

## 5) Reverse `server.dll` side

I then reversed named methods discovered by r2, especially:
- `sym.server.dll_DllGetClassObject`
- `method.Factory.virtual_0 / virtual_24`
- `method.Vault.1.virtual_0`
- `method.Vault.1.virtual_40`

### DllGetClassObject
`DllGetClassObject` checks incoming CLSID and creates a factory object if correct. Otherwise returns COM error.

### QueryInterface path (`method.Vault.1.virtual_0`)
It checks requested IID against known interface IDs:
- public IID
- hidden IID

If matched, it returns interface pointer; else returns `0x80004002` (`E_NOINTERFACE`).

### Hidden flag method (`method.Vault.1.virtual_40`)
This was the jackpot.

Function logic (simplified):
1. It contains 30 obfuscated bytes on stack.
2. For each index `i`, writes to output buffer:

```c
out[i] = obf[i] ^ 0x55;
```

3. Null terminates and returns success.

Obfuscated bytes from function:

```text
1d 01 17 2e 16 65 18 0a 64 1b 01 66 07 13 61 16
66 0a 65 17 13 00 60 16 61 01 64 65 1b 28
```

After XOR `0x55`:

```text
HTB{C0M_1NT3RF4C3_0BFU5C4T10N}
```

---

## 6) Final flag

```text
HTB{C0M_1NT3RF4C3_0BFU5C4T10N}
```

---

## 7) Why this challenge is designed this way

From my POV, the challenge is testing whether I can:
- recognize **COM architecture** from minimal hints (`DllGetClassObject` export)
- follow **IUnknown / IClassFactory / QueryInterface / vtable** flow
- recover hidden IID construction logic
- extract the final deobfuscation routine from the hidden interface method

The trick is not heavy anti-debugging; it is understanding Windows COM “contract” semantics and navigating interface dispatch correctly.

---

## 8) Useful command snippets I used

```bash
# Basic triage
file client.exe server.dll
strings -a client.exe | grep -i "dllgetclassobject\|flag\|hidden\|server.dll"
strings -a server.dll | grep -i "vault\|interface\|getflag"

# Imports/exports
objdump -p server.dll | less
rabin2 -i client.exe
rabin2 -E server.dll

# Deep reversing
r2 -q -e bin.relocs.apply=true -c "aaa; s main; pdr" client.exe
r2 -q -e bin.relocs.apply=true -c "aaa; afl~Vault; afl~Factory" server.dll
r2 -q -e bin.relocs.apply=true -c "aaa; pdr @ method.Vault.1.virtual_40" server.dll
```

---

## 9) My concise solve summary

I identified it as a COM challenge, traced `client.exe` loading `server.dll` and calling `DllGetClassObject`, followed the class factory and `QueryInterface` flow into a hidden interface, reversed the hidden method (`vtable +0x28`), and decoded its XOR-obfuscated output to recover the flag.
