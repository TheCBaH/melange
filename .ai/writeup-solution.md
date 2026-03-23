# Solution: `let rec` with `lazy` Forward Reference Bug

## The Bug

When compiling mutually recursive bindings where a non-function value eagerly references a `lazy` binding:

```ocaml
let rec a = { next = b }
and b = lazy { next = lazy a }
```

Melange generated JavaScript with a temporal dead zone violation:

```javascript
let a = {};
Caml_obj.update_dummy(a, { next: b });  // ReferenceError: b not yet declared
const b = { LAZY_DONE: false, VAL: function() { ... } };
```

## Root Cause

In `jscomp/core/lam_compile.ml`, the function `compile_recursive_let` selects a compilation strategy for each binding in a `let rec` group. Two cases interact badly:

**Case B** (line 315): A `Pmakeblock` whose args are all `Lfunction` or `Lconst` is compiled as a simple `const` declaration. Lazy blocks match this case because `lazy expr` is lowered to `Pmakeblock(_, _, [Lconst false; Lfunction ...])`.

**Case D** (line 389): A general `Pmakeblock` is compiled using the `update_dummy` pattern — a dummy object (`let x = {}`) is hoisted to the top, and `Caml_obj.update_dummy(x, ...)` fills it in later. The RHS is compiled eagerly with `NeedValue`, meaning all variable references in the RHS are evaluated immediately.

When `a` takes Case D and `b` takes Case B:
1. `a`'s dummy declaration `let a = {}` is hoisted to the top
2. `a`'s `update_dummy(a, { next: b })` is emitted next — this eagerly reads `b`
3. `b`'s `const b = { ... }` is emitted last — too late, `b` is in the temporal dead zone

The guard on Case B only checked the binding's own args (`args_either_function_or_const`). It did not consider whether **other bindings in the same group** would eagerly reference this binding via `update_dummy`.

## The Fix

**File**: `jscomp/core/lam_compile.ml`, Case B guard (line 315)

**Change**: Added a condition that prevents Case B from matching when any other binding in the same recursive group references this binding's identifier. When this condition is true, the binding falls through to Case D and uses the same `dummy + update_dummy` pattern, ensuring its dummy declaration is hoisted alongside the others.

### Before

```ocaml
| Lprim { primitive = Pmakeblock (_, _, _); args; _ }
  when args_either_function_or_const args ->
    (compile_lambda { cxt with continuation = Declare (Alias, id) } arg, [])
```

### After

```ocaml
| Lprim { primitive = Pmakeblock (_, _, _); args; _ }
  when args_either_function_or_const args
       && not
            (List.exists all_bindings ~f:(fun (other_id, other_arg) ->
                 (not (Ident.same other_id id))
                 && Lam_hit.hit_variable id other_arg)) ->
    (compile_lambda { cxt with continuation = Declare (Alias, id) } arg, [])
```

The new guard uses `Lam_hit.hit_variable` (already available in the codebase) to check whether any other binding in `all_bindings` references this identifier `id`. If it does, Case B is skipped and Case D handles the binding instead.

### Generated JavaScript After Fix

```javascript
let a = {};
let b = {};
Caml_obj.update_dummy(a, { next: b });
Caml_obj.update_dummy(b, { LAZY_DONE: false, VAL: function() { ... } });
const r = a;
```

Both `a` and `b` are declared as dummies first, then both are filled via `update_dummy`. No forward reference.

## Why This Approach

Several alternatives were considered:

| Approach | Pros | Cons |
|----------|------|------|
| **Strengthen Case B guard** (chosen) | Minimal change (5 lines), uses existing infrastructure, correct by construction | Slightly more `update_dummy` calls for lazy blocks that are referenced by siblings |
| Reorder output in `compile_recursive_lets_aux` | No change to case selection | Fragile — requires tracking which cases produce `const` vs `let` and sorting accordingly |
| Use `var` instead of `const`/`let` | Simple, avoids TDZ entirely | Regresses JS code quality, `var` has function-scope hoisting semantics that can mask other bugs |
| Emit Case B bindings into `declare_ids` | Keeps `const` for the binding | Would require splitting Case B into declaration + assignment, duplicating Case D logic |

The chosen approach is conservative: when in doubt about ordering, it falls back to the `update_dummy` pattern that is already proven correct for recursive bindings. The cost is that a lazy block referenced by a sibling binding gets an extra `update_dummy` call — this is negligible since `update_dummy` is a simple field copy.

## Correctness Argument

The fix is correct because:

1. **When no sibling references this binding**: The new guard is false, Case B still matches, and behavior is unchanged. No regression for the common case.

2. **When a sibling references this binding**: Case B is skipped. The binding falls to either Case C (if it matches the record-with-self-refs pattern) or Case D (general `Pmakeblock`). In Case D, the binding gets a hoisted dummy declaration, which is emitted before any `update_dummy` calls from other bindings. This eliminates the forward reference.

3. **`Lam_hit.hit_variable` traverses into function bodies**: This means even indirect references (e.g., `b` referenced inside a function inside `a`'s block) are detected. This is intentionally conservative — it may cause some bindings to unnecessarily use `update_dummy`, but it never misses a real dependency.

## Test

The fix is validated by the cram test at `test/blackbox-tests/let-rec-lazy.t`, which compiles the reproducer and verifies:
- The generated JS declares both `a` and `b` as `let` (dummy objects) before any `update_dummy` calls
- `node` executes the generated JS without errors
