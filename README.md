<div align="center">

# Hardware Programming

**Digital design coursework — building a MIPS processor from the register file up.**

Technical University of Cluj-Napoca

![HDL](https://img.shields.io/badge/HDL-VHDL-1B3A52?style=flat-square)
![Target](https://img.shields.io/badge/Target-FPGA-42708F?style=flat-square)
![Architecture](https://img.shields.io/badge/ISA-MIPS_32--bit-B86A10?style=flat-square)

</div>

---

## Contents

| Project | What it builds |
|---|---|
| [MIPS-CPU-Design](MIPS-CPU-Design) | A working 32-bit MIPS processor — datapath, control unit, instruction and data memory |

<br>

## MIPS CPU Design

A processor is not a mysterious object. It is a handful of components wired in a loop: something that remembers where you are, something that holds instructions, something that holds numbers, something that does arithmetic, and a decoder that tells the rest what to do this cycle.

Building one is the exercise that makes every abstraction above it — assembly, compilers, operating systems — stop being magic.

### The datapath

![MIPS datapath](docs/datapath.svg)

Five things happen to every instruction, in order:

| Stage | What happens |
|---|---|
| **IF** — instruction fetch | The PC addresses instruction memory; the 32-bit word comes out. PC advances by 4. |
| **ID** — instruction decode | The opcode splits into fields. Register numbers index the register file, which reads two values at once. |
| **EX** — execute | The ALU does the arithmetic, or computes a memory address, or compares two registers for a branch. |
| **MEM** — memory access | Only `lw` and `sw` do anything here. Every other instruction passes straight through. |
| **WB** — write back | The result returns to the register file. |

The control unit sits underneath all of it. It reads nothing but the opcode and produces every signal the datapath needs — which multiplexer input to select, whether memory writes, whether the register file writes, what the ALU should do. Change the opcode and the same wires carry a completely different instruction.

### Instruction formats

![MIPS instruction formats](docs/formats.svg)

Every MIPS instruction is exactly 32 bits. That single decision is why the fetch stage is trivial: the next instruction is always at PC + 4, with no decoding required to find out how long the current one was. x86 instructions vary from 1 to 15 bytes, and the decoder that has to cope with that is one of the most complex parts of the chip.

The three formats share their first six bits. The processor reads the opcode, learns which format it is holding, and only then knows how to interpret the remaining 26 bits.

### The register file

Thirty-two registers, each 32 bits, with two read ports and one write port — so the ALU can receive both operands in the same cycle it needs them.

| Register | Convention |
|---|---|
| `$zero` | Hardwired to 0. Writes are discarded. |
| `$at` | Reserved for the assembler |
| `$v0`–`$v1` | Return values |
| `$a0`–`$a3` | Arguments |
| `$t0`–`$t9` | Temporaries, caller-saved |
| `$s0`–`$s7` | Saved, callee-saved |
| `$sp`, `$ra` | Stack pointer, return address |

`$zero` is worth dwelling on. It costs a register but removes the need for several instructions: `move $t0, $t1` is just `add $t0, $t1, $zero`, and comparing against zero needs no immediate. A constant wired into the register file buys instruction-set simplicity.

### Control signals

| Signal | Effect when asserted |
|---|---|
| `RegWrite` | The register file commits a write this cycle |
| `RegDst` | Destination is `rd` (R-type) rather than `rt` (I-type) |
| `ALUSrc` | Second ALU operand is the sign-extended immediate, not a register |
| `MemRead` | Data memory drives its output |
| `MemWrite` | Data memory commits a write |
| `MemtoReg` | Write-back value comes from memory rather than the ALU |
| `Branch` | With the ALU zero flag, redirects the PC |
| `ALUOp` | Tells the ALU decoder which operation family to select |

Getting these right is most of the work. A datapath with a wrong `RegDst` produces a CPU that runs, reports no error, and writes every result to the wrong register.

### Simulating before synthesising

Synthesis is slow and an FPGA gives you almost nothing to look at when a design is wrong. Simulation is where debugging actually happens.

The productive order:

1. Test each component alone — register file, ALU, sign extender — with a testbench that drives known inputs and checks known outputs.
2. Wire the datapath together and run one instruction. Watch it in the waveform viewer.
3. Add instructions one at a time. Each new opcode is a new control-signal path, and each is a new chance to be wrong.
4. Only then synthesise.

A waveform viewer shows every signal at every clock edge. When the wrong value lands in a register, the waveform shows exactly which cycle and which wire carried it — which no amount of staring at HDL source will.

### Build and run

```bash
# Simulate
ghdl -a *.vhd && ghdl -e cpu_tb && ghdl -r cpu_tb --wave=cpu.ghw
gtkwave cpu.ghw

# Or open the project in Vivado / Quartus and run the testbench there
```


<br>

## What I still need from you

The sections above are architecture — true of any MIPS implementation. Three things I could not verify, because GitHub blocks automated access to repository subdirectories:

| Unknown | Why it matters |
|---|---|
| VHDL or Verilog | Badge, build commands, and file extensions all change |
| Single-cycle, multi-cycle, or pipelined | If pipelined, this README is missing its most interesting section — hazards, forwarding and stalls |
| Target board | Basys 3, Nexys, DE10 — determines the constraints file and pin mapping |

Also: only `MIPS-CPU-Design` appears in the contents table, because it is the only folder I know about. If there are others, they belong there.
