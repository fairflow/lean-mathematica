# Development

How this project is organized, the conventions the code follows, how to build
and check it, and how CI is currently wired. For what's actually proved, see
[MANUAL.md](MANUAL.md); for setup and first use, see [GUIDE.md](GUIDE.md).

## 1. Layout

```
Mathematica.lean       -- root import module: pulls in every Mathematica/*.lean file
Mathematica/            -- the Lean side of the bridge (11 files, see MANUAL.md §2)
Reverse/                -- the reverse bridge: lean_verify, a headless verification service
examples/                -- demos + the creative-telescoping case study (need a live kernel,
                            except CreativeTelescoping.lean, which is pure Lean)
wolfram/                 -- the Wolfram side: lean_form.wl (translation rules, OutputFormat,
                            WZCert), lean_verify.wl (LeanCheck), and .wl/.wls tests
src/                     -- the original Lean 3 sources (client2.py, server2.m, lean_form.m,
                            mathematica.lean, mathematica_parser.lean) — kept for reference
                            only; not built by this project's lakefile
docs/                    -- design docs: REVERSE_BRIDGE_DESIGN.md, zulip-announcement.md
lakefile.toml            -- Lake project file (Lean 4 / mathlib4)
lean-toolchain            -- pins leanprover/lean4:v4.31.0
leanpkg.toml               -- the original Lean 3 package file, kept alongside src/
```

The lakefile declares two libraries and one executable:

```toml
[[lean_lib]]
name = "Mathematica"      # defaultTargets — this is what `lake build` builds

[[lean_lib]]
name = "Reverse"

[[lean_exe]]
name = "lean_verify"
root = "Reverse.LeanVerify"
```

`examples/` is intentionally **not** a lakefile target — those files call a live
Wolfram kernel (`examples/CreativeTelescoping.lean` is the one exception; it
happens to be pure Lean and is checked directly with `lake env lean`, see §3).

## 2. Where this came from

This is a Lean 4 + mathlib4 port and extension of Rob Lewis and Minchao Wu's
Lean 3 `mathematica` bridge (`upstream` remote:
`robertylewis/mathematica`). [`MIGRATION.md`](MIGRATION.md) is the design
document for the port: it maps each Lean 3 file to its Lean 4 counterpart,
walks through the `Expr` constructor changes that mattered, explains why the
Lean 3 pexpr/expr split disappears, and records the open questions from the
port. Read it before touching `Mathematica/Reflect.lean`,
`Mathematica/Translate.lean`, or `Mathematica/Unreflect.lean` — those three
files carry almost all of the porting subtlety.

Beyond the port, this project extends the original with: a **sound** tactic
family (`mathematica_ring`, `mathematica_rw`, `mathematica_telescope` — none of
which existed in the Lean 3 version, which only had the trust-axiom
`mathematica_simp`), the **reverse bridge** (`Reverse/`, `LeanCheck` from
Wolfram), the creative-telescoping case study (`examples/CreativeTelescoping.lean`,
`ZeilbergerBridge.lean`), and the infoview graphics widget
(`Mathematica/Widget.lean`).

## 3. Building and checking

```sh
lake exe cache get      # fetch prebuilt mathlib .olean's
lake build               # builds Mathematica + Reverse (the lakefile's default targets);
                          # no Wolfram kernel required
```

Reconfirmed while writing this documentation: `lake build Mathematica` completes
in 1261 jobs with no errors, and there is no `sorry` anywhere in the repository
(`grep -rn "sorry" --include="*.lean" .` — no matches). See
[MANUAL.md](MANUAL.md) for the full breakdown, including what was and wasn't
independently re-verified.

To check the case study directly (pure Lean, no kernel):

```sh
lake env lean examples/CreativeTelescoping.lean
```

which both type-checks the file and, via its trailing `#print axioms ...`
lines, prints the axiom list for each theorem — the way to confirm a proof is
sound (only `propext, Classical.choice, Quot.sound`) rather than resting on
the project's `Mathematica.trust` oracle axiom.

To run anything that talks to Wolfram (`examples/Demos.lean`,
`examples/ZeilbergerBridge.lean`, any file using `mathematica_simp`/`_rw`/`_ring`/
`_telescope`/`#mathematica`/`mathematica%`), set the environment variables and use
`lake env lean` rather than `lake build` — see [GUIDE.md](GUIDE.md) §3. There is
no automated test suite that exercises the live-kernel path; verification there
is currently by hand (running the demo files) plus the Wolfram-side unit tests
in `wolfram/lean_form_test.wls` (`wolframscript -file wolfram/lean_form_test.wls`).

## 4. CI

Two GitHub Actions workflows exist ([`.github/workflows/`](.github/workflows/)),
both inherited from the `leanprover-contrib` tooling used across the Lean
community project ecosystem:

- **`update_versions.yml`** — on every push to `master`, updates the
  `lean-x.y.z`-named branches and builds the project (`leanprover-contrib/
  update-versions-action` + `lean-build-action`).
- **`upgrade.yml`** — runs nightly (`0 2 * * *`), bumps the pinned Lean/mathlib
  versions via `leanprover-contrib/lean-upgrade-action`, and re-runs the
  version-branch update.

Neither workflow currently builds `examples/` or runs anything against a live
Wolfram kernel — that isn't something CI can do without a Wolfram license and
installation, so it stays a manual step (§3). There is no workflow that greps
for `sorry` or fails a PR that introduces one; that check is currently manual
too (see MANUAL.md §0 for the command).

## 5. Proof-style conventions

Reading the source (particularly `examples/CreativeTelescoping.lean` and the
`Mathematica/*.lean` docstrings) makes a few conventions visible:

- **Soundness is explicit and labelled.** Every tactic's docstring states
  outright whether it can produce `Mathematica.trust` in `#print axioms`
  (`mathematica_simp` can; `mathematica_ring`/`mathematica_rw`/
  `mathematica_telescope`'s closing lemmas cannot). Case-study files end with
  `#print axioms` calls on their headline theorems as a standing, visible
  check rather than a one-off claim in a comment.
- **Certificates are Wolfram's job, checking is Lean's.** The recurring
  pattern (`Ring.lean`, `Rewrite.lean`, `Telescope.lean`,
  `CreativeTelescoping.lean`) is: ask Wolfram for a candidate identity or
  simplification, then discharge it with a small, standard closing tactic
  (`ring1`, `field_simp; ring`, `linear_combination`, `norm_num`) so the
  kernel — not the CAS — is what actually certifies the result.
  `Mathematica/Ring.lean`'s own docstring frames this explicitly as the
  Wolfram analogue of `polyrith`.
- **Numeric division is handled by rewriting to a boundary-safe form, not by
  side conditions bolted on after the fact.** `CreativeTelescoping.lean`'s L1b
  section is the worked example: the raw WZ certificate has a `/(n+1−k)²` that
  is ill-defined at the top of the range, so the proof rewrites the certificate
  into an equivalent form with a denominator that is never zero, rather than
  special-casing the boundary term.
- **Every module docstring records Lean-3 provenance and the specific Lean 4
  redesign decision**, with a pointer into `MIGRATION.md` by section number
  (e.g. `Reflect.lean` → MIGRATION.md §4, §7). New files touching the
  translation core are expected to keep that trail.
- **House style**: `autoImplicit = false` is set project-wide in
  `lakefile.toml`, matching the sibling Lean 4 project convention referenced
  in the file's own comment.

## 6. Contributing

- Read [MIGRATION.md](MIGRATION.md) first if the change touches `Reflect.lean`,
  `Translate.lean`, or `Unreflect.lean` — the porting rationale there explains
  why several things are shaped the way they are (e.g. why `Wire.lean` still
  parses over `List Char` rather than `String.Iterator`/`Std.Internal.Parsec`,
  a known, flagged efficiency TODO rather than an oversight).
- If you add a tactic or lemma that closes goals via Wolfram, be explicit in
  the docstring and in a `#print axioms` line about whether it can introduce
  `Mathematica.trust` — that distinction is load-bearing for how the rest of
  the project (and this documentation) describes results as sound versus
  oracle-backed.
- Run `grep -rn "sorry" --include="*.lean" .` before opening a PR; there are
  currently none in the tree and the project's claims in [MANUAL.md](MANUAL.md)
  depend on that staying true.
- `lake build` should stay kernel-free. If a new file needs a live Wolfram
  kernel to build, keep it out of the `Mathematica`/`Reverse` library targets
  (follow the existing `examples/` pattern) so `lake build` keeps working in
  environments without Wolfram installed, including CI.
