---
name: formal-proof
description: "Formally verify critical invariants, concurrency protocols, arithmetic/unsafe kernels, and Lean 4 theorems using model checking (TLA+/Quint, Kani, ESBMC, loom/Lincheck/Fray) and deductive/theorem provers (Verus, OpenJML, Lean 4), with anti-cheating verdict gates, real-bug triage, and CI integration. Use for /formal-proof, 'formally verify this', 'model-check this protocol', 'prove this invariant', Lean 4 proving/auditing, or escalating a state-audit hazard. Not for routine unit, UI, or CRUD tests."
---

# Formal Proof

Formally verify high-stakes promises where example tests fall short: concurrency races, retries, distributed protocols, integer/index arithmetic, memory/unsafe kernels, and mathematical theorems. Jump straight to the right formal verifier, prove every check can fail before trusting a pass, never trust exit code `0` alone, and separate real bugs from pedantic noise.

## 1. Scope & reuse first

- **Reuse existing checks first.** Look for existing TLA+/Quint specs, Kani/ESBMC/Verus/OpenJML harnesses, `loom`/Lincheck checks, Lean modules, or CI jobs running them, and extend what exists.
- **Check drift before trusting existing checks.** If covered code changed since the check was last verified, diff the covered files (including moved functions and new writers to guarded state) and separate stale/drifted checks from intentional promise changes (backed by docs/tests/commits) and real bugs. Update models from the code, never to chase green.
- **Freeze specs before proving.** Derive invariants, models, or theorem statements from docs and requirements before reading implementation details when practical. Never weaken a property, assumption, or invariant to make a check pass.
- **Be honest about what formal tools can touch.** If the target is huge I/O, reflection, or STL/JDK-heavy code that no verifier can run on directly, either model its protocol/state machine in TLA+, extract a faithful small kernel, or stop and say it belongs to runtime testing/fuzzing rather than faking a toy model.

## 2. Choose the right formal verifier (agent's discretion)

Go directly to the tool that settles the claim without wasting turns warming up on weaker checks:
- **Concurrency & protocols (any language):** `TLA+` / `Quint` model checking (plus trace validation against real execution logs) for distributed/async protocols; `loom` (Rust), `Lincheck` / `Fray` (JVM), or `testing/synctest` (Go) for in-process thread interleavings.
- **Bounded code verification:** `Kani` (Rust), `ESBMC` (small C/C++ kernels), or exhaustive brute-force over small finite domains (e.g. 32-bit math).
- **Unbounded proofs:** `Verus` (pure Rust cores), `OpenJML` (small pure Java math kernels), or `Lean 4` (mathematical/algorithmic proofs and Lean/Mathlib repos).

Use your judgment on tool choice and bounds. For small finite protocol models in Quint or TLA+, verify exhaustively with TLC (nightly over stale 2.19) rather than relying on random simulation or bounded Apalache alone.

## 3. Silent exit-0 traps & verdict gates

Never trust exit code `0` alone. Inspect structured verifier outputs, execution reach, and direct exit codes (never masked through a pipe):

- **Kani:** Contradictory `assume` exits `0` and prints `SUCCESS` on `assert!`. Place a reachability sentinel (`kani::cover!`) after all `assume`s, run with timeouts/memory caps, and verify sentinels are satisfied with zero failed checks. Use concrete playback to turn counterexamples into unit tests.
- **Verus:** Run with `--no-cheating` and structured output, confirming `verified > 0` and zero errors. Watch for `--no-verify` (exits `0` with `verified: 0`), `ensures true`, contradictory `requires`, `#[verifier::external]` hiding target code, or `#[cfg(verus_only)]` split-brain code.
- **ESBMC:** Never pass `--no-unwinding-assertions`. Verify a reachable `assert(false)` probe fails (contradictory `__ESBMC_assume` exits `0`), and make sure extracted kernels actually compile the target body (missing definitions silently become nondeterministic).
- **TLA+ / TLC & Quint:** Model each lock release as its own step and confirm zero states left on queue, no silent `0:0` dead actions under coverage, and both a reachability probe and a seeded-bug config fail as expected. `SYMMETRY` is sound for safety only, not liveness. In TLC trace validation, both a pass and a safety violation can return exit code `12`, so distinguish them by the violated invariant name and verify bad traces are rejected. Quint prints identical `[ok]` for random sampling and exhaustive TLC, so verify which backend actually ran.
- **Concurrency checkers (`Lincheck` / `Fray` / `go test -race`):** `Lincheck` and `Fray` do not catch `notify` lost wakeups or missing `volatile`; `go test -race` misses lock-valid atomic semantic races. Confirm scheduling races against real code (or clearly label delay-injected repros).

## 4. Lean 4 & Mathlib

When proving or auditing Lean 4 theorems:
- **Keep statements frozen:** Never edit frozen statement files, `lean-toolchain`, or lockfiles. If a statement is false or misformalized, stop and present evidence (`¬ target`, counterexample, or inconsistent hypotheses).
- **Audit the statement before proving:** Confirm hypotheses are satisfiable (not vacuously false) and check for unguarded totalized junk values (`Nat.sub` truncation, `/ 0 = 0`, integer `x ^ (1 / 2)`). For algorithmic specs, verify broken implementations are rejected.
- **Don't trust `lake build` exit `0` or LSP output alone.** Before calling a proof done, verify:
  1. Full module build of every promoted module.
  2. Source scan for forbidden constructs (`sorry`, `sorryAx`, `admit`, `axiom`, unallowlisted `native_decide`, `@[implemented_by]`, `@[extern]`, `unsafe`, `partial`) because `#print axioms` is blind to `@[implemented_by]` on `unsafe def`.
  3. `#print axioms` whitelist (`{propext, Classical.choice, Quot.sound}`) over every promoted theorem (whitelist, not blacklist: Lean 4.36+ emits per-use `native_decide` axiom names), plus any unproved proposition hypotheses taken as arguments.
  4. Kernel replay (`lake env leanchecker`) with a negative control when available.
  5. Frozen statement comparison (`lake comparator`, kept under `$HOME` to avoid bubblewrap path masking, or SHA-256 diff check on older toolchains).

## 5. Prove the check can fail

Before trusting a green check, mutate the **real implementation** (or model under test, reverting all mutants before finishing) while keeping the spec frozen:
- Try domain mutants: flip guards, return a constant/unbounded value, drop a side effect, remove a lock/CAS, or replay a past bug.
- Distinguish a real **functional kill** (your invariant/property failed) from an incidental crash/timeout, and confirm mutated code actually ran. Avoid mirror specs that simply restate the implementation's own bug.

## 6. Real bugs vs. pedantic LLM noise

Before calling any finding a real bug:
1. **Check real-pathway reachability:** Replay any model counterexample step-by-step against the real code and confirm it is reachable through a real public entry point (not only by feeding impossible state to an internal helper). If only reachable internally, label it latent/internal or drop it.
2. **Filter for real impact:** Prioritize silent data corruption, OOB/memory safety, data loss, deadlocks/races, and crashes/DoS on valid or untrusted inputs. Do not inflate minor exception-type differences already wrapped by callers or preconditions enforced upstream.
3. **Sweep siblings & dedup upstream:** When a primitive breaks, check sibling callers of the same pattern. On OSS repos, search open and closed issues, open and closed PRs (plus JIRA for Apache repos), and recent commits/backports before calling a bug new. Never file issues or PRs upstream without explicit user approval.
4. **Report plainly and honestly:** State the claim, harm, and counterexample as concrete steps in the code's own terms so the reader doesn't need to know the verifier's syntax. Match claim strength strictly to what actually ran (unbounded proof vs bounded check with its exact bounds). Do not edit product code unless asked.

## 7. CI integration

- **If formal verification CI already exists in the repo:** integrate new harnesses, models, or proofs directly into the existing setup and keep any covered-files/sync-commit note current.
- **If no formal verification CI exists yet:** keep exploratory runs clean and non-invasive by default. When done (or after fixing a confirmed bug), offer the user a simple choice: wire the checks into the repo's CI (fast bounded checks on PR, heavier model-checking sweeps nightly) or remove the scratch files. Do not generate nested project-local meta-skills.
