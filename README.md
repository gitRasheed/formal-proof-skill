# formal-proof

A coding-agent skill for formal verification: model checking (`TLA+`/`Quint`, `Kani`, `ESBMC`, `loom`/`Lincheck`), deductive verification (`Verus`, `OpenJML`), and `Lean 4` theorem proving.

Built for frontier coding agents to verify concurrency protocols, state machines, arithmetic/unsafe kernels, and mathematical theorems without falling for fake `exit 0` passes or pedantic LLM bug reports.

## How it works

1. **Go straight to the right verifier.** Picks the formal tool that settles the claim (`TLA+`/`Quint` for protocols and concurrency, `Kani`/`ESBMC` for bounded code checks, `Verus`/`OpenJML`/`Lean 4` for unbounded proofs) instead of wasting turns warming up on weaker checks.
2. **Never trust `exit 0` alone.** Guards against real silent-pass traps: Kani vacuous `assume` sentinels, Verus `--no-verify`/`#[verifier::external]`, ESBMC unwinding/nondet traps, TLC trace-validation `rc 12` ambiguity, Quint `[ok]` vs exhaustive TLC, and a 5-layer Lean 4 kernel/axiom/comparator gate.
3. **Prove every check can fail.** Mutates the implementation against the frozen spec before trusting a green pass, catching vacuous invariants and mirror specs.
4. **Separate real bugs from LLM noise.** Replays counterexamples against real public entry points, filters for real impact (data loss, corruption, memory safety, races), sweeps sibling callers, and deduplicates against upstream issues and PRs.
5. **Fits into CI cleanly.** Integrates into existing formal CI checks (diffing covered code to tell drift from bugs), or runs cleanly first and asks whether to wire the checks into CI or remove the scratch files.

## Install and use

```bash
npx skills add https://github.com/gitRasheed/formal-proof-skill --skill formal-proof -g
```

Add `-a claude-code`, `-a codex`, or `-a antigravity` for a single harness. Run it with `/formal-proof` or `$formal-proof`, optionally scoped to a path or area:

```text
/formal-proof src/consensus
```

Works well alongside [state-audit](https://github.com/gitRasheed/state-audit-skill): map implicit state machines with `/state-audit`, then verify concurrency or invariant claims with `/formal-proof`.
