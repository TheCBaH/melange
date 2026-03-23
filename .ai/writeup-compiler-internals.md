# Melange Compiler Internals: `let rec` Compilation and Debugging

## Overview

This document describes the internal compiler pipeline that handles OCaml `let rec` bindings and emits JavaScript, based on an investigation into a forward-reference bug. It also covers practical debugging techniques for working inside the Melange compiler.

## Compiler Pipeline for `let rec`

When the Melange compiler encounters a `let rec ... and ...` group, the processing flows through several stages:

### 1. Lambda Conversion (`jscomp/core/lam_convert.cppo.ml`)

OCaml's typed AST is lowered to a Lambda intermediate representation. `let rec` becomes `Lam.Lletrec` containing a list of `(Ident.t * Lam.t)` bindings.

Key transformation for `lazy`: a `lazy expr` is converted into `Pmakeblock` with a `Blk_record` tag containing fields `["LAZY_DONE"; "VAL"]`. The value is a block with `Lconst false` (not yet forced) and an `Lfunction` wrapping the body (the thunk). This means by the time the backend sees a lazy binding, it looks like any other `Pmakeblock` — the "laziness" is encoded structurally, not as a distinct Lambda form.

### 2. Strongly Connected Components (`jscomp/core/lam_scc.ml`)

Before code generation, the compiler partitions `let rec` bindings into strongly connected components (SCCs) using `scc_bindings`. This determines which bindings truly depend on each other and must be compiled as a group.

**Dependency analysis** (`hit_mask`, line 32): Walks the entire Lambda tree of each binding, recording which other bindings in the group are referenced. Importantly, this traversal enters function bodies (`Lfunction { body; _ } -> hit mask body`), so a reference inside a lazy thunk IS detected as a dependency.

**SCC computation** (`Scc.graph`): Standard graph SCC algorithm on the dependency graph. If all bindings form a single SCC, they are processed as one group.

**Sorting within a group** (`sort_single_binding_group`, line 95): Within an SCC, function bindings (`Lfunction`) are sorted before non-function bindings. This is a stable sort — non-function bindings preserve their original order relative to each other.

### 3. Recursive Let Compilation (`jscomp/core/lam_compile.ml`)

The function `compile_recursive_let` (line ~238) pattern-matches on each binding's Lambda form and selects a compilation strategy. There are five cases:

#### Case A — `Lfunction` (line 255)

Direct function declaration. Emitted as `const f = function(params) { ... }`. Returns no `declare_ids` (the declaration is inline).

#### Case B — `Pmakeblock` with all-function-or-const args (line 315)

When a block's arguments are all `Lfunction` or `Lconst`, the binding is emitted as a direct `const` declaration (`Declare(Alias, id)`). This is the path taken by lazy blocks, since `lazy expr` compiles to `Pmakeblock(_, _, [Lconst false; Lfunction ...])`.

Returns no `declare_ids` — the `const` declaration and initialization happen in a single JS statement.

#### Case C — `Pmakeblock` for records with only self/external refs (line 336)

When a record's fields are all either self-references (`Ident.same pid id`), constants, or variables NOT in `all_bindings`, the compiler can emit a dummy object followed by direct field assignments:

```javascript
let cell = {};
cell.content = x;
cell.next = cell;
```

The guard explicitly rejects fields that reference other bindings in the group (the `#1716` fix), to ensure those go through the `update_dummy` path where proper hoisting happens.

Returns no `declare_ids` — the `let` declaration is inline.

#### Case D — General `Pmakeblock` (line 389)

The catch-all for any `Pmakeblock` not matched above. Compiles the RHS with `NeedValue`, then emits:

```javascript
Caml_obj.update_dummy(id, compiled_value);
```

Critically, this returns `declare_ids = [S.define_variable ~kind:Variable id (E.dummy_obj tag_info)]` — a dummy object declaration that gets **hoisted** to the top of the output by `compile_recursive_lets_aux`.

#### Case E — Other (line 412)

Pathological/fallback case. Compiled as a direct `Alias` declaration.

### 4. Assembly (`compile_recursive_lets_aux`, line 419)

This function processes a single SCC group. It uses `List.fold_right` over the bindings, accumulating:
- `output_code`: the concatenated code blocks from each binding
- `ids`: the `declare_ids` from each binding (dummy object declarations)

After the fold, if there are any `ids`, they are prepended to the output:

```
[hoisted dummy declarations] ++ [code for binding 1] ++ [code for binding 2] ++ ...
```

`fold_right` processes bindings right-to-left but accumulates left-to-right (each new binding's code is prepended), so the final code order matches the input binding order.

### 5. JS Output

The assembled statements are emitted as JavaScript. Variables declared with `~kind:Variable` become `let` declarations; those with `~kind:Alias` become `const`.

## Debugging Techniques

### Inserting Debug Printouts

The most effective way to trace the compiler's behavior is inserting `Format.eprintf` calls at key decision points. Important locations:

**`lam_scc.ml:scc_bindings`** — Print the input bindings, number of SCC clusters, and the sorted order within each cluster:

```ocaml
Format.eprintf "[DEBUG scc_bindings] input: [%s], %d cluster(s)@."
  (String.concat ~sep:", "
     (List.map ~f:(fun (id, _) -> Ident.name id)
        (Nonempty_list.to_list groups)))
  (Int_vec_vec.length clusters)
```

**`lam_compile.ml:compile_recursive_let`** — Print which case each binding takes:

```ocaml
Format.eprintf "[DEBUG] id=%s -> Case B (all fn/const args)@." (Ident.name id)
```

**`lam_compile.ml:compile_recursive_lets_aux`** — Print the fold processing order and the number of hoisted declarations:

```ocaml
Format.eprintf "[DEBUG] fold_right processing: %s, returned %d declare_ids@."
  (Ident.name ident) (List.length declare_ids)
```

### Build and Test Cycle

```sh
# Build just the compiler binary
opam exec -- dune build bin/melc.exe

# Compile a single test file directly (bypassing dune/melange build)
opam exec -- dune exec -- bin/melc.exe /tmp/test.ml -o /tmp/test.js

# Debug output goes to stderr, JS goes to the output file
# So you see both in one invocation

# Run a single cram test
opam exec -- dune build @test/blackbox-tests/runtest

# Promote corrected cram output
cp _build/.promotion-staging/test/blackbox-tests/foo.t test/blackbox-tests/foo.t
```

### Key Gotchas When Adding Debug Code

1. **`String.concat` requires `~sep:` label** in the project's stdlib extensions (not the standard OCaml `String.concat` signature).

2. **Warnings are fatal** — unused variables, missing labels, etc. will fail the build. Use `_` prefixes for intentionally unused bindings.

3. **`Format.eprintf` with `@.`** flushes the formatter, ensuring output appears immediately even if the compiler crashes later.

4. **Lambda pretty-printing**: The project has `Lam_print` but it's not always convenient. For quick debugging, pattern-matching on the Lambda constructor and printing the constructor name as a string is often sufficient.

### Reading the Lambda IR

To understand what a given OCaml expression compiles to at the Lambda level, look at:

- `lam_convert.cppo.ml` for how specific OCaml constructs are lowered
- The `Lam` module (likely in `jscomp/core/lam.ml` or similar) for the IR type definition
- `lam_analysis.ml` for analysis passes that inform code generation decisions

### Understanding the JS Output

The JS IR types are defined in `jscomp/core/j.ml`. Key types:
- `J.statement` — a JS statement (variable declaration, expression statement, etc.)
- `J.expression` — a JS expression
- `J.block` — a list of statements

`Js_output.t` wraps a block with optional value, used during compilation to thread code and values through the compilation pipeline.

`Js_dump.ml` is the final pretty-printer that converts `J.*` types to actual JavaScript text.

## Dependency Graph for `let rec` Compilation

```
Lam.Lletrec bindings
        │
        ▼
  lam_scc.ml:scc_bindings
  (partition into SCCs, sort within groups)
        │
        ▼
  lam_compile.ml:compile_recursive_lets
  (iterate over SCC groups left-to-right)
        │
        ▼
  lam_compile.ml:compile_recursive_lets_aux
  (fold_right over bindings in one SCC group)
        │
        ▼
  lam_compile.ml:compile_recursive_let
  (per-binding: choose Case A/B/C/D/E strategy)
        │
        ▼
  Js_output assembly
  (hoist declare_ids, concatenate code blocks)
        │
        ▼
  js_dump.ml → JavaScript text
```
