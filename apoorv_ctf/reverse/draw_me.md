<img width="572" height="575" alt="Screenshot 2026-03-08 151349" src="https://github.com/user-attachments/assets/b7c53212-6a05-45ef-8a60-06e8c4c51a43" />
# Draw Me - CTF Challenge Writeup

## Challenge Description
“Draw me, just draw me”. The bat has been watching. Across centuries, across wars, across history, it always shows up. And now it's left you something: a cryptic shader program and a black image. Nothing renders. Nothing makes sense. The bat is in there somewhere, waiting to be drawn.

**Author**: makeki

## Provided Files
- `runner.html`: An HTML file setting up a WebGL 2.0 environment.
- `challenge.glsl`: A fragment shader containing the logic for a custom Virtual Machine.
- `program.png`: A 256x256 image that serves as the memory and instruction state for the VM.

## Analysis

### 1. `runner.html` and `challenge.glsl`
Upon inspecting the HTML and GLSL files, it became clear that the challenge implements a complete virtual machine entirely within a fragment shader.

The WebGL setup ping-pongs two framebuffers, using `program.png` as the initial texture (`u_state`). 

Looking inside `challenge.glsl`, the VM architecture is defined as follows:
- The state is stored in a 256x256 grid.
- **`coord (0, 0)`** stores the Instruction Pointer (IP).
- **`coord (1, 0)` through `(32, 0)`** serve as 32 general-purpose registers.
- **Memory/Instructions**: The rest of the image contains instructions and data.
- **VRAM**: Writing to Y-coordinates >= 128 maps to Video RAM, which is what the fragment shader renders.

The VM operates on 16 instructions per frame. If we decode the instructions:
- `Opcode 0`: NOP
- `Opcode 1`: LOAD_IMM (Load Immediate)
- `Opcode 2`: ADD
- `Opcode 3`: SUB
- `Opcode 4`: XOR
- `Opcode 5`: JMP
- `Opcode 6`: JNZ
- `Opcode 7`: DRAW (Writes to VRAM based on X, Y, and color registers)
- `Opcode 8`: STORE (Writes back 4 bytes to an arbitrary memory location)
- `Opcode 9`: LOAD (Reads from an arbitrary memory location)

### 2. Emulating the VM
The `program.png` file contained roughly 32,512 non-zero pixels. Rather than executing this manually or letting WebGL run indefinitely without insight into the intermediate states, we wrote a Python emulator to execute the VM logic perfectly out-of-band.

A crucial discovery was understanding that HTML `<img>` elements loaded into WebGL textures place `(0, 0)` at the top-left rather than the bottom-left. Once the orientation was corrected, the IP initialized successfully at `(0, 1)`, and our Python script executed exactly **6,124 steps** before hitting a `JMP` to its own current instruction (a classic HALT state at IP `250, 8`).

### 3. Extracting the Flag
Knowing the VM had halted, we dumped the region of memory corresponding to VRAM (rows 128 to 255) into a new image.

Since the values written to VRAM were very low intensity, we normalized them and dumped the output to ASCII art in our terminal. The rendering revealed a custom 5x5 pixel font displaying the flag characters one by one.

### 4. Reading the Leetspeak
After separating the VRAM letters out and reading them carefully (keeping in mind the author's note about lowercase and numbers), we parsed the leetspeak appropriately:
- `5` instead of `s`
- `1` instead of `i`
- `7` instead of `t`
- `0` instead of `o`
- `3` instead of `e`

## Final Flag
**`apoorvctf{gl5l_15_7ur1ng_c0mpl373}`**
