---
name: property-testing
description: Use when writing, reviewing, or debugging property-based tests in Go with pgregory.net/rapid — including when a property has never failed, asserts something trivial, discards most inputs, or produces a shrunk counterexample that needs interpreting, and when testing stateful, concurrent, or eventually consistent code against a model. Not for coverage-guided fuzzing, benchmarks, or example-based unit tests.
---

# Property-based testing (Go, rapid)

A property asserts a rule over the whole input domain and lets the generator
hunt for the counterexample. Worth it when the code has an algebraic shape —
an inverse, an invariant, an oracle. If none exists, say so: example tests
are the honest answer.

## Procedure

1. **Define the contract:** valid domain, observable guarantees, equivalence
   relation, and permitted side effects. Separate guarantees from accidents.
2. **Select complementary properties** (catalog below).
3. **Write the generator and property; label every `Draw`.** Prefer
   construction over rejection. Stateful preconditions may skip unavailable
   actions; measure skips and ensure progress. Test specified invalid-operation
   behavior rather than skipping it.
4. **REQUIRED: plant a bug and confirm the property fails. Revert.**
   Flip a comparison or drop a branch. Still passes → the property or
   generator is blind; fix it before trusting either.
5. **Tally regimes and discards.** Count which branch each run hits and how
   many inputs are rejected (counter map in a scratch test; rapid has no
   `collect`). Reshape the generator until the interesting regime dominates.
6. **Commit `testdata/rapid/`.**

## Contract and equality

Define `==` before using it: roundtrips may preserve meaning rather than
representation. Specify float edge cases and collection semantics. Rejection
does not promise rollback; correct results do not prove input preservation.
Check side effects against the API's guarantees. Details: `contracts.md`.

## Property catalog

Use this as a selection aid, not a ranking. Prefer an independent reference
or simple model when available; a model that repeats the implementation can
share its bugs. Combine agreement, structural invariants, error behavior,
and input preservation. Roundtrips establish consistency, not independent
conformance to a specification. "No crash" alone is usually insufficient.

| Property | Formula | Applies to |
|---|---|---|
| Model-based | `abstract(f(x)) == fModel(abstract(x))` | An independent, simple model (sorted slice for a tree, `map` for a cache) |
| Oracle | `new(x) == reference(x)` | Optimization, reimplementation, brute force vs. fast path |
| Roundtrip / inverse | `decode(encode(x)) == x` | Codecs, wire/disk formats, compress, apply/undo |
| Postcondition | relation between args and result | Inputs built with known outcome: `find k (insert k v t) == v` |
| Easy to verify | `check(result(x))` | Hard to compute, cheap to check |
| Idempotence | `f(f(x)) == f(x)` | Normalization, dedup, retry, reopen |
| Invariant | holds before and after | Sortedness, counts, internal structure |
| Error rejection | `f(invalid)` rejects as specified | Parsers, validators: error/no panic when promised; rollback only for atomic failure |
| Algebraic laws | commutative, associative, identity | Only when the structure is really there |

No oracle? See `metamorphic.md`. When representation is incidental (tree
shape, bucket order), assert equivalence through an abstraction
(`ToList(a) == ToList(b)`), and generate equivalent pairs deliberately —
two independent draws are almost never equivalent.

## Two ways a property asserts nothing

- **Tautology.** The property recomputes the implementation, so no shared
  bug can fail it. Constrain the result without recomputing it. (`f(x) ==
  f(x)` is a real determinism check when `f` may read map order, clocks, or
  hashes.)
- **Vacuity.** A filter or `Skip` rejects nearly every input, or a relation
  has an empty side on most draws. Push constraints into the generator;
  don't weaken the property.

## Review checklist

| Tell | Fix |
|---|---|
| `IntRange` + indexing to pick from a set | `SampledFrom` |
| `SliceOfN(g, 1, 50)` as if 50 were reachable | `SliceOfN(g, n, n)`, or `SampledFrom` over size regimes |
| Filter / `Skip` rejects most inputs | Generate by construction |
| Map order determines draws, operations, or replay choices | Stabilize order; member-only checks need no sorting |
| Property recomputes the implementation | Constrain, don't recompute |
| Only "no crash" asserted | Add contract-backed, complementary properties |
| Related inputs drawn independently | Correlate (reuse existing keys) |
| Property has never failed | Mutation check (step 4) |
| Generator builds values via the code under test | Add a generator-validity property (`generators.md`) |

## rapid generators

rapid biases toward small, simple values on purpose; the bias is also the
shrink direction. Measured on v1.3.0:

| Generator | Actual distribution |
|---|---|
| `Int()`, `Uint64()` | Bit-length geometric (mean ~16 bits for 64-bit). ~45% of values < 1024 |
| `IntRange(min, max)` | Skewed toward `min`. Never uniform |
| `SliceOf(g)`, `String()` | Length geometric, mean ~5: ~17% empty, ~2% longer than 20 |
| `SliceOfN(g, min, max)` | Length still mean ~5 — max is decoration. `SliceOfN(g, n, n)` for exact length |
| `Rune()`, `String()` | 50% of chars from a 30-rune nasty set (NUL, ESC, BOM…). Good for parsers, useless for realistic text |
| `Float64()` | ~48% of values in (0, 1) |

```go
// uniform pick from a set — never IntRange + indexing
term := rapid.SampledFrom(vocab).Draw(rt, "term")

// small vocabulary when behavior needs collisions (random strings are all unique)
doc := rapid.SliceOf(rapid.SampledFrom(vocab)).Draw(rt, "doc")

// draw the size first, derive the rest relative to it
n := rapid.IntRange(0, 100).Draw(rt, "n")
k := rapid.IntRange(-5, n+5).Draw(rt, "k") // k < n actually happens
scores := rapid.SliceOfN(scoreGen, n, n).Draw(rt, "scores")

// correlate: choose an existing key or an arbitrary key (not a guaranteed miss)
var k int
if len(keys) > 0 && rapid.Bool().Draw(rt, "hit") {
    k = rapid.SampledFrom(keys).Draw(rt, "kExisting")
} else {
    k = rapid.IntRange(0, 100).Draw(rt, "kArbitrary")
}
```

Deliberately reach both hits and misses: the arbitrary branch can hit too.
For guaranteed misses in a finite domain, draw from its absent keys; if none
remain, generate a non-full structure or test that full-domain case separately.
For several scale regimes (10 / 1k / 100k), `SampledFrom` over the regimes —
no default generator visits the large ones.
Shrinking, swarm configurations, and generator validity: `generators.md`.

## Stateful, concurrent, and eventually consistent code

Use `(*rapid.T).Repeat` with a model and a pool of live handles; compare
after every action. Full idiom, a runnable example, and crash,
failure-injection, and serializability patterns: `stateful.md`.

## Running and replaying

- Start every property invocation, including shrink attempts, from fresh
  state. Keep model and SUT mutable data separate; stop and join background
  workers before returning. See `stateful.md` for isolation pitfalls.
- Control clocks, randomness, services, environment, and scheduling; capture
  all nondeterministic choices needed for replay. A seed reproduces generated
  inputs, not necessarily a failing thread schedule. Observe completion,
  rather than treating sleeps as proof that asynchronous work finished.
- Budget like a unit test: cheap in CI, raise `-rapid.checks` nightly.
- Failures print the exact replay flag (`-rapid.failfile=...` or
  `-rapid.seed=N`); rerun with it. `-rapid.v` shows all draws.
- `rapid.MakeFuzz(prop)` turns a property into a `go test -fuzz` target:
  `f.Fuzz(rapid.MakeFuzz(func(t *rapid.T) { ... }))`.
- Ship generators with types: `func WidgetGenerator() *rapid.Generator[Widget]`.

## Interpreting failures

rapid shrinks a failing case and writes it under `testdata/rapid/` — commit
it. Diagnose against the contract, not the size of the witness:

- **Wrong property or generator** — the assertion imposes an unpromised
  guarantee, or generated inputs violate a precondition. Fix the responsible
  test component; valid inputs exposing a contract violation are real bugs.
- **Wrong model or replay setup** — check model assumptions and case isolation
  when a precondition fails during replay; reasonable-looking behavior alone
  does not establish which side is right.
- **Wrong code** — valid inputs expose a violated guarantee. Paste the case
  into a unit test only to lock a real regression.

Treat the shrunk case as a small witness, not a global minimum or proof of
necessity. Absent features are unnecessary for this reproduction: no duplicate
values means duplicates are not needed, not that distinctness is required.
Test nearby variants before attributing the bug to remaining features. After
the first fix, keep running: one property can find several distinct bugs.
Residual boilerplate may be the generator's floor.

## Supporting files

- `contracts.md` — valid domains, equality, side effects, oracle independence
- `stateful.md` — `Repeat`, example, crash/fault/serializability patterns
- `generators.md` — shrinking mechanics, generator validity, swarm testing
- `metamorphic.md` — relation families when no oracle exists
- `references.md` — sources and the measurements behind the advice
