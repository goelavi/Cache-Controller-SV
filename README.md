# Cache-controller
# Parameterized 2-Way Set-Associative Cache Controller

A fully parameterized, synthesizable 2-way set-associative cache controller written in SystemVerilog, complete with a custom (non-UVM) layered verification environment. The design supports configurable address width, data width, cache size, and block size, and implements a complete read/write/hit/miss/eviction/fill lifecycle through a dedicated finite state machine.

---

## Overview

This project implements a classic write-back, write-allocate cache with true LRU replacement, built from independent, reusable RTL blocks and validated using a self-checking, constrained-random verification environment simulated on **EDA Playground**.

The goal was to build both the design and its verification environment from scratch — without UVM — to get hands-on experience with the full hardware design and verification flow: RTL architecture, FSM-driven control logic, testbench construction, reference-model-based scoreboarding, and assertion-based protocol checking.

---

## Architecture

The cache is built from the following independently parameterized modules:

| Module | Description |
|---|---|
| `address_decoder.sv` | Splits a CPU address into tag, index, and offset fields |
| `tag_array.sv` | Stores per-set, per-way tag, valid, and dirty bits |
| `data_array.sv` | Stores cache block data, supports word-level and block-level read/write |
| `comparator.sv` | Compares incoming tag against both ways to detect hit/miss |
| `lru_logic.sv` | Tracks and updates the least-recently-used way per set |
| `memory_interface.sv` | Protocol-agnostic request/ready handshake to external memory |
| `cache_controller.sv` | Core FSM coordinating all of the above across a cache access |
| `top.sv` | Structural top-level module wiring all submodules together |

### Cache Configuration (default)

| Parameter | Value |
|---|---|
| `ADDR_WIDTH` | 32 |
| `DATA_WIDTH` | 32 |
| `CACHE_SIZE` | 8192 bytes |
| `BLOCK_SIZE` | 32 bytes |
| `NUM_WAYS` | 2 |
| `NUM_SETS` | 128 (derived) |
| `INDEX_BITS` | 7 (derived) |
| `OFFSET_BITS` | 5 (derived) |
| `TAG_BITS` | 20 (derived) |

All derived values are computed at elaboration time using `$clog2`-based `localparam`s — nothing is hardcoded, so the cache can be resized simply by changing the top-level parameters.

### FSM States

```
IDLE → CHECK_TAG → CACHE_HIT ─────────────→ RESPOND → IDLE
                 └→ CACHE_MISS → WRITE_BACK ─┐
                               └→ MEMORY_READ ┴→ CACHE_FILL → CACHE_HIT
```

- **IDLE** – waits for a CPU request
- **CHECK_TAG** – performs a single-cycle tag comparison
- **CACHE_HIT** – services a hit (read or write), updates LRU
- **CACHE_MISS** – selects a victim way and checks if write-back is needed
- **WRITE_BACK** – evicts a dirty line back to memory
- **MEMORY_READ** – fetches the requested line from memory
- **CACHE_FILL** – installs the new line, then re-enters `CACHE_HIT` to service the original request ("fill-then-service")
- **RESPOND** – asserts `cpu_ready_o` for one cycle to signal completion

---

## Verification Environment

A layered, class-based SystemVerilog testbench (no UVM) was built to verify the RTL:

```
Generator → Driver → DUT → Monitor → Scoreboard
                              ↓
                      Reference Model
```

- **Reference model** — an independent, software-based golden model of cache behavior (tag/index math, hit/miss, LRU, dirty tracking) that mirrors the DUT without depending on its internals.
- **Memory model** — a behavioral external memory with configurable random latency (2–6 cycles) and single-outstanding-request enforcement, used to realistically emulate write-back and fill timing.
- **Scoreboard** — automatically compares DUT outputs against the reference model on every transaction, flagging mismatches in data, hit/miss status, and replacement way.
- **Assertions (SVA)** — bound non-intrusively to the DUT (via `bind`, without modifying RTL) to check protocol-level properties such as:
  - An invalid line never produces a hit
  - A cache hit never issues a memory request
  - A dirty eviction is always followed by a write-back before the subsequent read
  - The FSM never enters an illegal state
- **Constrained-random stimulus** — address generation is biased toward a small working set to reliably exercise hits, misses, and evictions instead of relying on a fully random, always-cold address space.
- **Functional coverage** — tracked across operations, replacement events, dirty-bit transitions, and FSM state transitions.

Simulations were run on **EDA Playground**, producing automated scoreboard pass/fail results and mismatch diagnostics for both directed and randomized test suites.

---

## Repository Structure

```
├── rtl/
│   ├── address_decoder.sv
│   ├── tag_array.sv
│   ├── data_array.sv
│   ├── comparator.sv
│   ├── lru_logic.sv
│   ├── memory_interface.sv
│   ├── cache_controller.sv
│   └── top.sv
├── tb/
│   ├── cache_pkg.sv
│   ├── cpu_if.sv
│   ├── mem_if.sv
│   ├── memory_model.sv
│   ├── reference_model.sv
│   ├── generator.sv
│   ├── driver.sv
│   ├── monitor.sv
│   ├── scoreboard.sv
│   ├── assertions.sv
│   ├── coverage.sv
│   ├── environment.sv
│   ├── test_basic.sv
│   ├── test_random.sv
│   └── tb_top.sv
└── README.md
```

---

## Coding Standards

- Fully synthesizable SystemVerilog (no simulation-only constructs in RTL)
- `always_ff` for sequential logic, `always_comb` for combinational logic
- Enum-typed (`typedef enum`) FSM states — no magic numbers
- No latches — every `always_comb` branch fully assigned
- Active-high synchronous reset (`rst`), single clock (`clk`)
- No hardcoded bit widths — all widths derived from `parameter`/`localparam`

---

## Tools Used

- **HDL:** SystemVerilog
- **Simulation:** EDA Playground

---

## Notes

- The design is fixed at `NUM_WAYS = 2` for this project, but the architecture and parameterization approach are written to generalize to N-way set-associative caches.
- The memory interface is intentionally protocol-agnostic (a simple request/ready handshake) rather than tied to AXI/AHB, to keep the controller portable across different memory subsystems.
