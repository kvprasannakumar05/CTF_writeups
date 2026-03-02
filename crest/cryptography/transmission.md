# Ghost Mantis Crypto Challenge Writeup

## Challenge Description
> Investigators recovered fragments of an internal communication channel used by Ghost Mantis operators. The channel appears properly designed: public parameters were exchanged, messages were encrypted, no private keys were exposed. Yet intelligence reports suggest the system was optimized for performance over caution — and that decision may have introduced a subtle weakness. You are provided with a partial handshake transcript and several encrypted messages. No keys. No obvious mistakes. One message contains sensitive operator data. The rest are noise.

## Analysis
The provided file `transmissions.log` contains several RSA session transcripts (`alpha`, `gamma`, `delta`, `zeta`, `beta`, `epsilon`).
Each session contains:
- `modulus`: The public modulus $N$.
- `exp`: The public exponent $e$.
- `payload`: The encrypted ciphertext $C$.

After parsing the file and extracting the parameters for each session, we need to look for common RSA implementation vulnerabilities.
1. **Low Public Exponent Attack:** None of the messages had a critically low exponent combined with a small message where $M^e < N$.
2. **Common Factor Attack (GCD):** Checked the GCD of all pairs of moduli. None shared a single common prime factor (which would allow trivial factorization).
3. **Common Modulus Attack:** Checked if any two sessions used the *exact same modulus* but *different public exponents*. 

This check revealed that session `alpha` and session `gamma` shared the **exact same 2048-bit modulus** $N$:
- `alpha`: $e_1 = 65537$
- `gamma`: $e_2 = 31337$

(Note: `gamma`'s payload was lightly obfuscated/base64 encoded, which needed decoding first).

In a **Common Modulus Attack**, if the same plaintext message is encrypted with two coprime exponents $e_1$ and $e_2$ under the exact same modulus $N$, an attacker can recover the plaintext message without ever needing to factor $N$ or find the private keys.

## Exploitation
Since $\gcd(e_1, e_2) = 1$, by the Extended Euclidean Algorithm, there exist integers $x$ and $y$ such that:
$$ e_1 \cdot x + e_2 \cdot y = 1 $$

With the two ciphertexts corresponding to the same message:
$C_1 \equiv M^{e_1} \pmod N$
$C_2 \equiv M^{e_2} \pmod N$

We can recover the plaintext $M$ purely through modular arithmetic:
$$ C_1^x \cdot C_2^y \equiv (M^{e_1})^x \cdot (M^{e_2})^y \equiv M^{e_1 \cdot x + e_2 \cdot y} \equiv M^1 \equiv M \pmod N $$

*(If $x$ or $y$ happens to be negative, we simply compute the modular inverse of the ciphertext first).*

### Exploit Script
Here is the Python script `exploit.py` used to extract the data and perform the attack:

```python
import base64

def extgcd(a, b):
    # returns (g, x, y) such that a*x + b*y = g
    if a == 0:
        return (b, 0, 1)
    else:
        g, y, x = extgcd(b % a, a)
        return (g, x - (b // a) * y, y)

def common_modulus_attack(c1, c2, e1, e2, n):
    g, x, y = extgcd(e1, e2)
    if g != 1:
        return None
    
    if x < 0:
        c1 = pow(c1, -1, n)
        x = -x
    if y < 0:
        c2 = pow(c2, -1, n)
        y = -y
        
    m = (pow(c1, x, n) * pow(c2, y, n)) % n
    return m

# ... (Parsing script to extract n, e, c for alpha and gamma) ...
# alpha_c, gamma_c are the extracted integer ciphertexts

m_int = common_modulus_attack(alpha_c, gamma_c, 65537, 31337, shared_n)
print(bytes.fromhex(hex(m_int)[2:]).decode())
```

Running the attack yields the following string:
`H4sICE3am2kAA2ZsYWcudHh0AHMOcg0Oqc5NzCvJLI4vSi0tTk2JL8lIjU/OSMzLS81xSM/ILy5RrAUA/OGj+ycAAAA=`

## Flag Recovery
The recovered string starts with `H4sI`, which is the classic magic header for a Base64-encoded GZIP file. 

To decode and read the file, we can execute the following in the terminal:
```bash
echo "H4sICE3am2kAA2ZsYWcudHh0AHMOcg0Oqc5NzCvJLI4vSi0tTk2JL8lIjU/OSMzLS81xSM/ILy5RrAUA/OGj+ycAAAA=" | base64 -d | zcat
```

**Output:**
```
CREST{mantis_reused_the_channel@ghost!}
```
