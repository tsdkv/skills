---
name: property-testing
description: Write, review, and debug property-based tests in Go with pgregory.net/rapid — choosing what property to assert (roundtrips, oracles, invariants, metamorphic relations) and shaping rapid generators so tests exercise the interesting regime (small-value bias, IntRange min-skew, SampledFrom for uniform picks, fixed vocabularies). Use when writing or reviewing property tests, designing rapid generators, when a test asserts something trivial or vacuous, or when a shrunk counterexample needs interpreting. Not for coverage-guided fuzzing (go test -fuzz is fine as a complement), benchmarks, or example-based unit tests.
---

# Property-based testing

An example test asserts one point. A property asserts a rule over the whole
input domain and lets the generator hunt for the counterexample. Worth it
when the code has an algebraic shape — an inverse, an invariant, an oracle —
and not otherwise. If no shape exists, say so: example tests are the honest
answer.

## Property catalog

| Property | Formula | Where it applies |
|---|---|---|
| Roundtrip | `decode(encode(x)) == x` | Serialization, codecs, wire/disk formats |
| Inverse | `f(g(x)) == x` | compress/decompress, apply/undo |
| Oracle | `new(x) == reference(x)` | Optimization, reimplementation, brute force vs. fast path |
| Idempotence | `f(f(x)) == f(x)` | Normalization, dedup, retry, reopen |
| Invariant | holds before and after | Any transformation; internal structure (sortedness, counts) |
| Easy to verify | `check(result(x))` | Hard to compute, cheap to check (constraints, membership) |
| Commutativity | `f(a, b) == f(b, a)` | Binary/set operations, reordering of input |
| Associativity | `f(f(a,b), c) == f(a, f(b,c))` | Combining/merging operations |
| Identity | `f(x, e) == x` | Operations with a neutral element |

## Metamorphic relations

When no oracle exists, transform the input and predict a relation on the
outputs. Walk each family against each public operation; the ones that are
non-vacuous are tests.

| Family | Relation |
|---|---|
| Permutative | Reorder input → output unchanged or predictably reordered |
| Additive | Add input elements → output grows constrainedly (superset) |
| Multiplicative | Duplicate/scale input → output scales or preserves order |
| Invertive | Apply then undo → initial state |
| Inclusive | Run on a subset → output is a subset of the full run's |
| Exclusive | Disjoint inputs → disjoint outputs |

Strength ordering, weakest to strongest:
`no crash → invariant → idempotence → roundtrip / oracle`; equality beats
inequality beats smoke. Assert the strongest property the code supports —
"no crash" alone rarely justifies a property test.

## The two ways a property test asserts nothing

- **Tautology.** The property restates the implementation; no shared bug can
  fail it. Constrain the function without recomputing it. Exception:
  `f(x) == f(x)` is a real determinism property when `f` isn't obviously
  pure (map iteration order, clocks, hashes).
- **Vacuity.** A filter/`Skip` that rejects nearly every input passes
  without exercising anything. Push constraints into the generator so valid
  inputs are produced directly.

Both sides must have content: a subset relation is vacuous if generated
inputs rarely make both sides non-empty. Shape the generator, don't weaken
the property.

## rapid generators

rapid biases toward small, simple values on purpose — the bias is also the
shrink direction. Measured against v1.3.0 source:

| Generator | Actual distribution |
|---|---|
| `Int()`, `Uint64()`, ... | Bit-length geometric (mean ~16 bits for 64-bit), then uniform bits. ~45% of values < 1024 |
| `IntRange(min, max)` | Skewed toward `min`. Never uniform |
| `SliceOf(g)`, `String()` | Length geometric, mean ~5: ~17% empty, ~2% longer than 20 |
| `SliceOfN(g, min, max)` | Length still mean ~5 — max is decoration. `SliceOfN(g, n, n)` for exact length |
| `Rune()`, `String()` | 50% of chars from a 30-rune nasty set (NUL, ESC, BOM...). Good for parsers, useless for realistic text |
| `Float64()` | Exponent biased small: ~48% of values in (0, 1) |

Recipes:

```go
// uniform pick from a set — never IntRange + indexing (over-picks the front)
term := rapid.SampledFrom(vocab).Draw(rt, "term")

// fixed small vocabulary when behavior needs collisions
// (random strings make every value unique: df=1, empty intersections)
doc := rapid.SliceOfN(rapid.SampledFrom(vocab), 1, 50).Draw(rt, "doc")

// draw the size first, derive everything relative to it
n := rapid.IntRange(0, 100).Draw(rt, "n")
k := rapid.IntRange(-5, n+5).Draw(rt, "k")          // k < n actually happens
scores := rapid.SliceOfN(scoreGen, n, n).Draw(rt, "scores")
```

For several scale regimes (10 / 1k / 100k), `SampledFrom` over the regimes —
no default generator visits the large ones. Keep edge-case generators
(`IntRange(-5, ...)` for invalid input), but the min-skew means trivial paths
eat many runs: make the interesting regime reachable on most draws.

## Interpreting failures

rapid shrinks to a minimal counterexample and persists it under
`testdata/rapid/` — commit that directory, it replays in CI. Then decide:

- **Wrong property** if the counterexample is legitimate input the property
  never covered. Fix the property or the generator, not the code.
- **Wrong code** if the counterexample is minimal and clearly valid.
  Paste it into a unit test only to lock a real regression; otherwise the
  property test already covers it.

## References

- Wlaschin, [Choosing properties for property-based testing](https://fsharpforfunandprofit.com/posts/property-based-testing-2/)
- Chen et al., METRIC (MR identification via category-choice)
- Segura et al., A Survey on Metamorphic Testing
- [rapid docs](https://pgregory.net/rapid)
