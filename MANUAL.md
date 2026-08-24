# Manual — what's actually formalised here

A catalogue of the library's content by file, with an honest status for every
theorem: **PROVED** means machine-checked and `sorry`-free (verified below by
building the file and, where noted, by `#print axioms`); **OPEN** means it
contains a `sorry` or is otherwise incomplete. Nothing in this repository is
currently OPEN — see [§0](#0-headline-finding).

For how to use any of this, see [GUIDE.md](GUIDE.md) (quick start) and
[USER_GUIDE.md](USER_GUIDE.md) (full tactic reference). For *why* things are
built the way they are, see [DEVELOPMENT.md](DEVELOPMENT.md) and
[MIGRATION.md](MIGRATION.md).

## 0. Headline finding

```sh
grep -rn "sorry" --include="*.lean" .
```

returns **no matches anywhere in the repository** (`Mathematica/`, `Reverse/`,
`examples/`, and the reference-only `src/`). `lake build Mathematica` completes
cleanly (1261 jobs, no errors, reconfirmed while writing this manual), and
`lake env lean examples/CreativeTelescoping.lean` type-checks with every theorem's
axiom list limited to the three standard mathlib axioms
(`propext, Classical.choice, Quot.sound`) — no `sorry`, no project-specific
trust axiom.

This project is a mix of two kinds of content, and the "proved" label means
different things for each:

- **`Mathematica/` and `Reverse/`** are *implementation*, not mathematical
  claims — a parser, a reflection engine, a translation engine, and tactic
  code. There are no `theorem`/`lemma` declarations here to be sorry-free or
  not; what "complete" means is that the code compiles and the tactics behave
  as documented. It does compile, cleanly, with no `sorry` and no
  `sorry`-shaped escape hatches (`admit`, `native_decide` is not used either).
- **`examples/`** contains actual `theorem`s. These are individually marked
  below, because not all of them are equally trustworthy: some are proved
  *soundly* (kernel-checked, no oracle), and some are proved by *trusting*
  Wolfram outright via a declared axiom — the codebase is explicit and
  consistent about which is which, and this manual preserves that distinction
  rather than flattening it.

## 1. The trust model, precisely

One axiom is declared in the whole codebase:

```lean
-- Mathematica/Tactic.lean:159
axiom trust {P : Prop} (P' : Prop) (h : P') : P
```

Only `mathematica_simp` uses it. Everything else that closes a goal — the
tactics below marked "sound" and the entire `CreativeTelescoping.lean` case
study — produces a proof whose axiom list is the ordinary mathlib set.
`#print axioms` is the check: if `Mathematica.trust` doesn't appear, nothing
was taken on faith from the Wolfram kernel.

## 2. `Mathematica/` — the bridge library

No `theorem`/`lemma` declarations; each file is a module of the bridge, all
present, all compiling, no `sorry`:

| File | Role |
|---|---|
| [`MMExpr.lean`](Mathematica/MMExpr.lean) | The wire-protocol AST (`MMExpr`) and the `MFloat` record. Port of Lean 3's `mmexpr`/`mfloat`. |
| [`Wire.lean`](Mathematica/Wire.lean) | Parser for the terse wire grammar (`I[...]`, `T["..."]`, `Y[...]`, `A hd[...]`) that Wolfram's `OutputFormat` emits. |
| [`Reflect.lean`](Mathematica/Reflect.lean) | `Expr → String`: reflects a Lean `Expr` into the `Lean…[…]` forms the Wolfram side pattern-matches on. Runs in `MetaM` since Lean 4 `fvar`/`mvar`s need local-context lookups the Lean 3 version didn't. |
| [`Unreflect.lean`](Mathematica/Unreflect.lean) | The inverse structural leaves: `MMExpr → Level`/`Name`/`BinderInfo`. |
| [`Translate.lean`](Mathematica/Translate.lean) | The rule engine: `MMExpr → MetaM Expr`, the core of turning a Wolfram result back into a real Lean term (implicit args and instances synthesised via `mkAppM`; binders via `MetaM` telescopes). |
| [`Tactic.lean`](Mathematica/Tactic.lean) | Transports (`Transport.persistentKernel` spawns one long-lived `WolframKernel` over stdin/stdout; `mockTransport` for kernel-free testing), `evalWolfram`/`runCommandOn*`, and the `trust` axiom + `mathematica_simp`. |
| [`Ring.lean`](Mathematica/Ring.lean) | `mathematica_ring` — sound certificate mode: Wolfram's `PolynomialReduce` *finds* a `linear_combination` certificate, Lean's `ring1`/`linear_combination` *checks* it. |
| [`Rewrite.lean`](Mathematica/Rewrite.lean) | `mathematica_rw` — sound fixed-point subterm rewriting: Wolfram simplifies a subterm, a Lean certificate tactic (`rfl`/`ring1`/`field_simp;ring1`/`norm_num`/`simp`) validates each step. |
| [`Telescope.lean`](Mathematica/Telescope.lean) | `mathematica_telescope` — fetches a WZ (Wilf–Zeilberger) certificate live via the bridge (`WZCert`) and closes binomial-sum identities for the supported family (`p = 1, 2`), using the same closed-form lemmas verified by hand in `CreativeTelescoping.lean`. The file's own docstring is explicit that turning an *arbitrary* fetched certificate into an auto-generated boundary-correct proof is future work, not yet implemented — the current scope is intentionally narrow. |
| [`Syntax.lean`](Mathematica/Syntax.lean) | `mathematica%` (term) and `#mathematica` (command) embedding syntax. |
| [`Widget.lean`](Mathematica/Widget.lean) | `#mathematica_plot` — renders Wolfram graphics (anything `Export`-able to PNG) in the Lean infoview via ProofWidgets. |

## 3. `Reverse/` — the reverse bridge

| File | Role |
|---|---|
| [`LeanVerify.lean`](Reverse/LeanVerify.lean) | `lean_verify`, a headless executable (`lake exe lean_verify`): reads one wire-form claim per line on stdin, translates it to a `Prop` via the existing `Wire.parse` → `exprOfMMExpr` pipeline, tries `intros`/`decide`/`omega`/`rfl` (core tactics only — imported mathlib `elab` tactics don't dispatch in a standalone `importModules` process), and if it succeeds **adds the proof to the kernel**, then prints the verdict with its real axiom list. Design doc: [`docs/REVERSE_BRIDGE_DESIGN.md`](docs/REVERSE_BRIDGE_DESIGN.md). Labelled "P0 spike" in its own header — a working first cut of the reverse direction, not yet the full claim language `LeanCheck` on the Wolfram side implies. |

## 4. `examples/` — theorems, by status

### 4.1 `CreativeTelescoping.lean` — the flagship case study

Pure Lean, no live kernel needed to build (`lake env lean
examples/CreativeTelescoping.lean` was run to confirm this while writing this
manual). Four theorems, **all PROVED, sorry-free, axiom-clean**
(`#print axioms` → `[propext, Classical.choice, Quot.sound]` only, verified
directly, not just asserted in the source comments):

| Result | Statement | Status |
|---|---|---|
| `odd_sum` | `∑_{k<n} (2k+1) = n²` | PROVED — warm-up, hand-supplied certificate `G(k) = k²` |
| `S_succ` / `sum_choose` | `∑_{k=0}^{n} C(n,k) = 2ⁿ` | PROVED — the recurrence `S(n+1) = 2S(n)` via Pascal's rule, then induction |
| `wz_cert` | The WZ certificate identity for `(n+1)S(n+1) = (4n+2)S(n)`, `R(n,k) = k²(2k−3n−3)/(n+1−k)²` | PROVED — a rational-function identity discharged by `field_simp; ring` |
| **`sum_choose_sq`** | **`∑_{k=0}^{n} C(n,k)² = C(2n,n)`** (`Nat.centralBinom n`) | **PROVED, end to end** — the certificate found by Wolfram's `WZCert`, verified sound in Lean, telescoped over the boundary-safe form `Gc`, and closed by induction from `T_eq_central` |

`sum_choose_sq` is the project's flagship result: a nontrivial binomial identity
whose Wilf–Zeilberger certificate was *discovered* by Wolfram and *independently
verified* by Lean's kernel, with no oracle axiom anywhere in the proof.

### 4.2 `Demos.lean` — tactic demonstrations, mixed soundness

Requires a live Wolfram kernel to build (deliberately excluded from the default
`lake build` target — run with `lake env lean examples/Demos.lean` after setting
`MATHEMATICA_BRIDGE_LEANFORM`/`MATHEMATICA_BRIDGE_KERNEL`). No Wolfram kernel was
available in the environment used to write this manual, so these were **not**
independently re-run here; the table below reflects what the tactics are
documented and designed to do, consistent with the sound/oracle split confirmed
directly in §4.1 and in the `Mathematica/Ring.lean`/`Rewrite.lean`/`Tactic.lean`
source.

| Theorem(s) | Tactic | Soundness |
|---|---|---|
| `pythagorean`, `binomial`, `diff_of_squares`, `arith`, `add_zero'` | `mathematica_simp` | Oracle — closes via `Mathematica.trust`; the file's own comment states this and shows in `#print axioms` |
| `binomial_sound`, `with_hyp` | `mathematica_ring` | Sound by design (Wolfram finds a certificate, `ring1`/`linear_combination` checks it) — the file asserts `#print axioms` shows only `[propext, Classical.choice, Quot.sound]` |
| `rw_poly`, `rw_rational`, `rw_numeric`, `rw_subterm` | `mathematica_rw` | Sound by design, same basis |
| `tele_choose`, `tele_choose_sq` | `mathematica_telescope` | Fetches a certificate live via the bridge and closes via the library lemma from §4.1 |

### 4.3 `ZeilbergerBridge.lean` — live certificate discovery

Also requires a live kernel (same caveat as §4.2 — not independently re-run
here). Its stated purpose: unlike `CreativeTelescoping.lean`, where the
certificate was found in a separate Wolfram session and hand-transcribed, this
file calls `WZCert` through the bridge *at elaboration time* to fetch the same
certificate that `sum_choose_sq` verifies. It closes the "discovery" gap left
open by §4.1; it contains no new theorem statements of its own beyond driving
the discovery step.

## 5. Summary table

| Area | sorry count | Independently re-verified in this pass |
|---|---|---|
| `Mathematica/` (11 files, bridge implementation) | 0 | Yes — `lake build Mathematica` clean |
| `Reverse/LeanVerify.lean` | 0 | Yes — `lake build` (part of default targets) clean |
| `examples/CreativeTelescoping.lean` (4 theorems) | 0 | Yes — built directly, axiom lists checked |
| `examples/Demos.lean` (13 theorems) | 0 (per source) | No — needs a live Wolfram kernel, unavailable here |
| `examples/ZeilbergerBridge.lean` | 0 (per source) | No — needs a live Wolfram kernel, unavailable here |
| `src/` (original Lean 3 sources) | reference only, not built by this project | N/A |
