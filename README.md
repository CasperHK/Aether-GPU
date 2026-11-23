# Aether-GPU: Modern RISC-V GPGPU in Veryl

[![Veryl](https://img.shields.io/badge/HDL-Veryl-blue.svg)](https://veryl-lang.org/)
[![License](https://img.shields.io/badge/License-Apache--2.0-green.svg)](LICENSE)
[![Verification](https://img.shields.io/badge/Simulator-Verilator-orange.svg)](https://www.veripool.org/verilator/)

**Aether-GPU** is an open-source, hardware-efficient **RISC-V GPGPU / AI Accelerator** architecture. Drawing architectural inspirations from pioneer open-source GPGPU projects like **Vortex** and **Ventus (乘影)**, Aether-GPU is engineered from the ground up using **Veryl**—a modern, type-safe HDL compiling to clean SystemVerilog—and an **AI-assisted hardware development workflow**.

---

## Key Highlights

- **Modern HDL Stack:** Built exclusively in [Veryl](https://veryl-lang.org/), eliminating legacy SystemVerilog pitfalls (implicit latches, ambiguous types) while retaining zero-cost abstraction and full EDA tool compatibility.
- **AI-Assisted Workflow (`LLM + HDL Rules`):** Strict guardrails (`AI_HDL_RULES.md`) enable LLM coding agents to rapidly generate, lint, and refine hardware modules with compiler-in-the-loop diagnostics.
- **Decoupled Scalar + Vector Execution:**
  - **Scalar Core:** Compact `RV32I` subset handling control flow, program counter, and memory addressing.
  - **Vector/SIMT Engine:** Specialized AI/SIMT pipeline accelerating matrix arithmetic and multi-lane dot products.
- **Cycle-Accurate Reference Model:** Python-based Golden Model (`sim/golden_model.py`) defining hardware behavior prior to RTL synth and simulation.

---

## Architectural Evolution

Aether-GPU follows a phased roadmap from a minimal vector coprocessor to a full SIMT Multi-Streaming Multiprocessor (SM) cluster:

```
+-----------------------------------------------------------------------------------+
|                        Phase 1: Minimal Scalar + AI Vector Core                   |
|  [RV32I Controller] ---> [AI_LOAD_VEC / AI_DOT4 Compute] ---> [Byte-Addressed VRAM] |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
|                           Phase 2: Single-SM SIMT & Warp Pipeline                 |
|  [Warp Scheduler] ---> [Active Mask / Branch Stack] ---> [Multi-Lane Vector Unit] |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
|                     Phase 3: Banked Scratchpad & Memory Coalescing                |
|  [Multi-Bank Local SRAM] <---> [Memory Coalescer] <---> [AXI4 / Global Interconnect] |
+-----------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
|                           Phase 4: Multi-SM Cluster & Toolchain                   |
|  [SM Cluster + Mesh NoC] <---> [OpenCL / LLVM Compiler Stack / FPGA Prototype]    |
+-----------------------------------------------------------------------------------+
```

---

## Custom ISA & Execution Model (Phase 1 MVP)

In addition to standard `RV32I` base instructions (`ADDI`, `ADD`, `LW`, `SW`, `BEQ`, `JAL`), Aether-GPU reserves the RISC-V `custom-0` opcode space (`7'h0B`) for high-throughput AI operations:

| Instruction | Format / Types | Description |
| :--- | :--- | :--- |
| `AI_LOAD_VEC` | `custom-0`, `funct3 = 0x0` | Loads 4 contiguous `int8` elements from memory address `rs1` into vector register slot `v[rd & 3]`. |
| `AI_DOT4` | `custom-0`, `funct3 = 0x1` | Computes 4-element signed dot product ($\sum_{i=0}^{3} a_i \cdot b_i$) between `v[rs1 & 3]` and `v[rs2 & 3]`, storing `int32` result in scalar `rd`. |
| `AI_HALT` | `custom-0`, `funct3 = 0x7` | Terminates simulation execution in testbenches. |

---

## Repository Structure

```text
.
├── README.md               # Project overview and documentation
├── AI_HDL_RULES.md         # Guardrails for LLM agents generating Veryl code
├── Veryl.toml              # Veryl project and package manifest
├── docs/                   # ISA specification and architecture diagrams
│   └── isa_custom.md
├── src/                    # Veryl Hardware Source Files
│   ├── vram.veryl          # Byte-addressable SRAM / Scratchpad model
│   ├── rv_decode.veryl     # Instruction decoder + custom opcode parser
│   ├── regfile.veryl       # 32x32-bit scalar register file
│   ├── ai_dot4.veryl       # 4-lane vector multiply-accumulate (MAC) unit
│   └── rv_ai_core.veryl    # Core top-level integration
├── sim/                    # Verification Environment
│   ├── golden_model.py     # Python reference simulator & test vector generator
│   └── tb_vram.cpp         # Verilator C++ testbench for memory modules
└── tools/                  # Build helper scripts & open-source EDA flow (Yosys/OpenROAD)
```

---

## Getting Started

### Prerequisites

- **Rust & Cargo** (`>= 1.75`)
- **Veryl Compiler & LSP** (`veryl`, `veryl-ls`)
- **Verilator** (`>= 5.000`)
- **Python** (`>= 3.9`)

Install the Veryl toolchain:

```bash
cargo install veryl veryl-ls
```

### 1. Build RTL with Veryl

Check code formatting, run linting, and compile Veryl source files into synthesizable SystemVerilog:

```bash
# Lint Veryl source files
veryl lint

# Build SystemVerilog outputs
veryl build
```

### 2. Run the Golden Model

Verify target test vectors and generate reference register/memory state dumps:

```bash
python3 sim/golden_model.py
```

Expected output:
```text
Halted in 9 cycles.
x10 (dot4 result) = 7
mem[0x200]        = 7
```

### 3. Simulate with Verilator

Compile the generated SystemVerilog code and run cycle-accurate verification:

```bash
verilator --cc --exe --build -Wall \
  -Isrc \
  src/vram.veryl \
  sim/tb_vram.cpp \
  --top-module vram

./obj_dir/Vvram
```

---

## AI-Assisted Hardware Development Workflow

This repository relies on a human-AI collaborative design flow:

1. **Context Guardrails:** All AI coding sessions must adhere to `AI_HDL_RULES.md` (no inferred latches, explicit bit-widths, deterministic reset logic, clear `valid`/`ready` handshakes).
2. **Compiler-in-the-Loop:** Every code iteration generated by AI must pass `veryl lint` and `veryl build`. Compiler diagnostics are immediately fed back to the AI agent until zero errors remain.
3. **Golden Model Co-Verification:** RTL functionality must match `sim/golden_model.py` before merging PRs.

---

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
