# SystemVerilog Preference Guidelines

## Naming

In signal or port declarations, use one signal/port per line.

| Prefix | Use |
|--------|-----|
| `G_` | Parameters/generics; names are to be capitalized |
| `f_` | Registers (signals used inside a block that are registers) |
| `d_` | Combinatorial signals (used in a combinatorial way inside a file) |
| `w_` | Other signals (interconnect, or role unclear within the file) |

For module instantiations: prefix with `i_`, naming `<module> i_<module>`. When multiple instances exist, append underscore and number, e.g. `i_<module>_0`.