# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Melange is a compiler toolchain that compiles OCaml/Reason to JavaScript. It's a fork of the ReScript compiler focused on deep OCaml ecosystem integration via Dune. Licensed LGPL-2.1-or-later.

## Build Commands

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
- **JS snapshot tests** (`jscomp/test/dist`, `jscomp/test/dist-es6`): built manually by commenting the only line in `jscomp/dune` then running `opam exec -- dune build`
- Full CI via Nix: `nix build .#melange .#melange-playground`

## Key Details

- OCaml 5.4, Dune 3.21+, Menhir for parser generation
- `.ocamlformat` config: `parse-docstrings = false`
- JS reserved keywords map in `jscomp/ext/js_reserved_map.ml` (auto-updated via GitHub Action)
- Flow parser vendored in `jscomp/js_parser/` — upgrade process documented in CONTRIBUTING.md
- Submodule changes to `vendor/melange-compiler-libs` require updating the `flake.nix` input URL and running `nix flake update`
