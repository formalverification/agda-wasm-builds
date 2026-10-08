# agda-wasm-builds

Agda and the Agda Language Server compiled to WebAssembly (WASI), built by
the formalverification organization from pinned sources: one release per
build, every file pinned by its SHA-256.  They serve the organization's
browser workbench for Agda, and anyone else who wants an Agda that runs in
a browser or under a WASI runtime such as wasmtime.

## Releases

| Release | Agda | Language server | What was built, and how |
|---|---|---|---|
| [`agda-2.9.0-da66a8c-als-8`][r1] | 2.9.0, unreleased (agda/agda `da66a8c`, the `nightly` of 2026-10-05) | v8, with two patches for Agda 2.9.0 | [`agda-2.9.0-da66a8c-als-8/`](agda-2.9.0-da66a8c-als-8/) |

A release is named `agda-<Agda version>-<Agda commit>-als-<server
version>` and does not change once published: a new build is a new
release.  Its directory in this repository says what was built from which
sources and how, carries the patches applied, the license texts of
everything the binaries contain, and the release's `SHA256SUMS`.

## Pinning a file

Pin each file by its URL and by its hash in `SHA256SUMS`, for example in a
Nix flake:

    pkgs.fetchurl {
      url = "https://github.com/formalverification/agda-wasm-builds/releases/download/agda-2.9.0-da66a8c-als-8/agda.wasm";
      sha256 = "da2f1efc6b13e0d8b8a69311511c8ed6fbfbc2759bd8ec299004580fcfba1eb1";
    }

If a fetch stops with `hash mismatch in fixed-output derivation`, the file
is not the one published under that name: check the URL before you accept
another hash.  To find the hash of a file you are pinning for the first
time, put `lib.fakeHash` in the hash's place, run `nix build`, and copy the
hash the error prints after `got:`.

## Licenses

The binaries are built from free software only, and each release's
`NOTICE` says what each one contains and under which license, with the
license texts beside it.  They link GMP, which is distributed under the
GNU Lesser General Public License, version 3 or later; GMP's source, with
GHC's two patches to it, is attached to every release.
