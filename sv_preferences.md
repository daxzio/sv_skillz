# SystemVerilog preference guidelines

Style conventions: naming and formatting. Correctness rules live in
`sv_skillz.md`; on any conflict, correctness wins. These conventions apply
to new files. When editing an existing codebase, match the surrounding
style unless explicitly asked to restyle.

## Case rules

- lowercase snake_case: modules, signals, ports, instances, functions,
  tasks, generate labels, packages, file names.
- UPPER_SNAKE_CASE: parameters, localparams, enum members.
- Typedefs: lowercase snake_case with a `_t` suffix (`state_t`,
  `axi_req_t`), for enums, packed structs, and unions alike.

## Prefixes

| Prefix | Use |
|--------|-----|
| `G_`   | Module parameters (generics); uppercase, e.g. `G_WIDTH` |
| `C_`   | `localparam` constants; uppercase, e.g. `C_MAX` |
| `f_`   | Registered signals — driven in an `always_ff` in this module |
| `d_`   | Combinational signals — computed in this module by `always_comb` or `assign` |
| `w_`   | Interconnect — signals that only route between instance ports, not computed here |
| `v_`   | Block-local variables (the blocking-temporary case in `sv_skillz.md`) |
| `i_`   | Module instances |
| `gen_` | Generate block labels |

- Ports carry no role prefix and no direction suffix; direction is visible
  in the ANSI header. Drive ports directly from processes or `assign` —
  never create a prefixed shadow signal (`f_q` then `assign q = f_q;`)
  just to satisfy the prefix table.
- A register/next-value pair shares its base name: `f_state` in the
  `always_ff`, `d_state` in the `always_comb`. Applied to the FSM template
  in `sv_skillz.md`, `state` becomes `f_state` and `state_nxt` becomes
  `d_state`.

## Suffixes

- `_n` for active-low signals: `rst_n`, `cs_n`.
- `_t` for typedefs, as above.

## Clocks and resets

- Single clock domain: `clk`, with `rst_n` for the synchronously
  de-asserted reset and `arst_n` for the raw asynchronous input, matching
  the reset policy in `sv_skillz.md`.
- Multiple domains: `clk_<domain>` and `rst_<domain>_n`
  (`clk_axi` / `rst_axi_n`).

## Enum members

- Uppercase, sharing a short prefix derived from the type name:
  `state_t` members are `ST_IDLE`, `ST_RUN`, `ST_DONE`. Enum members land
  in the enclosing scope's namespace, so the shared prefix is what
  prevents collisions between enums.

## Instances

- Name instances `<module> i_<module>`. Multiple instances of the same
  module append an index: `i_<module>_0`, `i_<module>_1`.
- Inside a generate loop, use the bare `i_<module>` name with no index —
  the hierarchy supplies it (`gen_lane[0].i_pulse_stretch`), per
  `sv_skillz.md`.

## Packages

- Packages are named `<name>_pkg`, in a file `<name>_pkg.sv`.

## Formatting

- Spaces, not tabs; four spaces per indent level.
- One declaration per line; one signal or port per line.
- One module per file; the file name matches the module name.
- Keep lines at or under 100 characters.

## Worked example

Lint-clean under `verilator --lint-only -Wall`; demonstrates every prefix
except `v_`.

```systemverilog
module pulse_stretch #(
    parameter int G_WIDTH = 4
) (
    input  logic clk,
    input  logic rst_n,
    input  logic trigger,
    output logic pulse
);

    localparam logic [G_WIDTH-1:0] C_MAX = '1;

    logic [G_WIDTH-1:0] f_count;
    logic [G_WIDTH-1:0] d_count;
    logic               d_active;

    always_comb begin
        d_active = (f_count != '0);
        d_count  = f_count;
        if (trigger)       d_count = C_MAX;
        else if (d_active) d_count = f_count - 1'b1;
    end

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) f_count <= '0;
        else        f_count <= d_count;
    end

    assign pulse = d_active;
endmodule
```

```systemverilog
module pulse_stretch_bank #(
    parameter int G_LANES = 2,
    parameter int G_WIDTH = 4
) (
    input  logic               clk,
    input  logic               rst_n,
    input  logic [G_LANES-1:0] trigger,
    output logic [G_LANES-1:0] pulse
);

    logic [G_LANES-1:0] w_pulse;

    for (genvar g = 0; g < G_LANES; g++) begin : gen_lane
        pulse_stretch #(
            .G_WIDTH (G_WIDTH)
        ) i_pulse_stretch (
            .clk     (clk),
            .rst_n   (rst_n),
            .trigger (trigger[g]),
            .pulse   (w_pulse[g])
        );
    end

    assign pulse = w_pulse;
endmodule
```
