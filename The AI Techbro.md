# The AI Techbro - CTF Writeup

**CTF:** RootAccess2026  
**Category:** AI  
**Difficulty:** Hard

---

## Challenge

Neural network encoder (16D → 4D) replaces password hashing. Find a 16-char alphanumeric password that maps within Euclidean distance 0.00025 of target vector `[-8.175837, -1.710289, -0.7002581, 5.3449903]`.

**Files:**
- `checker.py` — Encoder architecture
- `encoder_weights_updated.npz` — Weights

---

## The Flaw

16D → 4D compression loses ~50+ bits of information. Many passwords map to similar outputs. Unlike crypto hashes, neural encoders are easy to optimize.

---

## Solution

**Strategy:** Greedy local search + Simulated annealing

### Algorithm

```
1. Random 16-char string
2. Greedy: For each position, try all 36 chars, keep best
3. Simulated annealing: Mutate 1-3 chars, accept with probability exp(-ΔE/T)
4. Temperature: 0.3 → 0.00005 (exponential decay)
5. Repeat until distance < 0.00025
```

### Implementation

```python
import torch
import numpy as np
import random
from checker import Encoder10

# Setup
encoder = Encoder10()
encoder.load_state_dict(torch.load('encoder_weights_updated.npz'))
encoder.eval()

target = torch.tensor([-8.175837, -1.710289, -0.7002581, 5.3449903])
CHARS = 'abcdefghijklmnopqrstuvwxyz0123456789'

def encode_string(s):
    return [(ord(c) - 80) / 40 for c in s]

def distance(password):
    input_vec = torch.tensor([encode_string(password)], dtype=torch.float32)
    with torch.no_grad():
        output = encoder(input_vec).squeeze()
    return torch.dist(output, target).item()

def greedy_search(password):
    current = list(password)
    improved = True
    while improved:
        improved = False
        for pos in range(16):
            best_char = current[pos]
            best_dist = distance(''.join(current))
            for char in CHARS:
                current[pos] = char
                if distance(''.join(current)) < best_dist:
                    best_dist = distance(''.join(current))
                    best_char = char
                    improved = True
            current[pos] = best_char
            if best_dist < 0.00025:
                return ''.join(current), best_dist
    return ''.join(current), distance(''.join(current))

def simulated_annealing(password, iterations=10000):
    current = list(password)
    best = current.copy()
    best_dist = distance(''.join(best))
    
    T_start, T_end = 0.3, 0.00005
    
    for i in range(iterations):
        T = T_start * (T_end / T_start) ** (i / iterations)
        
        # Mutate
        neighbor = current.copy()
        for _ in range(random.randint(1, 3)):
            neighbor[random.randint(0, 15)] = random.choice(CHARS)
        
        current_dist = distance(''.join(current))
        neighbor_dist = distance(''.join(neighbor))
        
        # Accept if better or with probability
        if neighbor_dist < current_dist or random.random() < np.exp(-(neighbor_dist - current_dist) / T):
            current = neighbor
            if neighbor_dist < best_dist:
                best = neighbor.copy()
                best_dist = neighbor_dist
        
        if best_dist < 0.00025:
            break
    
    return ''.join(best), best_dist

# Solver
def solve():
    for restart in range(100):
        password = ''.join(random.choices(CHARS, k=16))
        password, dist = greedy_search(password)
        if dist < 0.00025:
            return password, dist
        password, dist = simulated_annealing(password)
        password, dist = greedy_search(password)
        if dist < 0.00025:
            return password, dist
    return None, float('inf')

password, dist = solve()
print(f"Password: {password}")
print(f"Distance: {dist:.8f}")
```

---

## Result

```
Password: dmtiggdv08464561
Distance: 0.00015700
```

**Verification:**
```
Encoder output: [-8.175898  -1.7102152 -0.70017946  5.3450875 ]
Target vector:  [-8.175837  -1.710289   -0.7002581   5.3449903 ]
Distance: 0.000157 ✓
```

---

## Flag

```
root{50_57up!d_it5_br!ll!@n7}

```
<img width="761" height="325" alt="Screenshot 2026-02-11 100316" src="https://github.com/user-attachments/assets/0cbe1568-8ac7-428f-a690-49fcd7996fc2" />

