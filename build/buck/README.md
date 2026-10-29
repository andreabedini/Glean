# Building with Buck2

Work in progress: building Glean with [Buck2](https://buck2.build), with
the Haskell rules of [timbuktu](https://github.com/andreabedini/timbuktu)
and hsthrift's Buck2 build. So far this covers the packages of
`cabal.project` but glean-clang (see "Not yet").

buck2 itself, buildifier and yq are in `mise.toml`, at the versions
timbuktu uses:

    mise install

## hsthrift

hsthrift is where `install_deps.sh` and `cabal.project` put it, in
`hsthrift/`, at a revision with its Buck2 build (the `buck2` branch of
https://github.com/andreabedini/hsthrift for now). There it is the cell
`hsthrift`: its C++ dependencies (`hsthrift//build/buck/deps`, from
getdeps manifests, rocksdb included), its toolchains (`toolchains`, which
is `hsthrift/build/buck/toolchains`) and its Thrift rules come from there.
The cell `timbuktu` is the one hsthrift names, at the commit
`.buckconfig` pins.

See hsthrift's `build/buck/README.md` for the system packages the C++
dependencies need. Glean also needs icu and gtest (`libicu-devel` and
`gtest-devel` on Fedora; see `system/BUCK`). The C++ builds are big:
cap the memory of everything Buck2 runs in `.buckconfig.local`,

    [buck2_resource_control]
      status = if_available
      memory_max = 17179869184

## The build plan

The `BUCK` file at the root is a build plan made by timbuktu's `plan.bxl`
(its header says how to make it again; `plan.sh` here runs the same): GHC
9.6.7 and the Hackage index as of 2025-04-14, as `cabal.project` pins it,
with the flags `cabal.project` gives. The packages of this repository and
hsthrift's are local packages of one plan, as in `cabal.project`;
hsthrift's are given as `CELL//DIR=TARGET`, their `cabal_package` in
hsthrift's own plan, since a glob here cannot reach another cell.

`glean.cabal` is made from `glean.cabal.in` by the Makefile, with the C++
sources in it (Cabal builds them); make it before planning:

    make glean.cabal

and in hsthrift, `make setup-folly-buck` (see its README). `plan.bxl`
needs cabal-solver-plan, whose path goes in `.buckconfig.local`:

    [haskell]
      cabal_solver_plan = /path/to/cabal-solver-plan

## Generated code

What the Makefile generates before Cabal runs is made by the targets in
`BUCK` here, given to `plan.bxl` with `--generated DIR=TARGET`. A
component of a package gets the directories its modules, sources and
headers are in, so the generators can be executables of the glean package
itself:

- the Haskell of the Thrift files (`thrift-*`, the Makefile's
  `thrift-glean-hs`, and `hsthrift-thrift-lib` for hsthrift's thrift-lib),
  with `thrift_hs` of hsthrift's `build/buck/thrift/defs.bzl` and the
  thrift-compiler of this plan;
- `bytecode-hs`, glean/hs/Glean/RTS/Bytecode/Gen, by `gen-bytecode-hs`;
- `schema`, what `gen-schema` makes of glean/schema/source, and from it
  `schema-hs` (glean/schema/hs), `schema-cpp` (glean/schema/cpp, schema.h)
  and `schema-thrift-hs` (glean/schema/thrift/gen-hs2, the Makefile's
  `thrift-schema-hs`).

The generators are built for the target platform (`$(exe_target)`), like
the libraries, so that nothing is built twice.

## Building

The targets are named after the packages, e.g.

    buck2 build //:exe-glean //:exe-glean-server //:exe-glean-hyperlink
    buck2 run //:exe-glean -- --help

## Working on hsthrift or timbuktu

To try changes to hsthrift, `hsthrift/` can be a jj workspace (or a git
worktree) of a checkout of it. For timbuktu, put one in `.timbuktu` and use
it rather than the pinned commit, in `.buckconfig.local`:

    [cells]
      timbuktu = .timbuktu
    [external_cells]
      timbuktu = disabled

A cell has to be inside the project. Buck2 may not notice files of a
directory made after its daemon started: `buck2 kill` then.

## Not yet

- glean-clang: it is `build-type: Custom` (its Setup.hs finds LLVM 19 with
  llvm-config), which the plan's local packages do not support.
- The test suites.
