# SystemVerilog Design Guidelines


## Types and Declarations

- Prefer `logic` over `reg` and `wire` in all new code. `logic` works in both procedural and continuous contexts and avoids the classic `reg`/`wire` confusion. Reserve `wire` only when multi-driver resolution is needed (e.g., tri-state buses).
- Prefer ANSI-style module port declarations (ports declared inline in the module header) over the legacy non-ANSI two-section style.
- Always use sized, based literals: `8'd0`, `1'b1`, `16'hFF` — not bare `0`, `1`, `255`.
- Define constants as `localparam` rather than scattering magic numbers through the code.

## Assignments and Processes

- Use `<=` (non-blocking) in `always_ff` and `always @(posedge clk)` for registers.
- Use `=` (blocking) only in `always_comb`, `initial`, or for temporary variables in sequential blocks.
- Never mix blocking and non-blocking assignments to the same variable in one process.
- Use `always_ff` for registered logic; use `always_comb` for combinational logic. Avoid bare `always @(*)`.
- In `always_comb`, every assigned variable must be assigned on all paths to avoid latches.
- In combinatorial processes, do not read a signal on the RHS before it has been assigned on the LHS in that same process — doing so implies a latch or feedback loop.

## Module Port Ordering

- Use a consistent port order: clocks, resets, control/config inputs, data inputs, data outputs, status outputs.
- One signal/port per line.

## Problematic Code Patterns (Avoid These)

### Race Conditions and Sim/Synth Mismatch

- **Never use blocking (`=`) in sequential blocks.** Pipeline and feedback logic with blocking assignments can simulate correctly in one tool order but produce sim/synthesis mismatch or different behavior when blocks run in a different order. Use non-blocking (`<=`) for all registered outputs.
- **Never drive the same variable from more than one `always` block.** Multiple procedural drivers create undefined behavior and tool-dependent results.
- **Never use `#0` delays** to fix races. They add non-determinism and obscure real bugs.
- **Accumulation operators** (`+=`, `++`) are blocking; in sequential logic expand to non-blocking: `cnt <= cnt + 1'b1`.

### full_case and parallel_case

- **Avoid synthesis directives `full_case` and `parallel_case`.** They are comments to simulators but commands to synthesis, causing sim/synthesis mismatch. Use SystemVerilog `unique case` or `priority case` instead; they are language constructs with consistent sim/synth semantics.

### Latches

- **Incomplete case/if infers latches.** In `always_comb`, every assigned variable must be assigned on every path. Add `default:` or ensure all branches assign all outputs.
- **Note:** Even with `default:`, latches can still form if not every branch assigns every variable.

### Multiple Drivers and Combinational Loops

- **Never assign the same signal from multiple `always_comb` blocks.** A generate for-loop that creates multiple `always_comb` blocks driving one signal is illegal; put the loop inside a single `always_comb`.
- **Avoid initial values on `always_comb` outputs**—they create an implicit second driver.
- **Avoid combinational feedback loops** (signals feeding back without a register break); they cause oscillation and tool warnings.

### Signed/Unsigned and Width

- **Mixed signed/unsigned in one expression:** If any operand is unsigned, the whole operation is unsigned; sign-extension fails. Use explicit `$signed()` / `$unsigned()` or match types.
- **Width mismatches** are subtle; use explicit sizing `VALUE[N:0]` or `N'(VALUE)` and heed synthesis "signed to unsigned" / width warnings.

### Mixed Sequential and Combinational

- **Same variable, blocking and non-blocking in one block:** Synthesis error in many tools. Use only one style per variable.
- **Combinational logic with non-blocking in `always_comb`:** Wrong. Non-blocking in combinational logic uses old values (one-cycle lag); use blocking. Use non-blocking only in `always_ff`.

### X-Propagation

- **`if (x_signal)` takes the else branch in simulation**, but synthesis treats X as don't-care — a major sim/synth mismatch source.
- Use four-state types (`logic`) in simulation; be aware that two-state types (`bit`, `int`) mask X bugs.
- Consider X-propagation simulation modes if your tool supports them (e.g., Xcelium `xprop`, VCS `xprop`).

### Tool Pitfalls

- Use `always_comb`, not `always @(*)`; some tools mishandle wildcard sensitivity.
- Avoid arrayed interfaces in ports (ISim/Vivado). Avoid functions/tasks in interfaces (VCS). Avoid interfaces as top-level ports (flattening causes port mismatch).
- Avoid simple names like `length`, `size`, `out`, `in`—they can collide with built-ins.

## Parameters and Types

- Add explicit types to avoid lint warnings: `parameter integer X = 1`, `parameter [15:0] X = 16'd1`, `localparam logic [4:0] STATE_IDLE = 5'd0`.
- Use explicit comparisons for integer params used as booleans: `if (PARAM != 0)` instead of `if (PARAM)`; `(PARAM == 0)` instead of `!PARAM` in ternaries and conditionals.
- For truncation to narrower widths, use explicit sizing: `VALUE[N:0]` or `N'(VALUE)`.

## Case Statements

- Use a `default:` branch in `case`, or use `unique case` to avoid implicit latches and satisfy case-missing-default rules.
- `unique case` asserts all cases are covered and mutually exclusive; the simulator will warn at runtime if violated.
- `priority case` asserts all cases are covered but allows overlap (first match wins).

## Enumerated Types and Typedefs

- Use `typedef enum logic [N:0] { ... }` for FSM states and mode selectors. Specifying the width explicitly prevents width-mismatch lint warnings.
- Use `typedef struct packed { ... }` for bundled signals to avoid manual bit-slicing errors.
- Place shared typedefs in a package.

## Packages

- Use packages for shared types, parameters, and constants.
- Prefer explicit `pkg::symbol` over wildcard `import pkg::*` to avoid namespace collisions.

## FSM Coding Style

- Use a two-process style: one `always_ff` for the state register, one `always_comb` for next-state and output logic.
- Declare states with `typedef enum logic [N:0]` and explicit width.
- In the combinational next-state block, set a default next-state (typically the current state) before the `case` to prevent latches.
- Use `unique case` on the state variable so the simulator warns on unexpected states.

## Clock, Reset, and CDC

- Use consistent naming: `clk`, `rst_n` (or `arst_n` for async reset).
- Document async vs sync reset style per design.
- Never gate clocks with manual combinational logic; use a proper clock-gating cell (ICG).
- **Never sample an asynchronous signal directly.** Use a 2-stage (or more) synchronizer on the receiving clock domain. Single flop = metastability risk.
- **Multi-bit CDC:** Do not pass multiple correlated bits through separate synchronizers; use gray coding or a proper handshake/MUX scheme.
- **Async reset deassertion:** The dangerous moment is release, not assertion. Use synchronizer chains (async assert, sync deassert) to avoid recovery-time violations and metastability.
- Document CDC boundaries and where synchronizers are used.

## Assertions (SVA)

- Use immediate assertions (`assert(expr)`) for sanity checks in procedural code.
- Use concurrent assertions (`assert property (...)`) for protocol and timing checks.
- Guard synthesis-only builds with `` `ifdef SYNTHESIS`` or use bind files so assertions don't pollute synthesis netlists.

## Memory / RAM Inference

- Write single-port and dual-port RAM using the vendor-recommended coding template so that synthesis maps to block RAM (FPGA) or SRAM (ASIC).
- Avoid resets on memory arrays — reset prevents block RAM inference on most FPGA families.
- Use `(*ram_style = "block"*)` or equivalent attribute when the tool needs a hint.

## Generate Blocks

- Always label generate blocks: `gen_slice : for (genvar g = 0; ...)`. Unlabeled generates produce messy hierarchical names and make debug harder.
- Be aware of generate scope: variables declared inside a generate block are per-instance.

## Synthesis-Safe Coding

- `initial` blocks are ignored by ASIC synthesis. FPGA tools may use them for RAM initialisation, but do not rely on them for register reset values.
- Never use `force`/`release`, `deassign`, or procedural continuous `assign` in synthesizable code.
- Avoid `real`, `time`, `string`, and other non-synthesizable types in RTL.

## Instances and Interfaces

- In generate blocks, prefer `u_` or `i_` prefix on instances with indexes, e.g. `u_slice_i[g]`.
- Use `modport` in interfaces to separate DUT and TB views; keep port lists in `modport` declarations.

## Files and Formatting

- Use spaces, not tabs, for indentation.
- One declaration per line.
- One module per file; file name matches module name.
