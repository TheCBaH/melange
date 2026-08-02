# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Melange is a compiler toolchain that compiles OCaml/Reason to JavaScript. It's a fork of the ReScript compiler focused on deep OCaml ecosystem integration via Dune. Licensed LGPL-2.1-or-later.

## Build Commands

### Nix (preferred — this is what CI uses)

The flake dev shell provides the exact toolchain CI uses (OCaml 5.5 + dune 3.24 +
node/mocha/reason). Prefix any dune command with the dev shell:

```sh
nix develop -L '.?submodules=1#' --command dune build bin/melc.exe   # Build the compiler
nix develop -L '.?submodules=1#' --command dune build               # Build everything
nix develop -L '.?submodules=1#' --command dune runtest             # Unit + cram tests
nix develop -L '.?submodules=1#' --command dune build @test/blackbox-tests/<name>/runtest
nix develop -L '.?submodules=1#' --command dune build @melange-runtime-tests  # regen jscomp/test/dist
make shell                    # same dev shell, interactive ($SHELL)
make nix-<cmd>                # runs a single-word command in the dev shell
```

The dev shell only supplies dependencies: sources (including
`vendor/melange-compiler-libs`) come from the working tree, so local submodule
edits are picked up by `dune build` immediately. `_build/` is shared with any
opam switch you may also have — switching between the two forces a full rebuild,
so pick one and stay there.

Sandboxed builds of the packages themselves do *not* use the working tree's
submodule; `nix/default.nix` replaces `vendor/melange-compiler-libs` with the
`melange-compiler-libs` flake input. To validate local submodule changes that
way, override the input:

```sh
nix build -L '.?submodules=1#melange' \
  --override-input melange-compiler-libs path:./vendor/melange-compiler-libs
```

### OPAM (alternative)

```sh
opam exec -- dune build                    # Build the whole project
opam exec -- dune runtest                  # Run all tests (unit + cram)
make test                                  # Run tests via opam exec
make dev                                   # Build with dune build @install
opam exec -- dune exec -- bin/melc.exe  # Run the dev compiler
```

### Single test execution

```sh
opam exec -- dune build @test/blackbox-tests/<test-name>/runtest   # Run a single cram test
opam exec -- dune runtest test/unit-tests                           # Run unit tests only
```

Cram test output is auto-promoted: after a failing run, `dune promote` (or
`dune build @runtest --auto-promote`) updates the `.t` files.

### Setup (OPAM)

```sh
git submodule update --init --recursive --remote   # Initialize melange-compiler-libs
make opam-init                                      # Create local switch + install deps
```

### Playground

```sh
make playground-dev        # Build playground (dev profile)
make playground-test       # Test playground (release profile)
```

## Architecture

### Compiler pipeline

The compiler lives in `jscomp/`:
- **`core/`** — Main backend (`melangelib`): converts OCaml AST → JavaScript IR → JS output
- **`common/`** — `melange_ffi` library: FFI code shared between compiler core and PPX
- **`melstd/`** — `ext` library: internal stdlib extensions (data structures, utilities)
- **`js_parser/`** — Vendored Facebook Flow parser, used to classify `%mel.raw` JS code

### PPX preprocessor (`ppx/`)

- All `%mel.*` extensions and `@mel.*` attributes declared in `ppx/melange_ppx.ml`
- Handles FFI `external` declarations and attribute processing
- **Not a pure syntax transform**: some PPX-generated attributes are interpreted later during compilation (see `jscomp/core/lam_convert.ml`)

### Runtime and standard libraries (`jscomp/`)

- **`runtime/`** — `melange.js` library + `Caml_*` low-level primitives (compiled to JS)
- **`stdlib/`** — Melange standard library (depends on runtime)
- **`others/`** — Optional libraries: `melange.belt`, `melange.dom`, `melange.node`

### Binaries (`bin/`)

- `melc.exe` — Main compiler
- `melppx.exe` — PPX executable

### Vendored dependencies

- `vendor/melange-compiler-libs/` — Git submodule of OCaml compiler libs (must be initialized)

## Testing

- **Unit tests** (`test/unit-tests/`): Alcotest-based, ~12 test modules
- **Cram tests** (`test/blackbox-tests/`): 100+ `.t` files for integration/blackbox testing
- **JS snapshot tests** (`jscomp/test/dist`): checked-in JS produced by the two
  `melange.emit` stanzas in `jscomp/test/dune` (commonjs + esm). Regenerate with
  `dune build @melange-runtime-tests` — the stanzas `promote` into the source
  tree, so changed output shows up as a working-tree diff. The nix package's
  `postCheck` builds that alias and then runs `mocha "jscomp/test/dist/**/*_test.*js"`,
  i.e. the snapshots are also *executed* in CI.
- Full CI via Nix: `nix build .#melange .#melange-playground`

## Key Details

- OCaml 5.4, Dune 3.21+, Menhir for parser generation
- `.ocamlformat` config: `parse-docstrings = false`
- JS reserved keywords generated into `js_reserved_map.ml` by `jscomp/melstd/dune`
  from `jscomp/melstd/gen/keywords.list` (list auto-updated via GitHub Action)
- Flow parser vendored in `jscomp/js_parser/` — upgrade process documented in CONTRIBUTING.md
- `jscomp/core/j.ml` (the JS IR) is parsed *as an interface* by
  `jscomp/core/gen/gen_traversal.ml` to generate `js_record_{iter,map,fold}`. It
  picks the first type group carrying attributes, so a doc comment (`(** … *)`)
  in that file silently retargets the generator — use `(* … *)` there.
- A module's runtime fields are named by `Runtime_fields` (in compiler-libs):
  OCaml namespaces are flattened into the JS object's single one, with
  extension constructors and classes suffixed `$extension` / `$class`
- Submodule changes to `vendor/melange-compiler-libs` require updating the `flake.nix` input URL and running `nix flake update`
