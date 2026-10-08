# agda-2.9.0-da66a8c-als-8

Agda 2.9.0 (unreleased: agda/agda `da66a8c`, the `nightly` of
2026-10-05) and the Agda Language Server 8 built against it, compiled to
WebAssembly for WASI.  The files are the assets of the release
[`agda-2.9.0-da66a8c-als-8`][release].  `SHA256SUMS` beside this file
lists every one of them, and [`NOTICE`](NOTICE) says what each binary
contains and under which license.

## The files

| File | Bytes | What it is |
|---|---:|---|
| `agda.wasm` | 34,277,969 | Agda's own `agda`, a WASI command: batch checks, `--interaction`, `--interaction-json` |
| `als.wasm` | 35,880,158 | the language server, a WASI command speaking LSP on its standard input and output |
| `als-reactor.wasm` | 35,817,413 | the same server as a WASI reactor, driven from JavaScript through GHC's JavaScript FFI |
| `als-reactor.js` | 5,044 | the reactor's JavaScript glue, written by GHC's post-linker |
| `plan-agda.json`, `plan-als-command.json`, `plan-als-reactor.json` | | the three builds' cabal plans |
| `gmp-6.3.0-nodoc.tar.xz`, `gmpsrc.patch`, `0001-Enable-building-for-wasm32-wasi.patch` | | the source of the GMP the binaries link, with GHC's two patches to it |

## Using them

Check the files against `SHA256SUMS`, where you downloaded them:

    sha256sum --check --strict --ignore-missing SHA256SUMS

In a Nix flake, fetch a file by URL and by its hash in `SHA256SUMS`:

    pkgs.fetchurl {
      url = "https://github.com/formalverification/agda-wasm-builds/releases/download/agda-2.9.0-da66a8c-als-8/agda.wasm";
      sha256 = "da2f1efc6b13e0d8b8a69311511c8ed6fbfbc2759bd8ec299004580fcfba1eb1";
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

+  **Agda**: agda/agda `da66a8c75f11d10699a6b38b261efdf244b66f2a`, with
   Agda's default flags (`optimise-heavily` on, `use-xdg-data-home` off),
   built by `wasm32-wasi-cabal build --index-state=2026-10-05T16:41:29Z -O2
   --enable-split-sections exe:agda` in its own checkout, whose project
   names amesgen/splitmix `cea9e31bdd849eb0c17611bb99e33d590e126164` for
   wasm32.
+  **The server**: agda/agda-language-server
   `0f77a209243185cb8e36a11bfc7b6ab47f5125b4` (tag `v8`), its submodules
   `network` at `1dc870889eee4ac733335ced4e274b4dfe8ed369` and `lsp` at
   `965490d9fe64b68370f8fb1c4127ac9ce6f20afd` (the commits the tag
   records), its `agda` submodule replaced by the Agda commit above, and
   the two patches in [`patches/`](patches/), applied with `git apply
   --whitespace=nowarn`.  The first adds a flag `Agda-2-9-0` and the
   version-guarded branches Agda 2.9.0's API needs, and the import without
   which v8's reactor does not compile; the second is the server's wasm
   project file: the index state, lsp 2.8 from the server's own submodule
   (the reactor needs it), and lsp's WebSocket support off.  Then `hpack`,
   `autoreconf -i` in the `network` submodule, `cp cabal.project.wasm32
   cabal.project`, and `wasm32-wasi-cabal build -O2 --enable-split-sections
   --flag=Agda-2-9-0 exe:als`, with `--flag=reactor` added for the
   reactor.  The project file also pins haskell-wasm/foundation
   `8e6dd48527fb429c1922083a5030ef88e3d58dd3` (basement) and
   k0001/network-simple `2c3ab6e7aa2a86be692c55bf6081161d83d50c34`.
+  **Hackage**: index state `2026-10-05T16:41:29Z`, the Agda commit's time,
   for every build (`wasm32-wasi-cabal update
   hackage.haskell.org,2026-10-05T16:41:29Z`).
+  **The toolchain**: ghc-wasm-meta
   `58260711fd5c8ffbb81f4ac56e40323ff509d43b` (the commit Agda's own
   `flake.lock` pins at the Agda commit), its `wasm32-wasi-ghc-9_10`,
   `wasm32-wasi-cabal-9_10`, `binaryen` and `wasmtime` packages with its
   own nixpkgs (`4014d8bb312b04b49ec74adff5f9b32982bd0580`):
   wasm32-wasi-ghc 9.10.3.20260731, wasm32-wasi-cabal 3.14.2.0, binaryen
   version_131, wasmtime 47.0.2.  The native tools come from nixpkgs
   `b6018f87da91d19d0ab4cf979885689b469cdd41`: alex 3.5.4.0 and happy
   2.1.7 (they generate Agda's lexer and parser; cabal also builds wasm32
   ones, which cannot run), hpack 0.38.3, autoconf 2.72, automake 1.18.1,
   and node 22.22.2.
+  **The last steps**: `wasm-opt -Oz` on each module, and, for the
   reactor, `node $(wasm32-wasi-ghc --print-libdir)/post-link.mjs -i
   <the linked module> -o als-reactor.js` before it.  GHC's post-linker
   writes nothing under a node older than 22.18, which lacks
   `import.meta.main`.
+  **One fixed work directory**: the binaries record the paths they were
   built at (a package's data directory, and in the server the absolute
   path of Agda's generated parser, in its source locations), so a build
   gives these bytes only in the same directory.  The plans show it.

The build was run twice on 2026-10-07, the second time from an empty
work directory at the same path, and gave the same files and the same
plans, byte for byte (27 minutes on a 20-core workstation).

## What was checked

+  **Agda's own test suites**, under wasmtime, as Agda's wasm workflow
   runs them, on a wasm `agda` built by the same command from the same
   sources and toolchain in another work directory (so its bytes differ
   from `agda.wasm`'s): Succeed 2,016 of 2,016, Fail 1,853 of 1,853, and
   Common pass; Bugs 14 of 15, the one failure (`Issue8182`) a golden
   value holding `-v` output, which a build without Agda's `debug` flag
   does not print (2026-10-07).
+  **The command module in a browser**: a page that runs `als.wasm` in a
   worker checks a file, gives at a goal, reports a type error at its
   position, and accepts the 41 builtin interfaces built by this
   `agda.wasm` (Chromium 149, 2026-10-07).
+  **The reactor in a browser**: `als-reactor.wasm` and `als-reactor.js`,
   driven as above from one module worker that also holds the WASI file
   system, on a page with `crossOriginIsolated` false and
   `SharedArrayBuffer` undefined, in Chromium 149 and Firefox 157
   (2026-10-07).  It checked a `Main` importing two builtins and one
   importing 32, with `Main` alone checked against the builtin interfaces
   this `agda.wasm` built, gave at a goal, reloaded, and answered
   `Cmd_show_version` with 2.9.0.  An earlier link of the same sources,
   before `wasm-opt`, gave the same responses, in the same order, as the
   command module linked beside it.

[release]: https://github.com/formalverification/agda-wasm-builds/releases/tag/agda-2.9.0-da66a8c-als-8
