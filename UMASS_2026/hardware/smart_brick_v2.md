# Smart Brick v2 Writeup

I started with the provided challenge directory and found a single board design file:

- `smart-brick-v2.kicad_pcb`

That immediately matched the first hint. The file format is from **KiCad**, the open-source EDA suite. Since I did not need to render the board visually to solve it, I treated the `.kicad_pcb` file as a text netlist and reversed the logic directly.

## Initial Recon

The second hint mentioned **7 inputs**, which strongly suggested **7-bit ASCII**. That gave me a useful working hypothesis:

- the board likely accepts a 7-bit character
- some combinational logic checks whether that input matches specific hardcoded values
- the outputs probably reveal characters or character positions in the flag

Looking through the PCB file confirmed that idea.

### Input Header

The board has a connector `J1` wired like this:

- `Pin 1 -> /IN0`
- `Pin 2 -> /IN1`
- `Pin 3 -> /IN2`
- `Pin 4 -> /IN3`
- `Pin 5 -> /IN4`
- `Pin 6 -> /IN5`
- `Pin 7 -> /IN6`
- `Pin 8 -> GND`

There is also a second connector `J2` carrying:

- `+5V`
- `GND`

So the challenge logic really is centered around seven signal inputs.

## Understanding the Circuit

Next I enumerated the logic ICs present in the board file. The design uses standard TTL logic:

- `74LS04` inverter
- `74LS00` NAND
- `74LS02` NOR
- `74LS08` AND
- `74LS20` dual 4-input NAND
- `74LS21` dual 4-input AND
- `74LS27` triple 3-input NOR
- `74LS32` OR
- `74LS86` XOR

The outputs of those gates eventually feed MOSFETs (`2N7002`) which drive 19 LEDs:

- `D1` through `D19`

Each LED is driven through:

- a logic net
- the gate of a `2N7002`
- a resistor
- the LED to ground

So each final logic net corresponds to one LED turning on.

## Strategy

At this point, opening the board in KiCad would have been possible, but unnecessary. Since the hints explicitly mentioned `kiutils`, and the PCB file is plain text anyway, I decided to reverse it programmatically.

My plan was:

1. Parse every footprint and pad from the `.kicad_pcb` file.
2. Identify each logic chip and reconstruct its boolean function from the pinout.
3. Build expressions for every intermediate net.
4. Evaluate the 19 LED-driving output nets for all `0..127` possible 7-bit inputs.
5. See which ASCII characters activate which LEDs.

This turns the board into a truth-table problem.

## Reconstructing the Logic

I extracted the relevant gate outputs and modeled them according to the standard TTL pinouts.

Examples:

- `74LS04`: `output = NOT(input)`
- `74LS21`: `output = AND(a, b, c, d)`
- `74LS20`: `output = NAND(a, b, c, d)`
- `74LS27`: `output = NOR(a, b, c)`
- `74LS86`: `output = XOR(a, b)`

One small detail mattered during parsing: I had to use the **correct pin orientation** for the `74LS02` NOR gates. Once that was fixed, the truth table became clean and meaningful.

The final LED control nets were:

- `/G59`
- `/G62`
- `/G13`
- `/G19`
- `/G21`
- `/G24`
- `/G26`
- `/G29`
- `/G31`
- `/G36`
- `/G39`
- `/G41`
- `/G43`
- `/G45`
- `/G47`
- `/G49`
- `/G52`
- `/G54`
- `/G56`

That gives exactly **19 outputs**, which is a strong hint that the board encodes **19 character positions**.

## Brute Forcing the 7-Bit Input

I evaluated the circuit for every 7-bit value from `0` to `127` and checked which LED outputs became active.

Only a small set of printable ASCII characters produced any LED hits:

- `3`
- `4`
- `A`
- `G`
- `I`
- `M`
- `S`
- `T`
- `U`
- `_`
- `h`
- `n`
- `s`
- `t`
- `{`
- `}`

For each such input character, the circuit lit one or more LEDs. That means the board is effectively a **character matcher**:

- feed in one ASCII character
- the board lights every flag position where that character occurs

This also explains repeated letters. For example:

- `S` lit two positions
- `_` lit two positions
- `3` lit two positions

So instead of outputting the flag directly in binary, the hardware acts like a bank of hardwired comparators over each flag character position.

## Recovering the Flag

I mapped each active LED to its position from left to right and then filled in the character responsible for that LED.

The reconstructed 19-character string was:

`UMASS{In_Th3_G4t3s}`

## Final Answer

`UMASS{In_Th3_G4t3s}`

## Why This Works

The overall design is basically a ROM implemented out of discrete logic gates:

- the 7 input pins represent a 7-bit ASCII character
- the intermediate gates decode conditions on those input bits and inverted bits
- each output LED corresponds to one flag character position
- an LED turns on when the current 7-bit input equals the hardcoded character stored at that position

So by sweeping every possible 7-bit input and recording which LED positions light up, I can reconstruct the entire string without ever touching real hardware.

## Short Solve Summary

If I had to summarize the solve in one paragraph:

I recognized the file as a KiCad PCB, treated the seven inputs as 7-bit ASCII per the hint, parsed the board into a gate-level netlist, simulated the TTL logic for all 128 possible inputs, observed which of the 19 LED outputs lit for each character, and used that position-to-character mapping to recover the flag `UMASS{In_Th3_G4t3s}`.
