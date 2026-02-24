# DESIGN.md — “AegisCC”: A C Compiler That Makes Memory-Safety Bugs Impossible

## 0. Thesis

“Memory-safety bugs impossible” means: for any program accepted by the compiler, *no execution* can exhibit:
- out-of-bounds read/write
- use-after-free
- double-free
- invalid free
- uninitialized read (configurable: strict vs compatible)
- pointer provenance violations (no “invented pointers”)
- data races (if concurrency features are used)

This is achieved by **removing undefined behavior as a semantics crutch**, **tracking lifetimes/aliasing**, and **making bounds/provenance explicit** all the way down to codegen.

This is not “C but with warnings.” It’s “C the syntax” with **a new, enforceable memory model**.

---

## 1. Goals

### 1.1 Must-have
- Compile a large subset of C with familiar syntax and tooling shape.
- Prove (statically, with minimal runtime support) spatial + temporal memory safety for all accepted programs.
- Provide predictable performance: safety doesn’t mean “slow by default.”
- Interoperate with existing C via a hardened FFI boundary.
- Produce debuggable, actionable diagnostics (ownership/lifetime errors aren’t allowed to be mystical).

### 1.2 Nice-to-have
- Incremental adoption modes for legacy code (quarantine unsafe islands).
- “Unsafe” escape hatch that is explicit, auditable, and containable (capabilities + sandboxes).
- Whole-program optimization without breaking safety invariants.

---

## 2. Non-goals (explicitly rejected)
- Accepting *all* existing C code. Some C idioms are fundamentally memory-unsafe (e.g., type-punning arbitrary pointers, pointer arithmetic across object boundaries, relying on UB).
- Reproducing every edge-case of “C as implemented.” We define *Aegis C* semantics.
- Making exploitation impossible. We make *memory-safety bugs* impossible; logic bugs still exist.

---

## 3. Threat Model

### 3.1 In-scope failures prevented
- Heap, stack, global OOB
- UAF/dangling references
- double/invalid free
- classic buffer overflows (including struct tail overflows)
- iterator invalidation patterns
- data races (with opt-in concurrency model)
- miscompiled “UB-dependent” optimizations (because UB is not a thing)

### 3.2 Out-of-scope
- Logic errors (authZ, crypto misuse)
- Side-channels
- DoS via algorithmic complexity
- Hardware faults

---

## 4. High-level Approach

AegisCC implements memory safety via a **hybrid of static guarantees + minimal runtime metadata**:

1. **Pointers are capabilities** (fat pointers):
   - `{ base, length, address, provenance, lifetime_id, mutability_cap }`
2. **No pointer arithmetic escapes bounds**:
   - `p + i` is defined only within `[base, base+length)`.
3. **Lifetimes are tracked**:
   - A pointer is only valid while its *allocation lifetime* is alive.
4. **Aliasing is constrained**:
   - Mutability requires uniqueness (borrow/ownership rules).
5. **Memory layout is explicit**:
   - `sizeof`, `alignof`, and struct layout remain C-like, but pointer rules change.
6. **Unsafe operations are isolated**:
   - Anything that could violate invariants must occur inside an `unsafe {}` region with explicit capability tokens, and is checked/contained.

This design deliberately resembles the “best parts” of Rust/CHERI/SoftBound-like systems, but keeps C ergonomics where possible.

---

## 5. Language: “Aegis C” Surface Semantics

### 5.1 Types
- Scalars, structs, unions (restricted), enums
- Arrays with known or dynamic length (bounds tracked)
- Pointers are split into *kinds*:
  - `T*` — **safe pointer** (capability pointer with bounds+lifetime)
  - `raw T*` — **raw pointer** (only inside `unsafe`, cannot be dereferenced without checks/caps)
  - `view<T>` — **borrowed slice/view** of contiguous memory (like `(ptr,len)` but safe)
  - `own<T>` — **owning handle** to heap allocation (frees at end of scope unless moved)

### 5.2 Allocation model
- Stack allocations have lexical lifetimes.
- Heap allocations are owned by `own<T>` or `own<[T]>`.
- Global/static allocations have `'static` lifetime.
- `malloc/free` still exist but are *typed wrappers*:
  - `own<[u8]> buf = alloc(u8, n);`
  - `free()` is implicit on `own` drop unless moved.

### 5.3 Mutability and aliasing
- By default, a `T*` is a **shared** (read-only) borrow unless declared `mut T*`.
- You can have many shared borrows OR one mutable borrow at a time (per lifetime region).
- This is enforced by a borrow checker adapted for C control-flow.

### 5.4 Concurrency (opt-in)
- Data shared across threads must be:
  - immutable (`shared`), or
  - synchronized types (`atomic<T>`, `mutex<T>`, `arc<T>`-like)
- No data races by construction.

---

## 6. Compiler Architecture

### 6.1 Pipeline
1. **Parse** C-like syntax → AST
2. **Desugar** to Aegis Core (explicit borrows/regions)
3. **Type + effect check**:
   - ownership, borrowing, lifetime inference
   - bounds/provenance constraints
4. **Lower** to AegisIR:
   - explicit capability pointers
   - no implicit UB operations
5. **Optimize** with safety-preserving passes
6. **Codegen**:
   - target either:
     - **CHERI-like** capability hardware (best case), or
     - **software capabilities** (fat pointers + checks), or
     - **hybrid** (narrow pointers with side tables when provably safe)

### 6.2 Key internal representations

#### Aegis Core (post-desugar)
- every borrow is explicit:
  - `&x` shared
  - `&mut x` unique
- every allocation produces an `alloc_id`
- pointer arithmetic is a checked op:
  - `ptr_offset(p, i)` yields a new capability with adjusted address, same bounds

#### AegisIR (SSA)
- instructions carry memory effects:
  - `load cap<T>` requires `cap` valid & within bounds
  - `store cap<T>` requires unique/mutable capability
- provenance is tracked:
  - capabilities can only be derived from:
    - allocation results
    - slice/subslice operations
    - field projections within an object
- integer-to-pointer casts are forbidden in safe code; allowed only in `unsafe` with explicit “forge” tokens (see §9).

---

## 7. Safety Invariants (the non-negotiables)

For every dereference of a safe pointer `p: T*`:
1. **Live**: `lifetime_id(p)` is alive at that program point.
2. **In-bounds**: `[addr(p), addr(p)+sizeof(T)) ⊆ [base(p), base(p)+len(p))`
3. **Aligned**: `addr(p) % alignof(T) == 0` (or lowered to safe unaligned loads explicitly)
4. **Provenance**: `p` is derived from a valid allocation/object graph (no “invented” pointers).
5. **Aliasing**:
   - if `store` occurs, `p` must be uniquely borrowed (`mutability_cap == Unique`).

These are enforced by compile-time proof obligations; runtime checks exist only where the proof cannot be discharged.

---

## 8. Bounds Strategy

### 8.1 Default: fat pointers
A safe pointer carries `(base, len, addr)`; subobject pointers preserve parent bounds unless narrowed.

### 8.2 Narrowing
- Field projection narrows to the field’s subrange.
- Array indexing narrows to a single element capability if needed.
- Slice operations produce `view<T>` with explicit length.

### 8.3 Eliding checks (performance)
AegisCC performs bounds-check elimination when it can prove:
- loop index ranges
- non-aliasing and immutability
- monotonic pointer walks within the same allocation
This is a first-class optimization goal; the safety model is designed to make proofs easy.

---

## 9. Temporal Safety (lifetime + deallocation)

### 9.1 Ownership
- `own<T>` is move-only. Copying is forbidden unless explicitly cloned.
- Dropping `own<T>` deallocates memory once.

### 9.2 Borrows
- Any `T*` derived from `own<T>` is tied to the owner’s lifetime unless “escaped” by transferring ownership.

### 9.3 Legacy “free”
- `free(raw void*)` exists only in `unsafe`, and requires presenting a matching `alloc_cap` token that proves the pointer was allocated by this allocator and not already freed.

This prevents double-free/invalid-free at the type level.

---

## 10. Uninitialized Memory Policy

Two modes:

### 10.1 Strict (default for new code)
- Reads of uninitialized memory are rejected.
- `uninit<T>` exists; you must explicitly initialize before read.
- `memcpy`/`memmove` are typed and track init state for trivially-copyable types.

### 10.2 Compatible (for porting)
- Allows certain patterns but inserts runtime init-tracking for selected allocations (opt-in).
- Goal: migrate to strict over time.

---

## 11. Unions and Type Punning

C unions are a UB minefield. AegisCC supports:
- **Tagged unions** (safe discriminated unions)
- **Byte-level views**:
  - `view<u8>` over any object is allowed for serialization
- **Restricted unions**:
  - reading a different field than last written is illegal unless the union is declared `repr(pun)` inside `unsafe`, and accesses require explicit conversion ops with defined semantics.

---

## 12. Integers, Pointers, and Provenance

### 12.1 Safe code rules
- pointer → integer allowed (produces an opaque `addr` type, not a plain `uintptr_t`)
- integer → pointer forbidden

### 12.2 Unsafe “forging”
Inside `unsafe` you may convert `addr` back to a pointer only with:
- a *capability to an allocation* (`alloc_cap`) and
- an offset proven within bounds

This keeps “systems code” possible while preventing the classic exploit primitive of pointer fabrication.

---

## 13. The `unsafe {}` Escape Hatch (contained, auditable)

### 13.1 Principle
Unsafe code is allowed to do dangerous things, but:
- it must be explicit (`unsafe {}` blocks or `unsafe fn`)
- it must declare which invariants it assumes/establishes
- it cannot silently contaminate safe code

### 13.2 Capabilities
Unsafe operations require tokens:
- `alloc_cap(id)` — authority over a particular allocation
- `forge_cap` — authority to re-materialize pointers from addresses (rare, gated)
- `alias_cap` — authority to create overlapping mutable aliases (rare, gated)

These tokens can only be created by privileged modules or compiler intrinsics, enabling “unsafe islands” with hard boundaries.

---

## 14. Interop / FFI

### 14.1 Calling into legacy C
- All calls to external C are treated as `unsafe` by default.
- Safe wrappers must:
  - validate bounds (convert `view<T>` to `(ptr,len)`),
  - validate lifetimes (ensure the callee cannot retain pointers unless explicitly allowed),
  - and mark any “may write” effects.

### 14.2 Being called from legacy C
- Exported functions may accept raw pointers but must immediately:
  - wrap them into checked views with explicit lengths,
  - or reject.

### 14.3 Quarantine mode
AegisCC supports compiling a project as:
- `safe` modules (full guarantees)
- `compat` modules (some runtime tracking)
- `unsafe` modules (explicitly marked, audited)

The build fails if `unsafe` expands beyond declared boundaries.

---

## 15. Standard Library Requirements

AegisCC ships a small “safe libc” layer:
- `alloc<T>(n)` → `own<[T]>`
- `slice(view<T>, start, len)` → `view<T>`
- `copy(view<u8>, view<u8>)` with bounds checks
- `str` as `view<char>` or `view<u8>` with validated encoding helpers
- containers: `vec<T>`, `map`, etc., all with ownership + borrowing

Legacy libc remains available only via `unsafe`.

---

## 16. Diagnostics (make it usable, not a theorem prover cosplay)

When rejecting code, AegisCC reports:
- the exact lifetime/borrow conflict path (“mutable borrow here, shared borrow still alive here”)
- which pointer lost bounds/provenance and why
- suggested refactors:
  - convert to `view<T>`
  - narrow scope
  - introduce `own<T>` move
  - rewrite pointer-walk loops to indexed slices

A “why is this safe?” mode can emit a proof sketch for critical hot loops (useful for review/perf tuning).

---

## 17. Testing & Verification Strategy

### 17.1 Soundness
- Property-based testing of the borrow/bounds checker.
- Differential testing vs an interpreter for AegisIR.
- Fuzz compilation + execution with random programs constrained to the supported subset.
- “No UB” contracts: compile with multiple backends and compare observable behavior.

### 17.2 Runtime assertions (debug builds)
- optional canaries for allocation lifetime IDs
- trap on failed bounds/provenance checks
- shadow memory for init tracking (compat mode)

---

## 18. Codegen Backends

### 18.1 Capability hardware backend (ideal)
- Map safe pointers to hardware capabilities.
- Bounds and permissions enforced by the architecture.

### 18.2 Software capability backend
- Fat pointers in registers/stack
- checks inserted at deref sites
- aggressive elimination to regain performance

### 18.3 Hybrid ABI
- internal representation remains fat
- externally, ABI can pass “thin” pointers where proven safe (e.g., known-size stack objects)
- side-table metadata used for rare escaping pointers (configurable)

---

## 19. Incremental Adoption Plan (how this wins in the real world)

1. **Phase 0: Tooling parity**
   - preprocessors, includes, compile database, LSP integration
2. **Phase 1: Safe subset**
   - new code in safe Aegis C, no legacy dependencies
3. **Phase 2: Quarantine legacy**
   - wrap C libraries behind `unsafe` FFI modules
4. **Phase 3: Shrink unsafe**
   - convert hot/critical paths to safe code, leave “gnarly glue” isolated

The compiler should make “unsafe footprint” measurable (LOC, call graph reachability, capability usage).

---

## 20. Success Metrics

- **Soundness**: zero known counterexamples where accepted code can violate memory safety.
- **Compatibility**: can build a meaningful set of real-world C projects with bounded porting effort.
- **Performance**: within 0–20% of clang for typical safe code; hot loops approach parity after check elimination.
- **Auditability**: unsafe regions are small, explicit, and capability-gated.

---

## 21. Appendix: What We Forbid (and why)

These are rejected in safe code:
- integer → pointer casts (pointer forging)
- arbitrary pointer arithmetic across objects
- `memcpy` into non-trivially-copyable types without typed wrappers
- “borrow escaping” (returning pointers to locals)
- mutable aliasing (two `mut` pointers alive to same region)
- relying on UB for optimization or behavior

Yes, that breaks some C wizardry. That wizardry is why your pager goes off at 3am.

---

## 22. Minimal Example

```c
own<[u8]> buf = alloc(u8, n);
view<u8> v = buf.view();          // borrow as slice
for (size_t i = 0; i < v.len; i++) {
    v[i] = 0;                     // bounds proven, safe
}
// buf drops here, deallocates once; no dangling pointers possible

````markdown
# DESIGN.md — “AegisCC”: A C Compiler That Makes Memory-Safety Bugs Impossible

## 0. Thesis

“Memory-safety bugs impossible” means: for any program accepted by the compiler, *no execution* can exhibit:
- out-of-bounds read/write
- use-after-free
- double-free
- invalid free
- uninitialized read (configurable: strict vs compatible)
- pointer provenance violations (no “invented pointers”)
- data races (if concurrency features are used)

This is achieved by **removing undefined behavior as a semantics crutch**, **tracking lifetimes/aliasing**, and **making bounds/provenance explicit** all the way down to codegen.

This is not “C but with warnings.” It’s “C the syntax” with **a new, enforceable memory model**.

---

## 1. Goals

### 1.1 Must-have
- Compile a large subset of C with familiar syntax and tooling shape.
- Prove (statically, with minimal runtime support) spatial + temporal memory safety for all accepted programs.
- Provide predictable performance: safety doesn’t mean “slow by default.”
- Interoperate with existing C via a hardened FFI boundary.
- Produce debuggable, actionable diagnostics (ownership/lifetime errors aren’t allowed to be mystical).

### 1.2 Nice-to-have
- Incremental adoption modes for legacy code (quarantine unsafe islands).
- “Unsafe” escape hatch that is explicit, auditable, and containable (capabilities + sandboxes).
- Whole-program optimization without breaking safety invariants.

---

## 2. Non-goals (explicitly rejected)
- Accepting *all* existing C code. Some C idioms are fundamentally memory-unsafe (e.g., type-punning arbitrary pointers, pointer arithmetic across object boundaries, relying on UB).
- Reproducing every edge-case of “C as implemented.” We define *Aegis C* semantics.
- Making exploitation impossible. We make *memory-safety bugs* impossible; logic bugs still exist.

---

## 3. Threat Model

### 3.1 In-scope failures prevented
- Heap, stack, global OOB
- UAF/dangling references
- double/invalid free
- classic buffer overflows (including struct tail overflows)
- iterator invalidation patterns
- data races (with opt-in concurrency model)
- miscompiled “UB-dependent” optimizations (because UB is not a thing)

### 3.2 Out-of-scope
- Logic errors (authZ, crypto misuse)
- Side-channels
- DoS via algorithmic complexity
- Hardware faults

---

## 4. High-level Approach

AegisCC implements memory safety via a **hybrid of static guarantees + minimal runtime metadata**:

1. **Pointers are capabilities** (fat pointers):
   - `{ base, length, address, provenance, lifetime_id, mutability_cap }`
2. **No pointer arithmetic escapes bounds**:
   - `p + i` is defined only within `[base, base+length)`.
3. **Lifetimes are tracked**:
   - A pointer is only valid while its *allocation lifetime* is alive.
4. **Aliasing is constrained**:
   - Mutability requires uniqueness (borrow/ownership rules).
5. **Memory layout is explicit**:
   - `sizeof`, `alignof`, and struct layout remain C-like, but pointer rules change.
6. **Unsafe operations are isolated**:
   - Anything that could violate invariants must occur inside an `unsafe {}` region with explicit capability tokens, and is checked/contained.

This design deliberately resembles the “best parts” of Rust/CHERI/SoftBound-like systems, but keeps C ergonomics where possible.

---

## 5. Language: “Aegis C” Surface Semantics

### 5.1 Types
- Scalars, structs, unions (restricted), enums
- Arrays with known or dynamic length (bounds tracked)
- Pointers are split into *kinds*:
  - `T*` — **safe pointer** (capability pointer with bounds+lifetime)
  - `raw T*` — **raw pointer** (only inside `unsafe`, cannot be dereferenced without checks/caps)
  - `view<T>` — **borrowed slice/view** of contiguous memory (like `(ptr,len)` but safe)
  - `own<T>` — **owning handle** to heap allocation (frees at end of scope unless moved)

### 5.2 Allocation model
- Stack allocations have lexical lifetimes.
- Heap allocations are owned by `own<T>` or `own<[T]>`.
- Global/static allocations have `'static` lifetime.
- `malloc/free` still exist but are *typed wrappers*:
  - `own<[u8]> buf = alloc(u8, n);`
  - `free()` is implicit on `own` drop unless moved.

### 5.3 Mutability and aliasing
- By default, a `T*` is a **shared** (read-only) borrow unless declared `mut T*`.
- You can have many shared borrows OR one mutable borrow at a time (per lifetime region).
- This is enforced by a borrow checker adapted for C control-flow.

### 5.4 Concurrency (opt-in)
- Data shared across threads must be:
  - immutable (`shared`), or
  - synchronized types (`atomic<T>`, `mutex<T>`, `arc<T>`-like)
- No data races by construction.

---

## 6. Compiler Architecture

### 6.1 Pipeline
1. **Parse** C-like syntax → AST
2. **Desugar** to Aegis Core (explicit borrows/regions)
3. **Type + effect check**:
   - ownership, borrowing, lifetime inference
   - bounds/provenance constraints
4. **Lower** to AegisIR:
   - explicit capability pointers
   - no implicit UB operations
5. **Optimize** with safety-preserving passes
6. **Codegen**:
   - target either:
     - **CHERI-like** capability hardware (best case), or
     - **software capabilities** (fat pointers + checks), or
     - **hybrid** (narrow pointers with side tables when provably safe)

### 6.2 Key internal representations

#### Aegis Core (post-desugar)
- every borrow is explicit:
  - `&x` shared
  - `&mut x` unique
- every allocation produces an `alloc_id`
- pointer arithmetic is a checked op:
  - `ptr_offset(p, i)` yields a new capability with adjusted address, same bounds

#### AegisIR (SSA)
- instructions carry memory effects:
  - `load cap<T>` requires `cap` valid & within bounds
  - `store cap<T>` requires unique/mutable capability
- provenance is tracked:
  - capabilities can only be derived from:
    - allocation results
    - slice/subslice operations
    - field projections within an object
- integer-to-pointer casts are forbidden in safe code; allowed only in `unsafe` with explicit “forge” tokens (see §9).

---

## 7. Safety Invariants (the non-negotiables)

For every dereference of a safe pointer `p: T*`:
1. **Live**: `lifetime_id(p)` is alive at that program point.
2. **In-bounds**: `[addr(p), addr(p)+sizeof(T)) ⊆ [base(p), base(p)+len(p))`
3. **Aligned**: `addr(p) % alignof(T) == 0` (or lowered to safe unaligned loads explicitly)
4. **Provenance**: `p` is derived from a valid allocation/object graph (no “invented” pointers).
5. **Aliasing**:
   - if `store` occurs, `p` must be uniquely borrowed (`mutability_cap == Unique`).

These are enforced by compile-time proof obligations; runtime checks exist only where the proof cannot be discharged.

---

## 8. Bounds Strategy

### 8.1 Default: fat pointers
A safe pointer carries `(base, len, addr)`; subobject pointers preserve parent bounds unless narrowed.

### 8.2 Narrowing
- Field projection narrows to the field’s subrange.
- Array indexing narrows to a single element capability if needed.
- Slice operations produce `view<T>` with explicit length.

### 8.3 Eliding checks (performance)
AegisCC performs bounds-check elimination when it can prove:
- loop index ranges
- non-aliasing and immutability
- monotonic pointer walks within the same allocation
This is a first-class optimization goal; the safety model is designed to make proofs easy.

---

## 9. Temporal Safety (lifetime + deallocation)

### 9.1 Ownership
- `own<T>` is move-only. Copying is forbidden unless explicitly cloned.
- Dropping `own<T>` deallocates memory once.

### 9.2 Borrows
- Any `T*` derived from `own<T>` is tied to the owner’s lifetime unless “escaped” by transferring ownership.

### 9.3 Legacy “free”
- `free(raw void*)` exists only in `unsafe`, and requires presenting a matching `alloc_cap` token that proves the pointer was allocated by this allocator and not already freed.

This prevents double-free/invalid-free at the type level.

---

## 10. Uninitialized Memory Policy

Two modes:

### 10.1 Strict (default for new code)
- Reads of uninitialized memory are rejected.
- `uninit<T>` exists; you must explicitly initialize before read.
- `memcpy`/`memmove` are typed and track init state for trivially-copyable types.

### 10.2 Compatible (for porting)
- Allows certain patterns but inserts runtime init-tracking for selected allocations (opt-in).
- Goal: migrate to strict over time.

---

## 11. Unions and Type Punning

C unions are a UB minefield. AegisCC supports:
- **Tagged unions** (safe discriminated unions)
- **Byte-level views**:
  - `view<u8>` over any object is allowed for serialization
- **Restricted unions**:
  - reading a different field than last written is illegal unless the union is declared `repr(pun)` inside `unsafe`, and accesses require explicit conversion ops with defined semantics.

---

## 12. Integers, Pointers, and Provenance

### 12.1 Safe code rules
- pointer → integer allowed (produces an opaque `addr` type, not a plain `uintptr_t`)
- integer → pointer forbidden

### 12.2 Unsafe “forging”
Inside `unsafe` you may convert `addr` back to a pointer only with:
- a *capability to an allocation* (`alloc_cap`) and
- an offset proven within bounds

This keeps “systems code” possible while preventing the classic exploit primitive of pointer fabrication.

---

## 13. The `unsafe {}` Escape Hatch (contained, auditable)

### 13.1 Principle
Unsafe code is allowed to do dangerous things, but:
- it must be explicit (`unsafe {}` blocks or `unsafe fn`)
- it must declare which invariants it assumes/establishes
- it cannot silently contaminate safe code

### 13.2 Capabilities
Unsafe operations require tokens:
- `alloc_cap(id)` — authority over a particular allocation
- `forge_cap` — authority to re-materialize pointers from addresses (rare, gated)
- `alias_cap` — authority to create overlapping mutable aliases (rare, gated)

These tokens can only be created by privileged modules or compiler intrinsics, enabling “unsafe islands” with hard boundaries.

---

## 14. Interop / FFI

### 14.1 Calling into legacy C
- All calls to external C are treated as `unsafe` by default.
- Safe wrappers must:
  - validate bounds (convert `view<T>` to `(ptr,len)`),
  - validate lifetimes (ensure the callee cannot retain pointers unless explicitly allowed),
  - and mark any “may write” effects.

### 14.2 Being called from legacy C
- Exported functions may accept raw pointers but must immediately:
  - wrap them into checked views with explicit lengths,
  - or reject.

### 14.3 Quarantine mode
AegisCC supports compiling a project as:
- `safe` modules (full guarantees)
- `compat` modules (some runtime tracking)
- `unsafe` modules (explicitly marked, audited)

The build fails if `unsafe` expands beyond declared boundaries.

---

## 15. Standard Library Requirements

AegisCC ships a small “safe libc” layer:
- `alloc<T>(n)` → `own<[T]>`
- `slice(view<T>, start, len)` → `view<T>`
- `copy(view<u8>, view<u8>)` with bounds checks
- `str` as `view<char>` or `view<u8>` with validated encoding helpers
- containers: `vec<T>`, `map`, etc., all with ownership + borrowing

Legacy libc remains available only via `unsafe`.

---

## 16. Diagnostics (make it usable, not a theorem prover cosplay)

When rejecting code, AegisCC reports:
- the exact lifetime/borrow conflict path (“mutable borrow here, shared borrow still alive here”)
- which pointer lost bounds/provenance and why
- suggested refactors:
  - convert to `view<T>`
  - narrow scope
  - introduce `own<T>` move
  - rewrite pointer-walk loops to indexed slices

A “why is this safe?” mode can emit a proof sketch for critical hot loops (useful for review/perf tuning).

---

## 17. Testing & Verification Strategy

### 17.1 Soundness
- Property-based testing of the borrow/bounds checker.
- Differential testing vs an interpreter for AegisIR.
- Fuzz compilation + execution with random programs constrained to the supported subset.
- “No UB” contracts: compile with multiple backends and compare observable behavior.

### 17.2 Runtime assertions (debug builds)
- optional canaries for allocation lifetime IDs
- trap on failed bounds/provenance checks
- shadow memory for init tracking (compat mode)

---

## 18. Codegen Backends

### 18.1 Capability hardware backend (ideal)
- Map safe pointers to hardware capabilities.
- Bounds and permissions enforced by the architecture.

### 18.2 Software capability backend
- Fat pointers in registers/stack
- checks inserted at deref sites
- aggressive elimination to regain performance

### 18.3 Hybrid ABI
- internal representation remains fat
- externally, ABI can pass “thin” pointers where proven safe (e.g., known-size stack objects)
- side-table metadata used for rare escaping pointers (configurable)

---

## 19. Incremental Adoption Plan (how this wins in the real world)

1. **Phase 0: Tooling parity**
   - preprocessors, includes, compile database, LSP integration
2. **Phase 1: Safe subset**
   - new code in safe Aegis C, no legacy dependencies
3. **Phase 2: Quarantine legacy**
   - wrap C libraries behind `unsafe` FFI modules
4. **Phase 3: Shrink unsafe**
   - convert hot/critical paths to safe code, leave “gnarly glue” isolated

The compiler should make “unsafe footprint” measurable (LOC, call graph reachability, capability usage).

---

## 20. Success Metrics

- **Soundness**: zero known counterexamples where accepted code can violate memory safety.
- **Compatibility**: can build a meaningful set of real-world C projects with bounded porting effort.
- **Performance**: within 0–20% of clang for typical safe code; hot loops approach parity after check elimination.
- **Auditability**: unsafe regions are small, explicit, and capability-gated.

---

## 21. Appendix: What We Forbid (and why)

These are rejected in safe code:
- integer → pointer casts (pointer forging)
- arbitrary pointer arithmetic across objects
- `memcpy` into non-trivially-copyable types without typed wrappers
- “borrow escaping” (returning pointers to locals)
- mutable aliasing (two `mut` pointers alive to same region)
- relying on UB for optimization or behavior

Yes, that breaks some C wizardry. That wizardry is why your pager goes off at 3am.

---

## 22. Minimal Example

```c
own<[u8]> buf = alloc(u8, n);
view<u8> v = buf.view();          // borrow as slice
for (size_t i = 0; i < v.len; i++) {
    v[i] = 0;                     // bounds proven, safe
}
// buf drops here, deallocates once; no dangling pointers possible
````

---

## 23. Summary

AegisCC makes memory-safety bugs impossible by:

* redefining pointers as capabilities with bounds + lifetimes,
* enforcing ownership/borrowing to prevent UAF and mutable aliasing,
* eliminating UB as a semantic escape hatch,
* containing the unavoidable sharp edges behind explicit, token-gated `unsafe`.

The result is “C-shaped systems programming” with the one feature C never had:
**programs you can trust not to eat your heap for sport.**
