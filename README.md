# 16-Bit Custom FPGA Computer Architecture & Design Lab

Welcome to the engineering repository for my Capstone Project. This project documents the complete, bottom-up design and implementation of a custom **16-bit computer architecture programmed on an FPGA using Verilog HDL**. 

This repository serves as a live development journal tracking the evolution of the hardware—moving from discrete digital logic prototyping to full Instruction Set Architecture (ISA) modeling.

---

## 🚀 Phase 1: Prototyping Foundations (NI Multisim Lab)

Before diving into full CPU datapath design, foundational combinational and sequential logic structures were prototyped using **NI Multisim**. A primary milestone was implementing a **4-Bit Universal Shift Register** using generic digital components to master synchronous data paths and bit manipulation.

### Technical Prototyping Constraints
* **Components Used:** 4x `MUX_4_TO_1` Blocks, 4x `D_FF_POSSR` (Positive Edge-Triggered D Flip-Flops), and a 1Hz `DIGITAL_CLOCK` source.
* **Layout Paradigm:** Statically aligned MUX-to-FF columns with short direct connections, saving On-Page Connectors exclusively for global clocking networks and circular rotation feedback loops to maintain a clean schematic layout.

### Control Logic Function Table
The system operations are selected using two digital input constants (S₁, S₀) mapped directly to the MUX select pins (B as MSB, A as LSB):

| S₁ | S₀ | Selected Input | Register Operation | Hardware Implementation Rule |
| :---: | :---: | :------------: | :----------------- | :--------------------------- |
| **0** | **0** | Input 0 | **Synchronous Clear** | All MUX Input 0 pins hardwired directly to Ground (`GND`). |
| **0** | **1** | Input 1 | **Parallel Load** | Parallel external lines (I₃, I₂, I₁, I₀) fed to Input 1. |
| **1** | **0** | Input 2 | **Rotate Right 1-Bit**| Circular right shift: a₃ → a₂ → a₁ → a₀ → a₃. |
| **1** | **1** | Input 3 | **Rotate Left 1-Bit** | Circular left shift: a₃ ← a₂ ← a₁ ← a₀ ← a₃. |

---

## 🏛️ Phase 2: Capstone Architectural Decisions (16-Bit CPU Strategy)

Building on the shifting and feedback concepts mastered in Multisim, the project has officially transitioned into a **custom 16-bit computer architecture**. Below are the architectural design freezes established to keep the design elegant, lightweight, and hardware-efficient.

### 1. Hardware Description Language: Verilog HDL
* **Selection:** Verilog HDL was chosen over VHDL.
* **Justification:** Verilog's C-like syntax accelerates implementation of combinational and sequential blocks. It provides a cleaner learning curve for a first-time processor design while remaining fully compatible with entry-level academic FPGA platforms (e.g., Digilent Basys 3 or Terasic DE10-Lite).

### 2. Standard Word Size: 16-Bit
* **Data Word Size:** All general-purpose registers (GPRs) are exactly 16 bits wide, storing values from `0x0000` to `0xFFFF`.
* **ALU Width:** The Arithmetic Logic Unit natively accepts two 16-bit operands and yields a 16-bit result per execution cycle.
* **Buses:** Internal data buses are unified at a 16-bit width to maintain clean structural routing on the FPGA fabric.

### 3. Register File Sizing: 8 General-Purpose Registers
* **Selection:** An 8-register file model (R₀ through R₇).
* **Justification:** A 4-register architecture (2-bit address fields) creates software-side execution bottlenecks, forcing frequent data "spills" to RAM. An 8-register file strikes the ideal balance—minimizing internal FPGA multiplexer trees while giving assembly programs a comfortable layout of workspaces.

### 4. Instruction Word Bit-Budgeting
The architecture enforces fixed-length **16-bit instruction words** to eliminate multi-cycle instruction fetching overhead. For standard register-to-register ALU instructions (R-Type), the bit fields are explicitly allocated to balance opcode space and register flexibility:

```text
 15      11 10     8 7      5 4      2 1    0
+----------+--------+--------+--------+------+

|  OPCODE  | R_dest | R_srcA | R_srcB | Rsrvd|
| (5 bits) |(3 bits)|(3 bits)|(3 bits)| (2b) |
+----------+--------+--------+--------+------+
```

* **Opcode Field:** 5 bits allows for up to **32 unique operation codes**, easily accommodating our core instruction mix while leaving plenty of headroom for future extensions.
* **Register Address Fields:** 3 register fields (Destination, Source A, Source B) × 3 bits each = **9 bits total** to map across our 8 general-purpose registers.
* **Reserved Bits:** 2 bits are padded as zero for standard ALU operations, leaving architectural space to handle alternate formats (like 5-bit constants for I-Type immediate instructions or 11-bit offsets for J-Type jumps).

---

## 🛠️ Upcoming Roadmap Milestones
1. **Instruction Set Architecture (ISA) Definition:** Finalize the lean 32-slot opcode map (including `LOAD`, `STORE`, `ADD`, `SUB`, and control flow branches).
2. **Register File HDL:** Write and simulate the 8-register file module in Verilog with dual-read and single-write ports.
3. **ALU Design:** Implement the 16-bit math engine, natively incorporating the circular shift logic developed in Phase 1.

