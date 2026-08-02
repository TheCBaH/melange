# Runtime module fields and OCaml namespaces

## The bug

```ocaml
exception Foo of string
module Foo = struct end
```

`ocamlc` accepts this — the exception constructor and the module live in
different namespaces — but `melc` refused it:

```
Error: Foo are exported as twice
```

## Root cause

A module compiles to a JavaScript object, which has a single namespace. The
Lambda IR started out able to tell the two `Foo`s apart (they are distinct
identifiers, `Foo/271` and `Foo/272`), but everything downstream reduced them to
`Ident.name`, i.e. to the string `"Foo"`:

- `Tcoerce_structure` carried its runtime fields as `Ident.t list`;
- `Translmod` built `Blk_module` / `Fld_module` metadata with `Ident.name`;
- `Lambda.transl_address` named a field access after the last path component,
  regardless of which namespace the path was resolved in;
- Melange keyed exports and `.cmj` entries by `Ident.name`.

The guard in `Lam_coercion.handle_exports` that produced the error was therefore
right to exist: without it, CommonJS output would have emitted a duplicate `Foo`
property and ES modules a duplicate named export. The fix had to be in the
naming, not in the guard.

## The naming policy

`Runtime_fields` (new, `vendor/melange-compiler-libs/typing/runtime_fields.ml`)
pairs an identifier with the `Shape.Sig_component_kind.t` it was bound in, and
maps that pair to the name the field gets at runtime:

| namespace | runtime name |
|---|---|
| value | `foo` |
| module | `Foo` |
| extension constructor | `Foo$extension` |
| class | `foo$class` |

Only two pairs of namespaces can ever collide, because of OCaml's
capitalization rules: modules with extension constructors (both uppercase), and
values with classes (both lowercase). Suffixing the extension constructor and
the class is therefore enough, and `$` cannot occur in an OCaml identifier, so
the encoding is injective (`Runtime_fields.unmangle` decodes it exactly).

### Why the mangling is unconditional

The obvious refinement — only rename when a sibling field actually claims the
name — is wrong, and this is the crux of the design. Coercions read fields out
of a module while only knowing the signature they coerce *to*:

```ocaml
module M : sig exception Foo of string val v : int end = struct
  exception Foo of string
  module Foo = struct let x = 1 end
  let v = Foo.x
end
```

The structure's object has to distinguish its two `Foo`s; the ascribed signature
has only one. A collision-sensitive scheme would have the coercion read `.Foo`
from an object whose exception lives at `.Foo$extension` — silently picking up
the module. Making the name a function of the component alone removes the
disagreement by construction. The same argument applies to `include`, `open`,
functor argument coercions and cross-unit accesses, which all name fields from
one side of the boundary only.

### Compatibility aliases

Mangling changes the JavaScript-facing name of every exported exception and
class, so mangled fields are *also* exposed under their plain OCaml name when
nothing else claims it:

```js
module.exports = {
  Foo$extension: Foo,
  Foo,          // compatibility alias
  v,
}
```

OCaml-generated accesses always use the canonical (mangled) name; the alias
exists purely for handwritten JavaScript. It is emitted:

- for the unit's exports, in both CommonJS and ESM printers
  (`Js_dump_import_export`), where it costs nothing;
- for nested module objects (`Js_dump`), but only when the field's value is a
  variable. Fields of a coerced module can be arbitrary expressions and the
  object literal has nowhere to bind them, so duplicating e.g. a functor
  coercion wrapper is not worth a compatibility alias.

## Commit layout

The work is split so that the mechanical part can be reviewed and bisected
without wading through regenerated JavaScript. Each half spans both repositories
and they have to move together, because the parent repository records the
submodule revision:

1. **Plumbing** (compiler-libs `lambda: thread component namespaces …` +
   melange `melange: carry runtime field descriptors …`). The namespace is
   threaded everywhere a runtime field is named or read, but
   `Runtime_fields.mangle` still returns the OCaml name for every namespace, so
   **generated code is byte-identical**: building this tree leaves
   `jscomp/test/dist` and `node_modules` untouched and the whole suite green
   against the pre-existing snapshots.
2. **Naming** (compiler-libs `lambda: name extension constructors and classes
   apart …` + melange `melange: name exported extension constructors and
   classes apart`). `mangle` starts suffixing, the compatibility aliases are
   emitted, and every output change in the diff lands here.

The regenerated snapshots and the new cram test follow in their own commits.

## What changed

`vendor/melange-compiler-libs` (branch `melange-namespaced-runtime-fields`):

- `typing/runtime_fields.{ml,mli}` — new: the descriptor and the policy.
- `typing/typedtree.{ml,mli}` — `Tcoerce_structure`'s runtime fields are
  `Runtime_fields.t list`.
- `typing/includemod.ml` — builds them from the target signature.
- `lambda/translmod.ml` — the `fields` accumulator threaded through
  `transl_struct_item` carries kinds (values, exceptions, type extensions,
  modules, recursive modules, classes, and the signatures of `include`/`open`);
  `Blk_module` names and the `Fld_module` reads for `include`/`open` use
  `Runtime_fields.name`; `get_export_identifiers` returns descriptors.
- `lambda/lambda.ml` — `transl_address` takes the namespace of the component
  being accessed; `transl_{module,value,extension,class}_path` pass theirs, and
  the recursive call for the enclosing path components passes `Module`.
- `lib/melange_compiler_libs.ml` — exposes `Runtime_fields` to Melange.

Melange:

- `Lam_stats.exports`, `Lam_coercion.export_list` and `J.program.exports` hold
  descriptors; the duplicate-export guard compares runtime names.
- `Lam_stats_export` keys `.cmj` entries by runtime name, which is what
  `Lam_compile_env` resolves `Fld_module` accesses against — so the `.cmj`
  schema did not have to change: the canonical name already encodes the pair.
- `Js_dump_import_export` and `Js_dump` print the names and the aliases.

## Deviations from `names.md`

- **`.cmj` is not re-keyed by `(namespace, name)`.** The canonical name is an
  injective encoding of that pair, so string keys still work and the serialized
  schema is untouched.
- **`Blk_module` / `Fld_module` metadata still carries plain strings.** They now
  carry the *runtime* name, computed where the namespace is known. Melange never
  needs the namespace back, except to recognize a mangled name for the alias,
  which `Runtime_fields.unmangle` does exactly.
- Nested-module compatibility aliases are limited to variable-valued fields, as
  described above.

## Cost

Every exported exception and class changes name in generated JS, which is why
the diff regenerates ~100 files under `jscomp/test/dist` and the checked-in
`node_modules/*` output. `Stdlib.Not_found` reads as
`Stdlib.Not_found$extension` in generated code now. That is the price of a
context-free scheme; the alternative (collision-sensitive names) would need the
coercion to carry the *source* module's field names through
`compose_coercions`, which is a bigger and more fragile change.

## Gotchas found on the way

- `jscomp/core/j.ml` is parsed **as an interface** by
  `jscomp/core/gen/gen_traversal.ml`, which picks the first type group carrying
  attributes to generate `js_record_{iter,map,fold}`. A doc comment (`(** … *)`)
  attaches an `ocaml.doc` attribute and silently makes *that* group the chosen
  one — the generator then dies with `Not_found`. Use `(* … *)` in that file.
- `Includemod.is_runtime_component` is what decides field *positions*;
  `Runtime_fields.of_signature_item` is kept in sync with it, and
  `Types.bound_value_identifiers` is the same list without the kinds.

## Not done

The compiler-libs change lives in the submodule on branch
`melange-namespaced-runtime-fields`. It cannot be landed the usual way from
here: `flake.nix` pins `melange-compiler-libs` to a GitHub revision, so
`flake.lock` can only be updated once the branch is pushed to
`melange-re/melange-compiler-libs`. Until then, sandboxed nix builds need

```sh
nix build -L '.?submodules=1#melange' \
  --override-input melange-compiler-libs path:./vendor/melange-compiler-libs
```

while `nix develop` + `dune build` already use the working tree's submodule.
