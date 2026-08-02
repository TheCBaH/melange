# Development environment (nix)

Melange's CI builds with Nix, and the flake dev shell is the most reliable way to
get a toolchain that matches it. This file records what was verified to work in
this devcontainer.

## What's available

- `nix` 2.35 with flakes enabled (devcontainer feature), plus the
  `anmonteiro.nix-cache.workers.dev` substituter — the dev shell is already in
  the store, so entering it is fast.
- An opam switch (`5.5.0~alpha3`) also exists. It works, but **do not mix the
  two**: both drive the same `_build/`, and alternating between them forces a
  full rebuild every time. The nix shell is the one CI uses; prefer it.

## Everyday commands

```sh
nix develop -L '.?submodules=1#' --command dune build bin/melc.exe
nix develop -L '.?submodules=1#' --command dune runtest
nix develop -L '.?submodules=1#' --command dune build @test/blackbox-tests/<name>/runtest
nix develop -L '.?submodules=1#' --command dune build @melange-runtime-tests
make shell          # interactive dev shell
make nix-<cmd>      # single-word command inside the dev shell
```

The `?submodules=1` matters: `vendor/melange-compiler-libs` is a git submodule
and the flake ref would otherwise miss it.

Inside the shell, the toolchain is OCaml 5.5.0+flambda and dune 3.24.1, and
`$PWD/_build/install/default/bin` is put on `PATH` by `nix/shell.nix`.

## Where sources come from

Two different things use the submodule differently:

- **The dev shell** (`nix develop`) only provides *dependencies*. Everything
  compiled comes from the working tree, so edits under
  `vendor/melange-compiler-libs/` are picked up by the next `dune build`. This is
  the fast edit/build loop for compiler-libs changes.
- **The packages** (`nix build .#melange`, `.#melange-playground`) ignore the
  working tree's submodule: `nix/default.nix` has a `postPatch` that deletes
  `vendor/melange-compiler-libs` and copies the `melange-compiler-libs` *flake
  input* in its place. The input is pinned in `flake.lock` to a GitHub revision.

  To sandbox-build with local submodule changes:

  ```sh
  nix build -L '.?submodules=1#melange' \
    --override-input melange-compiler-libs path:./vendor/melange-compiler-libs
  ```

  The `path:` prefix matters here. Without it nix treats the override as a git
  input and fails with *"is a shallow Git repository, so 'revCount' is not
  available"*, because the devcontainer clones the submodule with `--depth 1`.
  `path:` takes the working tree as-is, so uncommitted submodule edits are
  picked up too.

  Landing a submodule change for real means pushing it to
  `melange-re/melange-compiler-libs` first, then bumping the flake input
  (`nix flake update melange-compiler-libs`) and the submodule pointer together.

## Running the compiler by hand

```sh
nix develop -L '.?submodules=1#' --command dune build bin/melc.exe
./_build/default/bin/melc.exe foo.ml            # prints JS to stdout
./_build/default/bin/melc.exe -bs-jsx 3 foo.ml
```

`melc` needs `.cmj`/`.cmi` files of dependencies on its include path; for
anything beyond a self-contained file, write a cram test instead
(`test/blackbox-tests/*.t`) — those set up dune projects with the real melange
rules.

## Tests

- `test/unit-tests/` — alcotest.
- `test/blackbox-tests/*.t` — cram. Failures print a diff; `dune promote`
  accepts it.
- `jscomp/test/dist/` — checked-in JS snapshots, regenerated (and promoted into
  the source tree) by `dune build @melange-runtime-tests`. CI additionally
  *runs* them with mocha, so they catch runtime breakage, not just output churn.

### Cram tests that run `node` pass locally but fail under nix

The repository has a checked-in `node_modules/` holding the compiled stdlib and
runtime JS. Locally, node finds it by walking up from `_build/…`, so a
`melange.emit` stanza with `(emit_stdlib false)` and no `(libraries …)` still
runs. `nix/default.nix` does not include `node_modules` in the package's source
fileset, so the same test dies with `Cannot find module 'melange/…'` in the
sandbox. Make the dependency explicit: declare `(libraries melange.js)` for
`Caml_*` runtime modules, and leave `emit_stdlib` at its default when the code
touches the stdlib (classes pull in `melange/camlinternalOO.js`).

Worth running `nix build` on any new cram test that shells out to `node`.
