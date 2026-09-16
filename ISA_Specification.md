# Project 816_NOT: Instruction Set Architecture (ISA) Specification

This document contains the structural blueprint and initial draft for the custom 16-bit Instruction Set Architecture (ISA) designed for **Project 816_NOT**.

---

## 1. Architectural Foundation
* **Word Size:** 16-bit instruction word budget.
* **Data Width:** 16-bit wide internal registers and data pathways.
* **Addressable Memory:** 16-bit base pointers allow access to up to **2¹⁶ = 65,536 unique memory locations (64 KB RAM)**.
* **Register File:** 8 General-Purpose Registers ($R_0$ through $R_7$), requiring **3 bits** per register selection field.

---

## 2. Instruction Formats (Bit-Map Layouts)

To simplify the hardware decoder in Verilog, the **Opcode** is kept in the exact same position (Bits [15:11]) across all formats.

### Register-to-Register Format (R-Type)
Used for pure arithmetic and logical operations between registers.
```text
 15      11 10     8 7      5 4      2 1    0
+----------+--------+--------+--------+------+
|  Opcode  |   Rd   |   Rs   |   Rt   | Rsrvd|
+----------+--------+--------+--------+------+
   5 bits    3 bits   3 bits   3 bits  2 bits
```
* **Rd (Bits [10:8]):** Destination register.
* **Rs (Bits [7:5]):** First source operand register.
* **Rt (Bits [4:2]):** Second source operand register.
* **Reserved (Bits [1:0]):** Padding for future expansions (e.g., shift amounts).

### Immediate / Memory Format (I-Type)
Used for operations involving small constant values, memory offsets, or conditional branches.
```text
 15      11 10     8 7      5 4              0
+----------+--------+--------+----------------+
|  Opcode  |   Rt   |   Rs   |   Immediate    |
+----------+--------+--------+----------------+
   5 bits    3 bits   3 bits       5 bits
```
* **Rt (Bits [10:8]):** Target register (pulls double duty as destination or source depending on opcode).
* **Rs (Bits [7:5]):** Base register holding the memory address pointer or first operand.
* **Immediate (Bits [4:0]):** 5-bit constant field (supports ranges $-16$ to $+15$ signed, or $0$ to $31$ unsigned).

### Load Upper Immediate Variant (Modified I-Type)
Eliminates the $Rs$ field to allow wider 8-bit immediate values for processing large constants.
```text
 15      11 10     8 7                       0
+----------+--------+-------------------------+
|  Opcode  |   Rt   |        Immediate        |
+----------+--------+-------------------------+
   5 bits    3 bits            8 bits
```

### Unconditional Jump Format (J-Type)
Used for direct control flow changes across large blocks of program memory.
```text
 15      11 10                               0
+----------+----------------------------------+
|  Opcode  |          Jump Address            |
+----------+----------------------------------+
   5 bits                   11 bits
```
* **Jump Address (Bits [10:0]):** 11-bit absolute target address or PC-relative offset (jumps across a block of 2,048 instructions).

---

## 3. The 32-Slot Opcode Table

With a 5-bit opcode field, the architecture supports up to 32 instructions. The current draft implements **17 baseline instructions**, leaving 13 slots open for future extensions.

| Group | Instruction | Opcode (Binary) | Syntax Example | Operation Description |
| :--- | :--- | :--- | :--- | :--- |
| **ALU** | `ADD` | `00000` | `ADD R1, R2, R3` | $R_1 = R_2 + R_3$ |
| **ALU** | `SUB` | `00001` | `SUB R1, R2, R3` | $R_1 = R_2 - R_3$ |
| **ALU** | `AND` | `00010` | `AND R1, R2, R3` | $R_1 = R_2 \ \& \ R_3$ |
| **ALU** | `OR` | `00011` | `OR R1, R2, R3` | $R_1 = R_2 \ | \ R_3$ |
| **ALU** | `XOR` | `00100` | `XOR R1, R2, R3` | $R_1 = R_2 \oplus R_3$ |
| **ALU** | `SLT` | `00101` | `SLT R1, R2, R3` | $R_1 = (R_2 < R_3) ? 1 : 0$ |
| **ALU** | `SLL` | `00110` | `SLL R1, R2, R3` | $R_1 = R_2 \ll R_3$ |
| **ALU** | `SRL` | `00111` | `SRL R1, R2, R3` | $R_1 = R_2 \gg R_3$ |
| **ALU** | `JR` | `01000` | `JR R2` | $PC = R_2$ (Return from function) |
| **Imm** | `ADDI` | `01001` | `ADDI R1, R2, 5` | $R_1 = R_2 + 	ext{Imm}_5$ |
| **Imm** | `ANDI` | `01010` | `ANDI R1, R2, 5` | $R_1 = R_2 \ \& \ 	ext{Imm}_5$ |
| **Imm** | `ORI` | `01011` | `ORI R1, R2, 5` | $R_1 = R_2 \ | \ 	ext{Imm}_5$ |
| **Mem** | `LW` | `01100` | `LW R1, 4(R2)` | $R_1 = 	ext{Memory}[R_2 + 	ext{Imm}_5]$ |
| **Mem** | `SW` | `01101` | `SW R1, 4(R2)` | $	ext{Memory}[R_2 + 	ext{Imm}_5] = R_1$ |
| **Mem** | `LUI` | `01110` | `LUI R1, 0x4F` | $R_1 = \{	ext{Imm}_8, 8'	ext{b}0\}$ |
| **Ctrl**| `BEQ` | `01111` | `BEQ R1, R2, 10` | $	ext{if } (R_1 == R_2) \ PC = PC + 	ext{Imm}_5$ |
| **Ctrl**| `BNE` | `10000` | `BNE R1, R2, 10` | $	ext{if } (R_1 != R_2) \ PC = PC + 	ext{Imm}_5$ |
| **Ctrl**| `J` | `10001` | `J 0x03FF` | $PC = 	ext{JumpAddress}_{11}$ |
| **Ctrl**| `JAL` | `10010` | `JAL 0x03FF` | $R_7 = PC + 1; \ PC = 	ext{JumpAddress}_{11}$ |

---

## 4. Key Architectural Strategies

### Managing 16-bit Constants (The Two-Step Load)
Because the constraint restricts immediate fields to small bit segments, a full 16-bit constant (e.g., `0x4F2A`) is loaded into a register by coordinating `LUI` and `ORI`:
1. `LUI R1, 0x4F` shifts `0x4F` into bits [15:8], changing $R_1$ to `0x4F00`.
2. `ORI R1, R1, 0x2A` executes a bitwise OR with the lower bits, yielding `0x4F2A` inside $R_1$.