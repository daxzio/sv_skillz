# SystemVerilog design guidelines

Correctness rules for synthesizable SystemVerilog RTL, written for AI coding
agents. Naming and formatting conventions live in `sv_preferences.md`.
Testbench and verification code are out of scope.

## How to apply these rules

- These rules always apply to synthesizable RTL. If they conflict with
  `sv_preferences.md`, correctness wins.
- New files follow both this file and `sv_preferences.md`. When editing an
  existing codebase, apply the correctness rules to the lines you touch but
  match the surrounding style, even where it differs from the preferences
  file, unless explicitly asked to restyle.
- After writing or modifying RTL, lint it and iterate:
  `verilator --lint-only -Wall <files>` (or Verible lint, or slang). Fix
  every warning, or attach a one-line justification to a narrowly scoped
  waiver. Never add blanket waivers.
- The templates at the end are the canonical shapes for common structures.
  Start from them. They use neutral signal names; apply `sv_preferences.md`
  naming on top.

## Types and declarations

- Use `logic` for all signals. Use `wire` only where multi-driver resolution
  is genuinely needed (tri-state buses) and for `inout` ports.
- Use ANSI-style module headers (ports declared inline), never the legacy
  two-section style.
- Use sized, based literals in fixed-width expressions: `8'd0`, `1'b1`,
  `16'h00FF`. Where the width is a parameter, never hardcode a size — use
  `'0`, `'1`, `{W{1'b0}}`, or a width cast `W'(expr)`. Bare integers are
  fine as loop iterators, genvars, and values of integer-typed parameters
  (`parameter int N = 4`).
- Name constants with `localparam` instead of scattering magic numbers.
  Derive widths rather than hand-computing them:
  `localparam int AW = $clog2(DEPTH);`.
- Never put a declaration initializer (`logic x = 1'b0;`) on a variable
  driven by an `always_comb` or `always_ff` block — it counts as a second
  driver (IEEE 1800-2017 9.2.2.2).

## Processes and assignments

- `always_ff` for registered logic, `always_comb` for combinational logic.
  Never bare `always @(*)` or `always @(posedge clk)`.
- In `always_ff`, use non-blocking (`<=`) for every variable that leaves the
  block. Blocking (`=`) is allowed only for variables declared inside the
  block, and only when written before read. Never use both styles on the
  same variable.
- In `always_comb`, use blocking (`=`) only. Non-blocking in combinational
  logic reads stale values.
- Never drive the same variable from more than one process.
- `++` and `+=` are blocking operators; in registered logic expand them:
  `cnt <= cnt + 1'b1;`.
- No delays in RTL: no `#0` race "fixes", no `#` delays, no `wait`, no event
  controls beyond the process sensitivity.

## Latch prevention

- The first statements of every `always_comb` assign a default value to
  every variable the block drives; later branches override. This is the
  required pattern — it prevents latches structurally rather than
  case-by-case.
- A `default:` branch alone is not sufficient: a latch still forms when any
  branch assigns only some of the block's outputs.
- A variable assigned inside an `always_comb` must be written before it is
  read within that block. Read-before-write means the previous value is
  stored — a latch or a combinational feedback loop.
- No combinational feedback: every loop must be broken by a register.

## Case statements

- Prefer `unique case` when exactly one branch should match. It adds a
  runtime check for no-match and multi-match — but the check only fires on
  stimulus that hits the violation, so it is not a substitute for the
  default-assignment pattern above.
- Use `priority case` when branches deliberately overlap; first match wins.
- Do not add a `default:` inside a `unique case` over a complete enum — it
  silences the no-match check. Latch safety comes from the
  default-assignment pattern, not from the default branch.
- A plain `case` (neither unique nor priority) must have a `default:`.
- Never use `casex`. Use `casez` only for genuine wildcard matching such as
  address decoding; prefer `case () inside` for ranges and value sets.
- Never use the `full_case` / `parallel_case` pragmas. They are comments to
  simulators but commands to synthesis, a direct sim/synth mismatch source.
  `unique` and `priority` are the language-level replacements.

## Expressions, width, and sign

- `===` and `!==` are simulation-only operators; never use them in RTL.
- Mixing signed and unsigned operands makes the whole expression unsigned
  and breaks sign-extension. Match the types, or use `$signed()` /
  `$unsigned()` explicitly.
- Truncate explicitly with `VALUE[N-1:0]` or `N'(VALUE)`. Treat every
  synthesis width warning as a bug until shown otherwise.
- Integer parameters used as booleans get explicit comparisons:
  `(PARAM != 0)`, not `if (PARAM)`; `(PARAM == 0)`, not `!PARAM`.
- `/` and `%` only by constant powers of two (shift and mask). Any other
  division is an explicit design decision — a pipelined divider or shared
  IP — never an inline operator.
- `*` synthesizes to multiplier hardware. Size the operands deliberately
  and consider pipelining wide products.

## Loops and generate

- Synthesizable loops need bounds that are static at elaboration and full
  unrolling. No `while`, no data-dependent bounds or exits. An iteration
  count that depends on data is an FSM, not a loop.
- Label every generate block on its `begin`:

  ```systemverilog
  for (genvar g = 0; g < N; g++) begin : gen_slice
    ...
  end
  ```

- The single-driver rule applies across generate iterations: a loop that
  creates N `always_comb` blocks driving one signal is illegal. Put the
  loop inside a single `always_comb`.
- Variables declared inside a generate block are per-instance.
- Instances inside a generate loop are indexed by the hierarchy
  (`gen_slice[0].u_slice`); never put an index in the instance name itself.
  Instance naming per `sv_preferences.md`.

## Parameters, types, and packages

- Give parameters explicit types: `parameter int N = 4`,
  `parameter logic [15:0] INIT = 16'h0000`,
  `localparam logic [4:0] IDLE_CODE = 5'd0`.
- Override parameters by name only: `#(.N(8))`. Never positional parameter
  lists. Never `defparam`.
- Declare FSM states and mode selectors as
  `typedef enum logic [W-1:0] { ... }` with explicit width and explicitly
  sized values.
- Use `typedef struct packed` for bundled signals instead of manual
  bit-slicing.
- Put shared types, parameters, and constants in packages. Reference them
  as `pkg::symbol`; avoid wildcard `import pkg::*`.

## Modules, ports, and instantiation

- Port order: clocks, resets, control/config inputs, data inputs, data
  outputs, status outputs. One port per line.
- Connect ports by name only: `.clk(clk)`. Never positional connections.
  Do not use `.*`.
- Default to flat ports for synthesizable RTL. If the project already uses
  interfaces: access them only through `modport`s, never place an interface
  on a top-level or synthesis-boundary port, and treat arrayed interface
  ports as a portability risk.
- Optional per project: `` `default_nettype none`` at the top of each file
  (restoring `` `default_nettype wire`` at the end) makes typo-created
  implicit nets an error; otherwise rely on the linter's implicit-net
  warning.

## Clock, reset, and CDC

- Default reset policy unless the project states otherwise: active-low
  `rst_n`, asynchronous assertion, synchronous de-assertion via a reset
  synchronizer per clock domain. Reset all control-path registers; datapath
  registers may stay unreset where the flow permits. Never reset memory
  arrays.
- Never gate a clock with hand-built combinational logic; use the library
  clock-gating cell (ICG).
- Never sample an asynchronous signal directly. Use two or more
  synchronizer flops in the destination domain, marked with the vendor
  attribute (`ASYNC_REG` on AMD/Xilinx, equivalents elsewhere) or wrapped
  in a dedicated synchronizer module the CDC tool recognizes.
- Synchronize a signal in exactly one place and fan out the synchronized
  copy. Two synchronizers on the same source can disagree for a cycle.
- Multi-bit crossings never go through independent single-bit
  synchronizers. Use gray-coded counters, a qualified handshake, or an
  async FIFO.
- Document every crossing: source domain, destination domain, mechanism.

## X-propagation

- `if (x_signal)` takes the else branch in simulation while synthesis
  treats X as don't-care — a major sim/synth mismatch source.
- Keep RTL signals four-state (`logic`); two-state `bit` and `int` mask X
  bugs (loop iterators excepted).
- Enable the simulator's X-propagation mode where available (VCS and
  Xcelium both have one).

## Assertions

- Use immediate assertions for sanity checks inside procedural code, and
  concurrent `assert property` for protocol and timing rules.
- Keep assertions out of synthesis: wrap them in `` `ifndef SYNTHESIS`` …
  `` `endif``, or place them in bind files. Most synthesis tools ignore
  SVA, but do not rely on that.

## Synthesis-safe constructs

- `initial` blocks are ignored or rejected by ASIC synthesis; FPGA flows
  may honour them for RAM and register init. Portable RTL takes reset
  values from the reset logic, not from `initial`.
- Never in RTL: `force`/`release`, `deassign`, procedural continuous
  `assign`, `fork`/`join`, and non-synthesizable types (`real`, `time`,
  `string`, classes, queues, dynamic and associative arrays).

## Memory inference

- Write RAMs to the template shapes below so synthesis maps them to block
  RAM (FPGA) or compiled SRAM (ASIC). No resets on the array or the read
  register. Use `(* ram_style = "block" *)` or the vendor equivalent only
  when the tool needs a hint.

## Templates

Canonical shapes, lint-clean under `verilator --lint-only -Wall`. Neutral
names — apply `sv_preferences.md` naming on top.

### Register, async assert / sync deassert reset

```systemverilog
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) q <= '0;
  else        q <= d;
end
```

### Register with enable

```systemverilog
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n)  q <= '0;
  else if (en) q <= d;
end
```

### Two-flop synchronizer, single bit

```systemverilog
(* ASYNC_REG = "TRUE" *) logic [1:0] sync_ff;

always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) sync_ff <= '0;
  else        sync_ff <= {sync_ff[0], async_in};
end

assign sync_out = sync_ff[1];
```

### Reset synchronizer (derives rst_n from arst_n)

```systemverilog
logic [1:0] rst_ff;

always_ff @(posedge clk or negedge arst_n) begin
  if (!arst_n) rst_ff <= '0;
  else         rst_ff <= {rst_ff[0], 1'b1};
end

assign rst_n = rst_ff[1];
```

### Two-process FSM

```systemverilog
typedef enum logic [1:0] {
  ST_IDLE = 2'd0,
  ST_RUN  = 2'd1,
  ST_DONE = 2'd2
} state_t;

state_t state, state_nxt;

always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) state <= ST_IDLE;
  else        state <= state_nxt;
end

always_comb begin
  state_nxt = state;  // default: hold
  busy      = 1'b0;   // default-assign every output
  unique case (state)
    ST_IDLE: if (start)   state_nxt = ST_RUN;
    ST_RUN: begin
      busy = 1'b1;
      if (done_in)        state_nxt = ST_DONE;
    end
    ST_DONE:              state_nxt = ST_IDLE;
  endcase
end
```

No `default:` branch, deliberately: the enum is fully covered, so a
corrupted state value trips the `unique` no-match warning in simulation.

### Single-port RAM

```systemverilog
logic [DW-1:0] mem [DEPTH];

always_ff @(posedge clk) begin
  if (we) mem[addr] <= wdata;
  rdata <= mem[addr];  // read-old-data; registered output, no reset
end
```

### Simple dual-port RAM (one write port, one read port)

```systemverilog
logic [DW-1:0] mem [DEPTH];

always_ff @(posedge clk) begin
  if (we) mem[waddr] <= wdata;
end

always_ff @(posedge clk) begin
  rdata <= mem[raddr];
end
```
