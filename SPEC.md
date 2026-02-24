# SPEC.md - AegisCC v0 Executable Spec

## 1. Scope

This spec defines the first implementable version of AegisCC ("v0"), derived from `DESIGN.md`.

v0 objective: compile and typecheck a safe C-like subset with static guarantees for:
- out-of-bounds access prevention
- temporal safety (no use-after-free, invalid free, double-free)
- no mutable aliasing violations
- no uninitialized reads (strict mode only in v0)
- pointer provenance preservation

v0 non-objectives:
- concurrency model
- full C compatibility
- CHERI backend
- full optimizer pipeline

## 2. Language Profile (v0)

### 2.1 Source unit
- One translation unit per compile invocation.
- `#include` unsupported in v0 parser (handled later by tooling phase).
- No macros/preprocessor in v0.

### 2.2 Supported declarations
- `struct`, `enum`, `typedef`
- function declarations/definitions
- local variable declarations
- global `const` variables only

### 2.3 Supported types
- Scalars: `u8 u16 u32 u64 i8 i16 i32 i64 bool usize isize`
- `void` (function return only)
- Arrays: `[T; N]` (compile-time N), `own<[T]>`, `view<T>`
- Pointers:
  - `T*` safe borrow pointer
  - `mut T*` unique mutable safe pointer
  - `raw T*` unsafe-only raw pointer
- Ownership:
  - `own<T>`
  - `own<[T]>`

### 2.4 Unsupported in v0
- unions (safe or restricted)
- varargs
- function pointers
- `goto`
- floating point
- integer-to-pointer casts in safe code

## 3. Surface Syntax (v0 subset)

### 3.1 Key constructs
- allocation: `own<[T]> x = alloc(T, n);`
- borrow shared: `T* p = &x;`
- borrow mutable: `mut T* p = &mut x;`
- view: `view<T> v = x.view();`
- unsafe block: `unsafe { ... }`

### 3.2 Expressions
- literals, identifiers
- unary: `&`, `&mut`, `*`, `-`, `!`
- binary: arithmetic/bitwise/comparison/logical on scalar types
- indexing: `v[i]`
- field access: `a.b`
- calls
- casts (restricted; no int->ptr in safe mode)

### 3.3 Statements
- `let`-style and C-style local declarations
- `if`, `while`, `for` (desugared to `while`)
- `return`
- block expression scopes

## 4. Static Semantics

### 4.1 Ownership
- `own<T>` and `own<[T]>` are move-only.
- Assignment or argument passing moves ownership unless type is `Copy`.
- Moved value cannot be used.
- Drop occurs once at end of lexical scope unless moved out.

### 4.2 Borrowing and aliasing
- Shared borrow (`T*`) is read-only.
- Mutable borrow (`mut T*`) requires uniqueness.
- Rule: many shared OR one mutable for overlapping region/lifetime.
- Borrow lifetime is inferred from use sites and scope.

### 4.3 Lifetimes
- Stack allocations: lexical lifetime.
- Heap allocations: tied to owner lifetime.
- Globals: `'static`.
- Any dereference requires referenced lifetime to be alive.

### 4.4 Initialization (strict)
- Every read must target definitely-initialized storage.
- `uninit<T>` allowed only with explicit initialization before any read.
- `memcpy`/`memmove` are not in v0 safe surface.

### 4.5 Provenance and casts
- Safe pointer values can only derive from:
  - allocation result
  - borrow of existing object
  - field/index/slice projection
  - function parameter/return already typed as safe pointer/view
- `ptr -> addr` allowed.
- `addr/int -> ptr` forbidden outside `unsafe`.

## 5. Dynamic Model

### 5.1 Capability pointer representation
All safe pointers lower conceptually to:
`{ base, len, addr, provenance_id, lifetime_id, mutability_cap }`

### 5.2 Required dereference checks (proof obligations)
For each `load/store` through safe pointer:
1. lifetime live
2. bounds include full accessed object
3. alignment satisfied (or explicit unaligned op)
4. provenance valid
5. for `store`: unique mutable capability active

Compiler must statically discharge these where possible; otherwise reject in v0 (no fallback runtime checks yet for M1).

## 6. Unsafe Model (v0)

### 6.1 Unsafe boundary
- `unsafe {}` and `unsafe fn` supported.
- Raw pointer dereference only legal in `unsafe`.

### 6.2 Capability tokens
Token names reserved and represented in IR/type checker:
- `alloc_cap(id)`
- `forge_cap`
- `alias_cap`

v0 policy:
- parser/typechecker recognize token requirements
- only compiler intrinsics may produce them
- no user-defined token minting

## 7. IR Contracts

### 7.1 Aegis Core (post-desugar)
Must make explicit:
- borrows (`BorrowShared`, `BorrowMut`)
- ownership moves/drops
- pointer offset/index ops (`PtrOffset`, `Index`)
- allocation IDs

### 7.2 AegisIR (SSA-like)
Instruction set minimum:
- `alloc`, `drop`
- `load`, `store`
- `gep`/field projection
- `bounds_narrow`
- `phi`
- `call`, `ret`

Each memory op carries effect metadata:
- `reads(region)`
- `writes(region)`
- required capability kind

## 8. Diagnostics Contract

Every rejected program must include:
- primary error with source span
- invariant violated (ownership/borrow/lifetime/bounds/provenance/init)
- at least one related span ("borrow created here", "owner dropped here")
- one actionable suggestion

Stable diagnostic IDs required:
- `E1xxx` ownership/move
- `E2xxx` borrow/alias
- `E3xxx` lifetime
- `E4xxx` bounds/provenance
- `E5xxx` init
- `E6xxx` unsafe/capability

## 9. CLI Contract (v0)

Command:
`aegiscc <input.c> [--emit ast|core|ir] [--mode safe|compat|unsafe] [--strict-init]`

v0 requirements:
- default mode: `safe`
- `compat` accepted but treated as alias of `safe` for now
- non-zero exit on any diagnostic

## 10. Test Contract

### 10.1 Test categories
- `tests/parse/*` valid/invalid syntax
- `tests/type/*` ownership/borrow/lifetime/init/provenance
- `tests/diag/*` golden diagnostics with IDs/spans
- `tests/ir/*` snapshot of Core + IR lowering

### 10.2 Minimum M1 coverage
- 30 parser tests
- 40 type/effect tests
- 20 diagnostics golden tests
- 10 lowering snapshots

## 11. M1 Exit Criteria

M1 is complete when:
1. Parser accepts the defined subset and rejects unsupported features with explicit diagnostics.
2. Desugar to Aegis Core is implemented for all supported constructs.
3. Type/effect checker enforces ownership, borrows, lifetimes, strict initialization, and safe provenance rules.
4. AegisIR generation works for well-typed programs.
5. Test contract minimums are met in CI.

