# MDA-Win8086 Kit Simulator

A single-file, browser-based simulator for the **MDA-Win8086** 8086 microprocessor trainer kit. Write assembly in MDA-WinIDE8086 syntax, assemble it, run it, and watch it drive a virtual board — a 4-LED bank, a 7-segment display, an 8×8 **bi-colour** dot matrix, a stepper motor, and a hex keypad — all mapped to the kit's real I/O ports.

Built for the Microprocessor Lab course. **Author: Hamza Alshafei.**

## Quick start

No install, no build, no dependencies. Just open the file:

1. Open `mda8086_simulator.html` in any modern browser (double-click it, or drag it into a browser tab).
2. Pick a program from the **Load example** dropdown, or paste your own `.ASM` into the editor.
3. Click **Assemble**, then **Run** (or **Step** to execute one instruction at a time).
4. Use **Speed** to slow down or speed up execution, **Stop**/**Reset** as needed.
5. Drive the keypad by **clicking the on-screen keys** while a program is running.

## Features

- **8086 assembler + interpreter** for the instruction/directive subset used in the lab.
- **Live register and flag view** (AX/BX/CX/DX/SI/DI, ZF, CF) and an instruction counter.
- **Virtual peripherals** driven through the kit's actual 8255 (PPI) port map:
  - **4-LED bank** — fixed physical colours (Red, Green, Yellow, Red), active-high.
  - **7-segment display** — standard common-anode hex font.
  - **Bi-colour 8×8 dot matrix** — red + green planes → **orange**; supports both the **scan** (multiplexed) and **flash** (full-frame) display methods.
  - **Stepper motor** — direction-aware, **1.8° per step** (200 steps/revolution).
  - **Hex keypad** — momentary keys with the real kit's strobe convention.
  - **Elevator control (MDA-007)** — a 4-floor elevator with an animated cage, up/down motor, green/red direction lamps, a FLOOR 7-segment, per-floor sign LEDs, and clickable call buttons.
  - **I/O control board (MDA-001)** — 8 clickable input slide switches, 8 output LEDs, and an FND 7-segment.
- The elevator and the I/O board share the same 8255 (ports 20–26H), just like the real kit — an **"accessory on ports 20–26H" selector** picks which one is connected (loading either board's example selects it automatically).
- **Error reporting with line numbers** — assemble and runtime errors tell you exactly which source line failed.
- **Built-in examples** for every device, plus a bi-colour matrix demo.
- Runs entirely client-side — a single self-contained HTML file.

## Virtual peripherals & port map

| Device | Port(s) | Notes |
|---|---|---|
| **LED bank** (4 LEDs) | `PPIB` = `1BH`, bits 0–3 | **Active-high** (bit = 1 lights the LED). Fixed colours L→R: **Red, Green, Yellow, Red** — bit0→Red, bit1→Green, bit2→Yellow, bit3→Red. |
| **7-Segment** | `PPIA` = `19H` | Active-low, standard common-anode hex font: segment `a` = bit0 … `g` = bit6, `dp` = bit7 (`0C0H`=`0`, `0F9H`=`1`, … `90H`=`9`). |
| **Dot matrix 8×8** (bi-colour) | `PPIB`=`1AH` (red), `PPIA`=`18H` (green), `PPIC`=`1CH` (scan), ctrl `1EH` | Two active-low data planes (0 = lit) + a scan register. **Red + green on the same dot = orange.** Orientation: **`PPIC` = column scan along the bottom** (bit0 = leftmost column), **`PPIB`/`PPIA` = row data down the side** (bit7 = top row). |
| **Stepper motor** | `PPIB` = `1BH` | 4-bit coil pattern. Each output pulse = **one 1.8° step**. Direction from the coil sequence: **`ROL` = counter-clockwise, `ROR` = clockwise** (a single pattern-reset at a loop boundary won't reverse it). |
| **Keypad** (hex) | port `01H` | Read with `IN`. **Bit7 = strobe: set = no key, clear = key ready**; the key code is in the low bits (hex keys `0`–`F` = `00H`–`0FH`, function keys = `10H`–`18H`). Keys are **momentary**: `OUT` to port `01H` (`OUT KEY,AL`) releases the key, and Reset/Assemble clears any held key. |

> The main-board port map above was reverse-engineered and confirmed against real lab sources (`MATRIX_2/3/4.ASM`, the Lab 1 report, and the MicroLab 2/3 exercises) plus the kit manuals.

### Elevator control — MDA-007 (separate 8255 at ports `20H`–`26H`)

Control byte `90H` (port A = input, ports B & C = output).

| Signal | Port / bits | Notes |
|---|---|---|
| **Cage position** | A (`20H`) high nibble, **input** | One-hot sensor: floor 1 = `10H`, floor 2 = `20H`, floor 3 = `40H`, floor 4 = `80H` (0 between floors). |
| **Call buttons** | A (`20H`) low nibble, **input** | Active-low: 1F = bit0, 2F = bit1, 3F = bit2, 4F = bit3 (buttons **latch** — click to press/release). |
| **Floor display** | B (`22H`), **output** | 7-segment digit for the current floor. |
| **Motor + lamp** | C (`24H`) low nibble, **output** | bit1 = motor up (green lamp), bit0 = motor down (red lamp); e.g. `0F6H` = up, `0F9H` = down, `0FFH` = stop. |
| **Floor-sign LED** | C (`24H`) high nibble, **output** | Active-low one-hot: floor 1 = bit4 … floor 4 = bit7 (e.g. `0BFH` = stop + floor-3 sign). |

> The elevator port map comes from the MDA-007 manual (Fig. 4) and the Elevator lab report. Note: the manual's Fig. 4 shows the cage-position sensor as a *binary* count (floor 3 = `3×H`), but the lab report's tested code reads it as **one-hot** (floor 3 = `40H`); the simulator matches the report so the lab example programs run as written.

### I/O control board — MDA-001 (same 8255 at ports `20H`–`26H`)

Control byte `90H` (port A = input, ports B & C = output). Shares the 8255 with the elevator — the **accessory selector** picks which board is connected.

| Signal | Port | Notes |
|---|---|---|
| **Input switches** | A (`20H`), **input** | 8 slide switches; switch on = bit 1 (click to toggle). |
| **FND display** | B (`22H`), **output** | 7-segment digit. |
| **Output LEDs** | C (`24H`), **output** | 8 LEDs, **active-high** (`00H` = all off). |

## Dot matrix: scan vs. flash

The matrix supports the two methods taught in the lab:

- **Scan method** — energise one column at a time via `PPIC` (a one-hot pattern that walks across the columns) while `PPIB`/`PPIA` supply that column's rows. The simulator keeps a per-column persistence buffer, so the whole shape stays lit (persistence of vision) instead of flickering a single column. This is how you draw arbitrary shapes/letters.
- **Flash method** — select several columns at once (`PPIC` with multiple bits, e.g. `0FFH`) and drive the row data; the frame is shown live. Good for solid fills and colour cycling.

## Built-in examples

- **LED bank** — turns on Red, Yellow, Green three times (Lab 1, Task 1).
- **7-Segment countdown** — counts 9→0, three times.
- **Dot Matrix (MATRIX_2.ASM)** — scans a shape into the matrix (red).
- **Dot Matrix colour** — cycles the whole matrix red → green → orange (flash method).
- **Stepper motor** — steps the rotor with a rotating coil pattern.
- **Keypad read demo** — reads a key and echoes its code.
- **Elevator MDA-007** — drives the cage up to floor 3, stops, shows "3" and lights the floor-3 sign.
- **I/O board MDA-001** — mirrors the 8 input switches onto the 8 output LEDs (flip switches while it runs).

## Writing your own programs

Use MDA-WinIDE8086 syntax. Supported features:

- **Instructions:** `MOV`, `OUT`, `IN`, `ADD`, `SUB`, `CMP`, `INC`, `DEC`, `AND`, `TEST`, `CLC`, `ROL`, `ROR`, `SHL`/`SAL`, `SHR`, `PUSH`, `POP`, `CALL`, `RET`, `JMP`, `JZ`/`JE`, `JNZ`/`JNE`, `JC`, `JNC`, `LOOP`, `NOP`, `INT`.
- **Directives:** `SEGMENT`/`ENDS`, `ASSUME`, `PROC`/`ENDP`, `EQU`, `ORG`, `DB`, `END`.
- **Operands:** byte/word registers, immediates (`10110000B`, `0C0H`, decimals), `OFFSET label`, and memory via `BYTE PTR CS:[SI]`, `BYTE PTR [label]`, or `BYTE PTR label`. A data label may sit on its own line above its `DB` bytes.
- **Errors** are reported with the offending source line, e.g. `Runtime error: Unimplemented instruction: MUL (line 12)` or `Assemble error: ... (line 5)`.

A minimal example (blink the red LED at bit 0):

```asm
CODE    SEGMENT
    ASSUME CS:CODE,DS:CODE,ES:CODE,SS:CODE
PPIC_C  EQU 1FH
PPIB    EQU 1BH
    ORG 1000H
    MOV AL,10000000B
    OUT PPIC_C,AL          ; 8255 mode 0, all outputs
BLINK:
    MOV AL,00000001B       ; bit0 = Red LED on (active-high)
    OUT PPIB,AL
    CALL DELAY
    MOV AL,00000000B       ; all off
    OUT PPIB,AL
    CALL DELAY
    JMP BLINK
DELAY:  MOV CX,0FFFFH
D1:     NOP
    LOOP D1
    RET
CODE    ENDS
    END
```

## Notes & limitations

- The interpreter implements the subset listed above; instructions or addressing modes outside it raise an error that names the source line.
- Timing loops (`NOP`/`LOOP`) don't map to real wall-clock time; use the **Speed** slider to control animation speed.
- The keypad reads the **on-screen** keys (not your physical keyboard). A key stays "pressed" until the program releases it with `OUT KEY,AL`, or until you Reset/Assemble.
- Stepper direction is inferred from the coil-excitation sequence; an *intentional* mid-program reversal is accurate to within one step at the turnaround.

## License

Educational project — free to use and modify for learning.
