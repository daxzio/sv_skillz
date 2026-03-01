# sv_skillz

SystemVerilog coding guidelines for AI-assisted RTL development.

## What is this?

This repo contains a curated set of SystemVerilog design rules, coding standards, and naming conventions. The guidelines are written so they can be fed to **AI coding agents** — when an agent generates, reviews, or refactors SystemVerilog code, it follows these standards automatically.

## Why?

AI coding assistants are powerful but don't inherently know your team's conventions or the subtle hardware-design pitfalls that cause sim/synth mismatches, metastability, latches, and other bugs that are expensive to find late. By encoding these rules in a format the AI can reference, you get:

- **Consistent RTL** that follows the same style and conventions across the team.
- **Fewer common bugs** — blocking/non-blocking misuse, missing defaults, CDC violations, X-propagation issues, and other classic mistakes are caught at authoring time rather than in verification.
- **Onboarding help** — the guidelines double as a reference for engineers new to SystemVerilog or to the project's coding style.

## Files

| File | Purpose |
|------|---------|
| `sv_skillz.md` | **Correctness rules** — assignments, process types, CDC, FSM style, synthesis-safe coding, assertions, and common anti-patterns. These rules affect the functional quality and reliability of the design. |
| `sv_preferences.md` | **Style preferences** — naming conventions, signal prefixes, formatting. These are subjective choices that improve readability and consistency but have no impact on code correctness. Kept separate so teams or individuals can swap in their own style without touching the correctness rules. |

## Usage

These files can be loaded as rules, skills, or system prompts in any AI coding agent so it applies them when working on SystemVerilog projects. Consult your agent's documentation for how to attach reference files or custom instructions.

## License

This work is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE).
