# Project 816_NOT ISA Specification

This document contains the structural blueprint and initial draft for the custom 16-bit Instruction Set Architecture (ISA) designed for **Project 816_NOT**.

---

## 1. Architectural Foundation

* **Word Size:** 16-bit instruction word budget.
* **Data Width:** 16-bit wide internal registers and data pathways.
* **Addressable Memory:** 16-bit base pointers allow access up to **2¹⁶** = 65,536 unique memory locations (64 KB RAM).

---

## 2. Instruction Formats

In order to simplify our implementation, the Opcode will occupy the same exact position: bits **[15:11]** across all instruction formats.

### Register-to-Register Format (R-Type)
Generally used for arithmetic and logic operations between registers.

| 15 . . . 11 | 10 . . . 8 | 7 . . . 5 | 4 . . . 2 | 1 . . 0 |
| :---: | :---: | :---: | :---: | :---: |
| **Opcode** | **Rd** | **Rs** | **Rt** | **Reserved** |
| 5 bits | 3 bits | 3 bits | 3 bits | 2 bits |

* **Rd (Bits [10:8]):** Destination register.
* **Rs (Bits [7:5]):** First source operand register.
* **Rt (Bits [4:2]):** Second source operand register.
* **Reserved (Bits [1:0]):** Padding for future expansions.

### Immediate/Memory Format (I-Type)
Used for operations involving small constant values, memory offsets, or conditional branches.

| 15 . . . 11 | 10 . . . 8 | 7 . . . 5 | 4 . . . . . . . . . . . . . . . 0 |
| :---: | :---: | :---: | :---: |
| **Opcode** | **Rt** | **Rs** | **Immediate** |
| 5 bits | 3 bits | 3 bits | 5 bits |

* **Rt (Bits [10:8]):** Target register. Depending on opcode, may be a source or destination.
* **Rs (Bits [7:5]):** Base register, which may hold the memory address pointer or first operand.
* **Immediate (Bits [4:0]):** 5-bit constant field (ranges: -16 to +15 signed, or 0 to 31 unsigned).

### Load Upper Immediate Variant (Modified I-Type)
Eliminates the `Rs` field to allow wider 8-bit immediate values for processing large constants.

| 15 . . . 11 | 10 . . . 8 | 7 . . . . . . . . . . . . . . . . . . . . . . . 0 |
| :---: | :---: | :---: |
| **Opcode** | **Rt** | **Immediate** |
| 5 bits | 3 bits | 8 bits |

### Unconditional Jump Format (J-Type)
Used for direct control flow changes across large blocks of program memory.

| 15 . . . 11 | 10 . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 0 |
| :---: | :---: |
| **Opcode** | **Jump Address** |
| 5 bits | 11 bits |

* **Jump Address (Bits [10:0]):** 11-bit absolute target address or PC-relative offset (range is 2048 instructions).

---

## 3. The 32-Slot Opcode Table

With 5 bits available for representing opcodes, the architecture supports a maximum of 32 instructions. This v1 draft implements 19 baseline instructions, which leaves 13 slots open for future expansions.

| Group | Instruction | Opcode (Binary) | Syntax Example | Operation Description |
| :--- | :--- | :--- | :--- | :--- |
| **Ctrl** | `NOP` | `00000` | `NOP` | No Operation (Pipeline stall safeguard) |
| **ALU** | `ADD` | `00001` | `ADD R1, R2, R3` | R₁ = R₂ + R₃ |
| **ALU** | `SUB` | `00010` | `SUB R1, R2, R3` | R₁ = R₂ - R₃ |
| **ALU** | `AND` | `00011` | `AND R1, R2, R3` | R₁ = R₂ & R₃ |
| **ALU** | `OR` | `00100` | `OR R1, R2, R3` | R₁ = R₂ \| R₃ |
| **ALU** | `XOR` | `00101` | `XOR R1, R2, R3` | R₁ = R₂ ⊕ R₃ |
| **ALU** | `SLT` | `00110` | `SLT R1, R2, R3` | R₁ = (R₂ < R₃) ? 1 : 0 |
| **ALU** | `SLL` | `00111` | `SLL R1, R2, R3` | R₁ = R₂ << R₃ |
| **ALU** | `SRL` | `01000` | `SRL R1, R2, R3` | R₁ = R₂ >> R₃ |
| **Imm** | `ADDI` | `01001` | `ADDI R1, R2, 1` | R₁ = R₂ + Imm₅ |
| **Imm** | `ANDI` | `01010` | `ANDI R1, R2, 2` | R₁ = R₂ & Imm₅ |
| **Imm** | `ORI` | `01011` | `ORI R1, R2, 3` | R₁ = R₂ \| Imm₅ |
| **Mem** | `LW` | `01100` | `LW R1, 4(R2)` | R₁ = Memory[R₂ + 4] |
| **Mem** | `SW` | `01101` | `SW R1, 4(R2)` | Memory[R₂ + 4] = R₁ |
| **Mem** | `LUI` | `01110` | `LUI R1, 0x4F` | R₁ = {Imm₈, 8'b0} |
| **Ctrl** | `BEQ` | `01111` | `BEQ R1, R2, 10` | if (R₁ == R₂) PC = PC + Imm₅ |
| **Ctrl** | `BNE` | `10000` | `BNE R1, R2, 10` | if (R₁ != R₂) PC = PC + Imm₅ |
| **Ctrl** | `J` | `10001` | `J 0x03FF` | PC = JumpAddress₁₁ |
| **Ctrl** | `JAL` | `10010` | `JAL 0x03FF` | R₇ = PC + 1; PC = JumpAddress₁₁ |
| **Ctrl** | `HALT` | `11111` | `HALT` | Halts execution loop (Hardware exit) |
