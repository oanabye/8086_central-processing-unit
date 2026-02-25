# 🖥️ MIPS Pipelined Processor — Sum of Even Numbers

A fully functional **5-stage pipelined MIPS processor** implemented in VHDL, targeting the **Basys3 / Nexys FPGA** board. The processor executes a custom MIPS assembly program that computes the **sum of even numbers** from a memory array, with real-time output displayed on the 7-segment display.

---

## 📐 Architecture Overview

The design follows the classic **5-stage MIPS pipeline**:

```
IF  →  ID  →  EX  →  MEM  →  WB
```

| Stage | Module | Description |
|-------|--------|-------------|
| **IF** | `IFetch` | Instruction Fetch — PC register, ROM, branch/jump MUX |
| **ID** | `REG_ID` | Decode — Register File (32×32-bit), sign/zero extension |
| **EX** | `REG_EX` | Execute — ALU, branch address computation, `rWA` selection |
| **MEM** | `MEM` | Memory Access — 64-entry data RAM (sync write, async read) |
| **WB** | *(top-level)* | Write-Back — MemtoReg MUX, writeback to Register File |
| **UC** | `UC` | Main Control Unit — decodes opcode, generates control signals |
| **Top** | `test_env` | Top-level — pipeline registers, I/O, SSD driver, MPG |

---

## 🧠 Supported Instructions

| Category | Instructions |
|----------|-------------|
| **R-Type** | `add`, `sub`, `and`, `or`, `xor`, `sll`, `srl`, `sra` |
| **I-Type** | `addi`, `andi`, `lw`, `sw`, `beq`, `bne`, `bgtz` |
| **J-Type** | `j` |

---

## 📋 Loaded Program — Sum of Even Numbers

The ROM inside `IFetch` contains a hand-assembled MIPS program that:

1. Initialises a counter `$1 = 0`, a sum accumulator `$3 = 0`, and a memory index `$4 = 0`
2. Sets the iteration limit `$2 = 10` (processes 10 elements)
3. Loops through the data memory array (`lw` from byte address `$4`)
4. Tests parity with `andi $6, $5, 1` — skips odd numbers
5. Accumulates even numbers into `$3`
6. On loop exit, stores the final result: `sw $3, 80($0)`

```mips
addi  $2, $0, 10        # loop limit = 10
add   $1, $0, $0        # counter = 0
add   $3, $0, $0        # sum = 0
add   $4, $0, $0        # memory index = 0

loop:
  beq   $1, $2, end     # if counter == 10, exit
  lw    $5, 0($4)       # load MEM[$4]
  andi  $6, $5, 1       # test LSB (odd/even)
  beq   $6, $0, add_sum # if even, branch
  addi  $1, $1, 1       # counter++
  addi  $4, $4, 4       # advance to next word
  j     loop

add_sum:
  add   $3, $3, $5      # accumulate even value
  addi  $1, $1, 1
  addi  $4, $4, 4
  j     loop

end:
  sw    $3, 80($0)       # store result to MEM[80]
```

> **Data memory** is preloaded with values 1–17. The even values (2, 4, 6, 8, 10, 12, 14, 16) sum to **72 (0x48)**.

---

## 🗂️ File Structure

```
├── IFetch.vhd      # Instruction Fetch (PC, ROM, branch/jump MUX)
├── REG_ID.vhd      # Decode stage (Register File, immediate extension)
├── REG_EX.vhd      # Execute stage (ALU, ALU Control, branch address)
├── MEM.vhd         # Data Memory (sync write, async read)
├── UC.vhd          # Control Unit (opcode → control signals)
├── SSD.vhd         # 7-Segment Display driver (8-digit multiplexed)
├── MPG.vhd         # Monostable Pulse Generator (button debouncer)
└── test_env.vhd    # Top-level: pipeline registers + board I/O
```

---

## 🔌 I/O Mapping (Basys3)

| Signal | Function |
|--------|----------|
| `CLK` | 100 MHz system clock |
| `BTN(0)` | Manual clock step (debounced via MPG) |
| `BTN(1)` | Synchronous reset (active high) |
| `SW[7:5]` | Select value shown on 7-segment display |
| `LED[10:8]` | `ALUOp[2:0]` |
| `LED[7:0]` | Control signals: RegDst, ExtOp, ALUSrc, Branch, Jump, MemWrite, MemtoReg, RegWrite |
| `an[7:0]` | 7-Seg anode select |
| `cat[6:0]` | 7-Seg cathode segments |

### 7-Segment Display — SW[7:5] Selector

| `SW[7:5]` | Displayed Value |
|-----------|----------------|
| `000` | Current Instruction |
| `001` | PC + 4 |
| `010` | RD1 (ALU operand A, post-ID/EX register) |
| `011` | RD2 (ALU operand B, post-ID/EX register) |
| `100` | Extended Immediate |
| `101` | ALU Result |
| `110` | Memory Read Data |
| `111` | Write-Back Data |

---

## ⚙️ Pipeline Registers

All four pipeline registers are implemented in a single clocked process inside `test_env`:

| Register | Signals Latched |
|----------|----------------|
| **IF/ID** | `Instruction`, `PC4` |
| **ID/EX** | All control signals + `RD1`, `RD2`, `Ext_Imm`, `func`, `sa`, `rd`, `rt`, `PC4` |
| **EX/MEM** | `Zero`, `BranchAddress`, `ALURes`, `RD2`, `WA` + control signals |
| **MEM/WB** | `MemData`, `ALUResOut`, `WA` + control signals |

---

## 🚀 Getting Started

### Prerequisites
- Vivado Design Suite 2020.x or later
- Basys3 (or compatible Nexys) FPGA board

### Steps

```bash
# 1. Clone the repository
git clone https://github.com/<your-username>/mips-pipeline-vhdl.git

# 2. Open Vivado → New Project → Add all .vhd source files
# 3. Set test_env.vhd as the top-level entity
# 4. Add the Basys3 XDC constraints file
# 5. Run Synthesis → Implementation → Generate Bitstream → Program Device
```

### Running the Program
1. Press **BTN(1)** to reset the processor (PC → 0)
2. Press **BTN(0)** repeatedly to step through clock cycles
3. Use **SW[7:5]** to inspect pipeline signals on the display
4. After ~30 steps, `MEM[80]` holds the result — display `0x00000048` (**72**)

---

## 🔍 Key Design Notes

- **Branch resolution** happens in the MEM stage: `PCSrc = Branch_EX_MEM AND Zero_EX_MEM`
- **Jump target**: `PC4_IF_ID[31:28] & Instr_IF_ID[25:0] & "00"`
- **No hazard detection unit** — NOPs are hand-inserted in the ROM to resolve data and control hazards
- **Register file** writes on the **falling edge** and reads asynchronously, avoiding same-cycle RAW hazards
- **Data memory** has **synchronous write** and **asynchronous read**

---

## 📚 Skills Demonstrated

- RTL design and simulation in **VHDL**
- 5-stage **pipelined processor** architecture (datapath + control path)
- **MIPS ISA** — R/I/J-type instruction encoding
- **FPGA implementation** with Xilinx Vivado toolchain
- Hand-written **MIPS assembly** with manual hazard management
- Hardware debugging on real FPGA hardware

---

## 👩‍💻 Author

**Oana** — [GitHub Profile](https://github.com/oanabye)
