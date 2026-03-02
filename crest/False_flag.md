# False Flag CTF Writeup

## Challenge Description
**Operation: Ghost Mantis is entering containment.**

During takedown, the University Cyber Response Team (UCRT) collected multiple artifacts from compromised research systems, including live-response logs, memory-derived indicators, automated attribution findings, and captured network traffic.

As attribution analysis begins, conflicting evidence emerges. Certain indicators strongly suggest overlap with a historically known threat actor. Automated tooling reports medium confidence matches. Memory strings reference legacy components.

But Ghost Mantis is deliberate. They understand attribution workflows. And they know how investigators think.

One indicator was intentionally planted to mislead the investigation.

You must identify it. format: `CREST{falseflag_xxxxxx}`

## Provided Files
A single password-protected ZIP archive: `Ucrt_casefiles.zip`.

---

## Solution Steps

### 1. Analyzing the Archive
We are given `Ucrt_casefiles.zip`. Attempting to extract it directly results in a password prompt. Our first step is to figure out the password or break the encryption.

Running `exiftool` or `zipinfo` reveals that the ZIP contains three files utilizing AES encryption (specifically WinZip AES PBKDF2).

```bash
zipinfo Ucrt_casefiles.zip
```
Output:
```
Archive:  /home/prasanna/crest/falseflag/Ucrt_casefiles.zip
Zip file size: 6630 bytes, number of entries: 4
drwxrwxr-x  6.3 unx        0 bx stor 25-Dec-27 10:02 Ucrt_casefiles/
-rw-rw-r--  6.3 unx     1005 Bx u099 25-Dec-23 20:26 Ucrt_casefiles/attribution_summary.txt
-rw-rw-r--  6.3 unx      727 Bx u099 25-Dec-23 20:27 Ucrt_casefiles/telemetry_state.log
-rw-rw-r--  6.3 unx     6075 Bx u099 25-Dec-27 09:56 Ucrt_casefiles/traffic_capture.pcap
```

### 2. Cracking the Zip Password
Since we don't have the password, we have to extract the hash and perform a dictionary attack using a tool like John the Ripper.

First, extract the hash:
```bash
zip2john Ucrt_casefiles.zip > zip.hash
```

Next, crack the hash using the popular `rockyou.txt` wordlist:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

John successfully cracks the password. The cracked password contains leading spaces, so we must be careful when passing it to our unzipping utility.
The cracked password is: `    ciocolatax`.

We extract the files using `7z` (which handles AES Zip files securely):
```bash
7z x -p"    ciocolatax" Ucrt_casefiles.zip
```

### 3. Artifact Analysis
After extracting the archive, we find three files:
1.  `attribution_summary.txt`: An automated finding report showing execution, network, credential, and memory string artifacts.
2.  `telemetry_state.log`: A live-response collection log.
3.  `traffic_capture.pcap`: A small network capture.

The challenge description provides major hints:
*   "Certain indicators strongly suggest overlap with a historically known threat actor."
*   "Automated tooling reports medium confidence matches."
*   "Memory strings reference legacy components."

We look at the **[Recovered Strings]** section inside `attribution_summary.txt`:
```text
[Recovered Strings]
S1. "svcmon_update.ps1"
S2. "compile with logging disabled"
S3. "ZxShell loader initialized"
S4. "tasking channel opened"
```

### 4. Identifying the False Flag
The automated tooling flags `ZxShell loader initialized`.
**ZxShell** is a notorious, historically well-documented legacy Remote Access Trojan (RAT) used primarily by Chinese state-sponsored APT groups (such as APT17 / DeputyDog). 

By planting this specific memory string within their malware, the "Ghost Mantis" group knew that automated DFIR tooling would pick it up, misleading the investigation into attributing the attack to an older, legacy threat actor rather than investigating the true perpetrators.

Therefore, the intentionally planted false flag indicator is **zxshell**.

## Final Flag
`CREST{falseflag_zxshell}`
