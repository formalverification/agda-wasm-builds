# agda-2.9.0-da046d6-als-11

Agda 2.9.0 (the release: agda/agda's tag `v2.9.0`, `da046d6`, of 2026-11-13)
and the Agda Language Server 11 (its tag `v11`, its first release for Agda
2.9.0) built against it, compiled to WebAssembly for WASI.  The files are
the assets of the release [`agda-2.9.0-da046d6-als-11`][release].  It
replaces [`agda-2.9.0-da66a8c-als-8`](../agda-2.9.0-da66a8c-als-8/), built
from the `nightly` of 2026-10-05 and the server's v8 with a patch for Agda
2.9.0.  `SHA256SUMS` beside this file lists every one of them, and
[`NOTICE`](NOTICE) says what each binary contains and under which license.

## The files

| File | Bytes | What it is |
|---|---:|---|
| `agda.wasm` | 34,508,797 | Agda's own `agda`, a WASI command: batch checks, `--interaction`, `--interaction-json` |
| `als.wasm` | 36,179,686 | the language server, a WASI command speaking LSP on its standard input and output |
| `als-reactor.wasm` | 36,112,189 | the same server as a WASI reactor, driven from JavaScript through GHC's JavaScript FFI |
| `als-reactor.js` | 5,044 | the reactor's JavaScript glue, written by GHC's post-linker |
| `plan-agda.json`, `plan-als-command.json`, `plan-als-reactor.json` | | the three builds' cabal plans |
| `gmp-6.3.0-nodoc.tar.xz`, `gmpsrc.patch`, `0001-Enable-building-for-wasm32-wasi.patch` | | the source of the GMP the binaries link, with GHC's two patches to it |

## Using them

Check the files against `SHA256SUMS`, where you downloaded them:

    sha256sum --check --strict --ignore-missing SHA256SUMS

In a Nix flake, fetch a file by URL and by its hash in `SHA256SUMS`:

    pkgs.fetchurl {
      url = "https://github.com/formalverification/agda-wasm-builds/releases/download/agda-2.9.0-da046d6-als-11/agda.wasm";
      sha256 = "64ce7ae30de134ff4753cdb7c43a8de1498de666101241b098d9d78789d01155";
    }

The plain `agda` runs under wasmtime when the guest sees the host's paths
as they are, which also stops Agda's search for a project root at `/`.
Run `--setup` once, with `Agda_datadir` an existing directory (the build
does not use `use-xdg-data-home`, so that directory is the data directory
itself), then check a file:

    run () { wasmtime run --dir /::/ --env HOME="$HOME" --env PWD="$(pwd)" \
               --env Agda_datadir="$Agda_datadir" agda.wasm "$@"; }
    run --setup
    run Main.agda

The language server needs `--raw` for Agda's own JSON responses (without
it, it answers in its own forms).  Its command module, `als.wasm`, needs a
host whose standard input it can poll without blocking, as a browser
worker can give it through a `SharedArrayBuffer`.  The reactor needs no
shared memory: instantiate it with its glue as the `ghc_wasm_jsffi`
import (pass the glue's default export an empty object, and assign the
instance's exports to that object once instantiated), call
`_initialize` through your WASI host, which reads the server's argv
there, program name first, then `run_setup`, `new_language_server`,
and `run_language_server` without awaiting it.  Keep one `recv_message`
pending at all times; each `send_message` and `recv_message` carries one
JSON message, with no `Content-Length` header.

**Interfaces do not cross word sizes in Agda 2.9.0**: its serializer
writes `Int` and `Word` at the machine's width (agda/agda PR 8065), so
this 32-bit Agda misreads an interface a 64-bit Agda wrote, silently
checking the module again or, reading a misdecoded length, hanging; and a
64-bit Agda fails on one this Agda wrote.  Build the interfaces this Agda
reads with this Agda.

## How it was built

Every input is pinned, as follows:

+  **Agda**: agda/agda `da046d6903b7f587f5add669650e52f378f57834` (tag
   `v2.9.0`), with Agda's default flags (`optimise-heavily` on,
   `use-xdg-data-home` off), built by
   `wasm32-wasi-cabal build --index-state=2026-11-13T06:53:39Z -O2 --enable-split-sections exe:agda`
   in its own checkout, whose project names amesgen/splitmix
   `cea9e31bdd849eb0c17611bb99e33d590e126164` for wasm32.
+  **The server**: agda/agda-language-server
   `2aab2ac66bee78fcc7b6d4f5d7cef74dfbbe51f4` (tag `v11`), its submodules
   `network` at `1dc870889eee4ac733335ced4e274b4dfe8ed369` and `lsp` at
   `965490d9fe64b68370f8fb1c4127ac9ce6f20afd` (the commits the tag records),
   its `agda` submodule replaced by the Agda commit above, and the two
   patches in [`patches/`](patches/), applied with
   `git apply --whitespace=nowarn`.  The first is the import without which
   the reactor does not compile (v11 has the `Agda-2-9-0` flag and the port
   to Agda 2.9.0's API itself, which the previous release patched into v8);
   the second is the server's wasm project file: the index state, lsp 2.8
   from the server's own submodule (the reactor needs it), and lsp's
   WebSocket support off.  Then `hpack`, `autoreconf -i` in the `network`
   submodule, `cp cabal.project.wasm32 cabal.project`, and
   `wasm32-wasi-cabal build -O2 --enable-split-sections --flag=Agda-2-9-0 exe:als`,
   with `--flag=reactor` added for the reactor.  The project file also pins
   haskell-wasm/foundation `8e6dd48527fb429c1922083a5030ef88e3d58dd3`
   (basement) and k0001/network-simple
   `2c3ab6e7aa2a86be692c55bf6081161d83d50c34`.
+  **Hackage**: index state `2026-11-13T06:53:39Z`, the Agda commit's time,
   for every build (`wasm32-wasi-cabal update
   hackage.haskell.org,2026-11-13T06:53:39Z`).
+  **The toolchain**: ghc-wasm-meta
   `58260711fd5c8ffbb81f4ac56e40323ff509d43b` (the commit Agda's own
   `flake.lock` pins at the Agda commit, as at the previous release's), its
   `wasm32-wasi-ghc-9_10`, `wasm32-wasi-cabal-9_10`, `binaryen` and
   `wasmtime` packages with its own nixpkgs
   (`4014d8bb312b04b49ec74adff5f9b32982bd0580`): wasm32-wasi-ghc
   9.10.3.20260731, wasm32-wasi-cabal 3.14.2.0, binaryen version_131,
   wasmtime 47.0.2.  The native tools come from nixpkgs
   `b6018f87da91d19d0ab4cf979885689b469cdd41`: alex 3.5.4.0 and happy 2.1.7
   (they generate Agda's lexer and parser; cabal also builds wasm32 ones,
   which cannot run), hpack 0.38.3, autoconf 2.72, automake 1.18.1, and node
   22.22.2.
+  **The last steps**: `wasm-opt -Oz` on each module, and, for the
   reactor, `node $(wasm32-wasi-ghc --print-libdir)/post-link.mjs -i
   <the linked module> -o als-reactor.js` before it.  GHC's post-linker
   writes nothing under a node older than 22.18, which lacks
   `import.meta.main`.
+  **One fixed work directory**: the binaries record the paths they were
   built at (a package's data directory, and in the server the absolute
   path of Agda's generated parser, in its source locations), so a build
   gives these bytes only in the same directory.  The plans show it.

The build was run twice on 2026-11-25, the second time from an empty
work directory at the same path, and gave the same files and the same
plans, byte for byte (35 and 32 minutes on a 20-core workstation, beside
other builds).

## What was checked

All on 2026-11-25, on a 20-core workstation, by
formalverification/agda-native-ide's issue 108 (its findings comment has
each run):

+  **The plain `agda` under wasmtime**: it reports "Agda version 2.9.0",
   and under nixpkgs' wasmtime in the Nix build of agda-native-ide it
   built the 41 builtin interfaces (format version 202610261), checked
   all 343 modules of agda-algebras (at its commit `66c324f8`) and the 253
   modules of the standard library 2.4.1 they import, and then checked
   each of those 343 modules cold over the interfaces it wrote, with
   exactly one `Checking` line each.
+  **The reactor in a browser**: `als-reactor.wasm` and `als-reactor.js`, in
   agda-native-ide's page in headless Chromium 149, with
   `crossOriginIsolated` false: its browser check passed (a check, a give, a
   type error at its position, a refused postulate, an import of
   agda-algebras, a hover in a library module), its conformance suite passed
   13 of 13 scenarios, 12 of them answered exactly as the native Agda at the
   same tag answers them and the 13th differing only by the batch lane a
   page does not have, and each of agda-algebras' 343 modules checked with
   only `Main` checked over the interfaces this `agda.wasm` built.
+  **Not run for this release**: Agda's own test suites under wasmtime,
   which the previous release's README reports for the nightly; and the
   command module in a browser (`als.wasm` is built and pinned beside the
   reactor, from the same plan less one package).

[release]: https://github.com/formalverification/agda-wasm-builds/releases/tag/agda-2.9.0-da046d6-als-11
