# FIFO Design & Verification

A synchronous FIFO (First-In-First-Out) buffer written in SystemVerilog, verified with a class-based testbench (generator/driver/monitor/scoreboard architecture) in Vivado XSim.

## Design

`fifo.sv` implements a **16-entry, 8-bit-wide synchronous FIFO**:

| Signal | Direction | Width | Description |
|--------|-----------|-------|--------------|
| `clk`   | input  | 1   | Clock |
| `rst`   | input  | 1   | Synchronous reset |
| `wr`    | input  | 1   | Write enable |
| `rd`    | input  | 1   | Read enable |
| `din`   | input  | 8   | Write data |
| `dout`  | output | 8   | Read data |
| `full`  | output | 1   | High when FIFO holds 16 entries |
| `empty` | output | 1   | High when FIFO holds 0 entries |

Writes are blocked while `full`, reads are blocked while `empty`. Depth tracked internally via a 5-bit counter; read/write pointers wrap through a 16-entry memory array.

## Verification

`fifo_tb.sv` is a self-checking, class-based testbench:

- **`transaction`** — randomized operation (`oper`: write/read, ~50/50 weighted distribution) plus data fields.
- **`generator`** — produces `count` randomized transactions, paced by an event handshake with the scoreboard.
- **`driver`** — drives write/read sequences onto the DUT's pin-level interface via a virtual interface (`fifo_if`).
- **`monitor`** — passively samples DUT signals after each operation completes.
- **`scoreboard`** — maintains a reference queue (`din[$]`) mirroring expected FIFO contents; checks every read against the expected value (FIFO order — first written, first read) and flags mismatches.
- **`environment`** — wires the above together, runs `pre_test` → `test` → `post_test`, and reports a final mismatch count before calling `$finish()`.

### Handshake architecture

Unlike a driver-paced generator, this testbench paces the generator from the **scoreboard**:

```
generator → driver → DUT → monitor → scoreboard → (event) → generator
```

Each transaction is fully retired (driven, sampled, and checked) before the next one is generated — simpler to reason about, at the cost of some throughput.

## Running the simulation

In Vivado:

1. Add `fifo.sv` to the design sources and `fifo_tb.sv` to the simulation sources.
2. Set the top-level simulation module to `fifo_tb`.
3. **Increase the simulation runtime** — XSim's default of `1000ns` is not enough to complete a full transaction set. In Simulation Settings → Simulation tab, set `xsim.simulate.runtime` to `2000000ns` (or higher, scaled to `count`), or run `run -all` from the Tcl console.
4. Launch behavioral simulation.

Adjust the number of transactions via `env.gen.count` in the `fifo_tb` module before running.

## Sample result

```
Error Count : 0
$finish called at time : 690 ns
```

All writes and reads verified against the reference model with zero mismatches; FIFO ordering confirmed correct.

## Known coverage gaps

- `full` flag behavior is not exercised unless `count` and/or the `oper` distribution are tuned to issue more than 16 consecutive writes.
- `empty`-during-read behavior is not exercised unless reads are attempted with no prior writes (or after fully draining the FIFO).

## Repo structure

```
.
├── fifo.sv                          # DUT
├── fifo_tb.sv                       # Testbench
└── fifo_verification_summary.md     # Latest run log/summary
```
