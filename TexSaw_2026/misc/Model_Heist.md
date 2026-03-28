# Neural Networks Are Like Onions - Writeup 
<img width="437" height="482" alt="image" src="https://github.com/user-attachments/assets/b58c782c-9f09-4eba-b0b9-f4de60a0a282" />

## Challenge Info
- Category: Misc
- Prompt: `Neural networks are like onions - or was that ogres?`
- Flag format: `texsaw{...}`
- Given file: `model.h5`

## Initial Approach
When I started, I only had one file (`model.h5`), so I treated this as a model-steganography challenge.

I first did quick recon to understand what was inside the file:

```bash
ls -la
file model.h5
strings -n 6 model.h5 | head -n 80
```

From `strings`, I noticed:
- This is a Keras/TensorFlow model.
- There is a suspicious layer named `secret_layer` with `26` units.

That strongly suggested the hidden data is likely in model weights, not in plain metadata.

## Environment Setup
I needed Python HDF5 tooling to inspect `.h5` internals. Since system Python was externally managed, I used a local virtual environment:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install h5py numpy
```

## Enumerating HDF5 Structure
I listed datasets and shapes:

```bash
.venv/bin/python - <<'PY'
import h5py
f=h5py.File('model.h5','r')
def walk(g,p=''):
    for k,v in g.items():
        pp=f'{p}/{k}'
        if isinstance(v,h5py.Dataset):
            print(pp, v.shape, v.dtype)
        else:
            walk(v,pp)
walk(f)
PY
```

Important datasets:
- `model_weights/secret_layer/sequential/secret_layer/kernel` shape `(128, 26)` float32
- `model_weights/secret_layer/sequential/secret_layer/bias` shape `(26,)` float32

The `secret_layer` kernel looked like the best candidate for an encoded payload.

## Extraction Strategy
I tried common decoding methods:
- Raw byte scans for `texsaw{`
- Bit-plane extraction from float32 representations
- Attribute and metadata checks

None of those gave a direct hit.

Then I tested scalar transforms over weights. The successful transform was:

1. Flatten `secret_layer/kernel`
2. Multiply each float by `1000`
3. Round to nearest integer
4. Cast to `uint8`
5. Convert to bytes
6. Search for `texsaw{...}`

This immediately produced the flag string.

## Exploit Code
I saved this as `solve.py`:

```python
#!/usr/bin/env python3
import h5py
import numpy as np


def main():
    model_path = "model.h5"
    dataset_path = "model_weights/secret_layer/sequential/secret_layer/kernel"

    with h5py.File(model_path, "r") as f:
        weights = f[dataset_path][()]

    # Core trick: scale floats, round, and reinterpret as byte values.
    payload = np.rint(weights.reshape(-1) * 1000).astype(np.uint8).tobytes()
    marker = b"texsaw{"
    start = payload.find(marker)
    if start == -1:
        raise RuntimeError("Flag marker not found")
    end = payload.find(b"}", start)
    if end == -1:
        raise RuntimeError("Flag closing brace not found")

    flag = payload[start : end + 1].decode()
    print(flag)


if __name__ == "__main__":
    main()
```

Run:

```bash
.venv/bin/python solve.py
```

Output:

```text
texsaw{w3ight5_t3ll_t4l3s}
```

## Final Flag
`texsaw{w3ight5_t3ll_t4l3s}`

## Tools I Used
- Shell basics: `ls`, `file`, `cat`
- Fast file discovery: `rg --files`
- Static inspection: `strings`
- Python 3.13 in local venv
- Python packages: `h5py`, `numpy`
- Custom one-off Python scripts for dataset enumeration and decoding

## Why This Worked
The challenge hid plaintext bytes in neural net weights by storing values that map back to ASCII after scaling and rounding. The clue about onions/ogres hints at “layers,” pushing me toward inspecting hidden data in an internal model layer (`secret_layer`).
