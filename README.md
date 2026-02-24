# AegisCC

AegisCC is a C-like compiler focused on memory safety by construction: accepted programs should not allow out-of-bounds access, use-after-free, invalid/double free, mutable aliasing violations, or pointer-provenance violations.

## Project Goals

- Compile a practical C-like subset with familiar syntax.
- Enforce ownership, borrowing, lifetimes, bounds, and provenance.
- Produce clear diagnostics with actionable fixes.
- Support incremental adoption via safe/compat/unsafe boundaries.

## Build and Test

```bash
cargo build --workspace --all-targets
cargo test --workspace --all-targets
```

## CLI Usage

```bash
aegiscc <input.c> [--emit ast|core|ir] [--mode safe|compat|unsafe] [--strict-init]
```

Examples:

```bash
# Emit parsed AST
cargo run -p aegiscc -- path/to/file.c --emit ast

# Emit lowered core form
cargo run -p aegiscc -- path/to/file.c --emit core

# Emit IR (default)
cargo run -p aegiscc -- path/to/file.c --emit ir
```

## Supported v0 Subset

- Function definitions, blocks, local declarations, return statements.
- Integer literals and basic arithmetic expressions.
- Function calls.
- Assignment statements (`x = expr;`).
- Safety-related intrinsic patterns currently recognized in lowering:
  - `borrow(x)`
  - `mut_borrow(x)`
  - `release_borrow(x)`
  - `move(x)`

Known unsupported features in v0 include unions, goto, varargs, function pointers, and floating-point types.

## License

GPLv3. See `LICENSE` for full terms.

