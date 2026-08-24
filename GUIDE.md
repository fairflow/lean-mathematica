# Getting Started

This is a Lean 4 library (built on mathlib4), not an application — there is nothing
to deploy or run as a service. "Using" it means importing `Mathematica` into your
own Lean project and calling its tactics, or opening this repo and building it
yourself. This guide covers both. For the full tactic reference and internals, see
**[USER_GUIDE.md](USER_GUIDE.md)**; this file is the short path to a first build.

## 1. Prerequisites

- **Lean/Lake toolchain**: this project pins `leanprover/lean4:v4.31.0` via
  [`lean-toolchain`](lean-toolchain) and depends on `mathlib4` at `v4.31.0`
  ([`lakefile.toml`](lakefile.toml)). Install [`elan`](https://github.com/leanprover/elan)
  if you don't already have a Lean toolchain manager — it reads `lean-toolchain`
  automatically and fetches the right version.
- **Wolfram (Mathematica)**, only if you want to *run* the tactics, not just build
  the library. `lake build` alone (below) never touches Wolfram — it only compiles
  the Lean 4 source, so CI and casual browsing need no Wolfram installation at all.
  A live kernel is required for `mathematica_simp`, `mathematica_rw`,
  `mathematica_ring`, `mathematica_telescope`, `#mathematica`, `mathematica%`, and
  the demo/example files, since those actually talk to `WolframKernel` over
  stdin/stdout.

## 2. Clone and build the library

```sh
git clone https://github.com/fairflow/mathematica-in-lean.git
cd mathematica-in-lean
lake exe cache get   # fetch prebuilt mathlib .olean's (saves rebuilding mathlib)
lake build            # builds the `Mathematica` library (the lakefile's default target)
```

`lake build` compiles `Mathematica/` (and `Reverse/`) only — it does not need a
Wolfram kernel and does not build `examples/`, which are excluded from the default
target because they call a live kernel. This build was reconfirmed while writing
this guide: `lake build Mathematica` completes cleanly with no errors.

## 3. Point it at a Wolfram kernel

Only needed to actually *run* a tactic, not to build the library.

```sh
export MATHEMATICA_BRIDGE_LEANFORM="$(pwd)/wolfram/lean_form.wl"   # required, absolute path
export MATHEMATICA_BRIDGE_KERNEL=/path/to/WolframKernel             # optional; defaults to
                                                                      # the usual macOS location
```

Then run a file that uses the bridge with `lake env lean <file>` (not `lake build`,
which deliberately skips anything requiring a live kernel):

```sh
lake env lean examples/Demos.lean
```

## 4. First example

Once the environment variables above are set and a kernel is reachable, a minimal
use of the bridge in your own file looks like:

```lean
import Mathlib.Data.Real.Basic
import Mathematica

open Mathematica

-- sound: Mathematica finds the certificate, Lean's `ring1` checks it —
-- `#print axioms` stays free of any trust axiom.
example (x : ℝ) (h : x - 1 ≠ 0) : (x ^ 2 - 1) / (x - 1) = x + 1 := by mathematica_rw

-- compute a value in Mathematica, bring it back as a Lean term
#eval (mathematica% "Prime[100]" : Nat)   -- 541
```

If you only want to read or build on the *sound* parts of the library (no live
kernel, no trust axiom) — e.g. reusing the certificate-checking pattern from the
flagship case study — see
[`examples/CreativeTelescoping.lean`](examples/CreativeTelescoping.lean), which is
pure Lean and builds under plain `lake build`/`lake env lean` with no Wolfram
kernel at all.

## 5. Where to go next

- **[USER_GUIDE.md](USER_GUIDE.md)** — every tactic, transports, and how the
  bridge works under the hood.
- **[MANUAL.md](MANUAL.md)** — reference catalogue of what's actually formalised
  here, module by module, each marked proved/sorry-free or open.
- **[DEVELOPMENT.md](DEVELOPMENT.md)** — project layout, proof-style conventions,
  build/CI, and how to contribute.
- **[MIGRATION.md](MIGRATION.md)** — the Lean 3 → Lean 4 port map and design
  rationale, useful if you're touching `Mathematica/Translate.lean`,
  `Reflect.lean`, or `Unreflect.lean`.
