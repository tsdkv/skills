---
name: property-testing
description: Write, review, and debug property-based tests in Go with pgregory.net/rapid — choosing what property to assert (roundtrips, oracles, model-based, invariants, metamorphic relations), shaping rapid generators so tests exercise the interesting regime (small-value bias, IntRange min-skew, SampledFrom for uniform picks, fixed vocabularies, correlated inputs, per-run swarm configurations), validating that a test can actually fail (mutation checks, sample inspection, discard counting), and testing stateful, eventually consistent, or concurrent systems against a model (op-sequence replay, crash and failure injection, serializability). Use when writing or reviewing property tests, designing rapid generators, when a test asserts something trivial or vacuous, when a property has never failed, or when a shrunk counterexample needs interpreting. Not for coverage-guided fuzzing (go test -fuzz is fine as a complement), benchmarks, or example-based unit tests.
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
| Model-based | `abstract(f(x)) == fModel(abstract(x))` | A simple model stands in for the system (sorted list for a tree, `map` for a cache). The oracle generalized |
| Postcondition | relation between args and result | Construct inputs with known outcome: `find k (insert k v t) == v` |
| Error rejection | `f(invalid)` errors cleanly | Parsers, validators: error returned, no panic, no partial state |
| Idempotence | `f(f(x)) == f(x)` | Normalization, dedup, retry, reopen |
| Invariant | holds before and after | Any transformation; internal structure (sortedness, counts) |
| Easy to verify | `check(result(x))` | Hard to compute, cheap to check (constraints, membership) |
| Commutativity | `f(a, b) == f(b, a)` | Binary/set operations, reordering of input |
| Associativity | `f(f(a,b), c) == f(a, f(b,c))` | Combining/merging operations |
| Identity | `f(x, e) == x` | Operations with a neutral element |

Roundtrip and oracle/model-based dominate real usage (~71% of an
86-test Hypothesis corpus; the most-used pattern in Jane Street
interviews) — look for them first. The algebraic laws (commutativity,
associativity, identity) almost never appear in the wild; write them
when the structure is really there.

Model-based is the strongest bug-finder per property: against 8 buggy
BST variants, model-based properties found every bug in their operation
and failed fastest (~8 vs. ~50 tests for an equivalent postcondition)
because they validate the whole result, not a sampled key.

Error rejection punches above its weight: across 40 projects under
mutation testing, each property test killed ~50× as many mutants as the
average unit test, and exception-raising and collection-inclusion
checks were the most effective categories (>19× over the rest).

```go
// abstract the concrete tree to a sorted []kv model, then compare
got := ToList(Insert(t, k, v))
want := ModelInsert(ToList(t), k, v) // model ops on plain slices
```

### Equivalence, not equality

When representation is incidental (tree shape, bucket order), define
equivalence through the abstraction (`ToList(a) == ToList(b)`) and
assert that — not structural equality. Then test that each operation
preserves equivalence. Generate equivalent pairs deliberately (same
inserts, different order): two independent draws are almost never
equivalent, so the naive property discards nearly every case.

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
| Dead-zone (EMI) | Mutate input regions that cannot affect the result → output identical |

Choosing good relations (Segura et al.):

- Prefer relations whose follow-up run diverges from the source run —
  different paths taken correlates with fault detection.
- Tight relations (equality) beat loose ones (inequality); component-level
  relations beat whole-system ones.
- n operations × n possible input modifiers = O(n²) candidate relations.
  Walk the cross-product per operation, not just the six families.

The dead-zone family deserves its own recipe: find the part of the input
that cannot affect the result for this operation — fields a parser
ignores, elements beyond a limit, key order in a map — and mutate only
it, demanding a bit-identical output. This is differential testing with a
single implementation: the oracle is the unchanged output, no reference
needed. It found 147 confirmed bugs in GCC/LLVM by pruning never-executed
code: variants take different optimizer paths but must produce the same
result.

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

## Prove your tests can fail

A property that has never failed is unproven — it may assert nothing, or
the generator may never reach the buggy regime. Over a third of Jane
Street's experienced PBT users admitted they don't interrogate passing
properties. Validate the test itself:

- **Mutation check.** Plant a bug — flip a comparison, drop a branch —
  and confirm the property fails. Revert after. If it still passes, the
  property or generator is blind; fix it before trusting either.
- **Inspect samples.** Eyeball a few generated values (`gen.Example()`,
  or `t.Log` in a scratch run). If they all look alike, the distribution
  is wrong no matter what the code says.
- **Count the regimes.** Hughes measured 80% of runs hitting the trivial
  branch (key absent from the tree) — most effort wasted on cases that
  can't fail. rapid has no `collect`/`label`; tally outcomes in a counter
  map in a scratch test, then reshape the generator until the interesting
  regime is common. In stateful tests count the operation mix too —
  Hughes' frequency statistics once revealed an API operation that was
  never generated at all.
- **Watch discards.** A precondition or `t.Skip` that rejects most inputs
  is a vacuous test (see above). Push constraints into the generator.
  Measured on Etna's mutation benchmarks: bespoke correct-by-construction
  generators solved every task while naive generate-and-filter missed 43 —
  and the gap widens as preconditions get sparser.

## Test the generator

If a generator builds values via the code under test (folding `insert`
to build a tree), a bug in the code poisons the generator — and every
property drawing from it fails as a false positive. Write one property
asserting every drawn value satisfies the datatype invariant:

```go
rapid.Check(t, func(rt *rapid.T) {
    v := treeGen.Draw(rt, "v")
    if !valid(v) {
        rt.Fatalf("generator produced invalid value: %v", v)
    }
})
```

If this fails, fix the generator before reading any other failure. rapid
shrinks inside the generator, so this covers shrink validity too — unlike
QuickCheck-style shrink functions, which can shrink into values that
break the invariant (a top shrinking complaint among PBT users).

rapid shrinks the way Hypothesis does: it reduces the underlying sequence
of random choices (shorter first, then earlier draws) and re-runs the
generator, so every shrunk value is one the generator can produce —
validity survives shrinking for free, and there is no shrink function to
write. Help it by structuring custom generators with few, independent
draws: a drawn length followed by that many element draws forces the
shrinker to shrink two coupled regions in lockstep, while self-delimiting
structures (`SliceOf`, or draw-until-stop-bit) shrink cleanly.

Two structural rules decide how well a generator shrinks (de Vries):

- **Independent draws beat dependent ones.** A later draw computed from
  an earlier one can only shrink after the earlier draw is fixed — the
  shrinker never revisits a committed dependency. Keep draws independent
  where possible; accept that genuinely dependent draws shrink
  one-directionally.
- **Filtering stalls shrinking.** A filter that rejects shrunk values is
  a dead end: 22 shrinks only to odd numbers, so a filtered even-number
  generator stops at 22. Generate constrained values by construction
  (draw any number, multiply by two) — filtering hurts shrinking exactly
  the way it hurts generation.

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

### Correlate related inputs

Independent draws are almost never related, and bigger inputs make it
worse: when a failure needs a key that is actually present, two equal
keys, or a value near a boundary, the probability of an accidental hit
shrinks as the structure grows — Etna tasks solvable in milliseconds at
size 3 timed out at size 30. Manufacture the relationship:

```go
// reuse keys that exist instead of always drawing fresh ones
k := rapid.IntRange(0, 100).Draw(rt, "k")
if len(keys) > 0 && rapid.Bool().Draw(rt, "hit") {
    k = rapid.SampledFrom(keys).Draw(rt, "kExisting")
}
```

Keep the bias probabilistic — the miss case must stay reachable. And keep
it cheap: S3's ShardStore team found that mirroring production
distributions bought nothing, and settled on trusting default randomness
unless there is quantitative evidence a bias pays (theirs: keys drawn
from previously-written keys, sizes near page boundaries).

## Swarm testing: vary the configuration per run

A single tuned generator configuration visits the same region of the
test space on every run. Swarm testing instead draws a random
configuration per run — each feature or op-type is coin-tossed off in
some runs. It pays through two mechanisms: features actively suppress
behaviors (pop calls keep a stack shallow — a 50/50 push/pop mix
triggers an overflow bug once per ~370,000 tests, while runs with pushes
only trigger it once per 16), and features passively crowd each other
out of finite tests. A week of swarm testing found 42% more distinct
C-compiler crashes than the hand-tuned all-features configuration.

```go
// coin-toss features at the top of the property; each run gets its own mix
allowDelete := rapid.Bool().Draw(rt, "allowDelete")
allowResize := rapid.Bool().Draw(rt, "allowResize")
ops := rapid.SliceOfN(opGen(allowDelete, allowResize), 1, 100).Draw(rt, "ops")
```

Keep the all-features configuration in the rotation — some bugs need
many features at once (100 random configs cover any given 5-feature
combination with 95% probability). If one feature dominates the system's
complexity (file-system rename), include it with probability above 50%.

## Workflow

- **Budget like a unit test.** Property tests live in the normal suite;
  practitioners cap them at 50ms–30s. Most killable bugs die young — 55%
  of mutants were caught by the first generated input and 76% within 20 —
  so keep CI runs cheap and raise the run count (`-rapid.checks`) for
  nightly or out-of-band runs instead of slowing the build; big check
  counts buy the tail.
- **Ship generators with types.** Export
  `func WidgetGenerator() *rapid.Generator[Widget]` alongside the type so
  downstream tests compose it instead of re-deriving it badly.
- **Properties are documentation.** Reviewers read the property, not the
  runs — write it for the reader. A concise property beats a page of
  example cases in review.

## Stateful and nondeterministic systems

rapid has no state-machine DSL; the idiom is a drawn operation sequence
replayed against both the system and a trivial model, comparing
observable state at the end:

```go
ops := rapid.SliceOfN(opGen, 1, 100).Draw(rt, "ops")
sys, model := NewSystem(), NewModel()
for _, op := range ops {
    op.Apply(sys)
    op.Apply(model)
}
// compare sys's observable state to model
```

Compare after every operation, not just at the end — a divergence
localized to its operation is far easier to read than a corrupted final
state.

Operations that return objects need a pool (Hypothesis's bundles
pattern). If an op produces a handle that other ops consume — `Create`
returning a queue, `Merge` returning a new heap — later ops must draw
from what earlier ops returned, not from freshly generated objects,
which loses the stateful coupling. Keep a slice of live handles during
replay: creating ops append to it, consuming ops pick from it with
`SampledFrom`. This is how a merge of two merged heaps — the failure no
human would construct — becomes reachable.

Patterns from testing deployed synchronizers (Dropbox, ownCloud),
deterministic simulators (FoundationDB, Antithesis, Dropbox Nucleus),
and model-checked storage (Amazon ShardStore):

- **Hidden background work.** When the system acts on its own (sync
  daemons, retries), its state isn't knowable from observations. Track a
  set of possible model states; a run is valid if *some* interleaving of
  conjectured internal events (uploads/downloads) explains them.
- **Stabilize step.** For eventually consistent systems, add a
  wait-for-quiescence operation and assert agreement only there —
  mid-flight divergence is legitimate. Never stabilizing is a failure.
- **Background work belongs in the alphabet.** Include maintenance and
  lifecycle operations (compact, flush, reclaim, reboot) even when they
  are no-ops in the model — that is what proves they don't corrupt
  observable state.
- **Crash is an operation.** For durable systems, add a reboot op and
  check two properties after it: persistence (whatever was durable
  before the crash is readable after) and forward progress (after a
  clean shutdown, everything is durable).
- **Failure injection relaxes, never inverts.** Add fault ops (drop a
  write, fail a disk once); after an injected failure an operation may
  fail, but may never return wrong data. Run the suite with and without
  injection.
- **Flaky failures count.** A failure that doesn't reproduce is still a
  failure. Repeat each run and fail if any repetition fails (3× while
  searching, 20× while shrinking in the Dropbox study — timing-dependent
  sequences also resist shrinking).
- **Serializability via interleaving search.** For concurrency: run a
  sequential prefix, then two op batches in parallel, then search for
  any interleaving of the parallel results that is consistent with the
  model; if none exists, that is a definite bug. Two parallel branches
  suffice for most concurrency bugs and keep the search feasible (~10k
  interleavings for 16 ops). Validate the model against sequential runs
  first — a model that already disagrees sequentially makes parallel
  results unreadable. Timing-dependent failures defeat shrinking, so
  repeat each candidate N times and count it passing only if all N pass
  (PULSE's `?ALWAYS(10)`); when shrinking, move ops from the parallel
  branches into the sequential prefix wherever possible, so the final
  counterexample is *minimally parallel* — the ops left parallel are the
  ones that truly race. (This found Klarna's database corruption in ~10
  minutes after six weeks of human hunting; the shrunk trigger was 5–6
  calls.)
- **Determinism is a feature you build.** Go randomizes map iteration
  order — iterating a map inside a property makes runs non-reproducible
  and defeats shrinking; sort keys first. Order the op alphabet
  simplest-first so the shrinker prefers cheap ops when minimizing.
- **Deterministic simulation.** For deep testing: single-thread the core,
  inject seeded randomness and fake clock/network/disk, and every failure
  replays from a seed forever. Run continuously — millions of scenarios a
  day, not once before release. The unit-test-scale version: a user-level
  scheduler that takes every scheduling decision from a seed (PULSE) —
  fine-grained interleavings and perfect replay without rewriting the
  system.
- **Tight data model.** Strong invariants make validity checks cheap; a
  permissive model leaves "undesirable but legal" outcomes you can't
  assert against.

## Interpreting failures

rapid shrinks to a minimal counterexample and persists it under
`testdata/rapid/` — commit that directory, it replays in CI. Then decide:

- **Wrong property** if the counterexample is legitimate input the property
  never covered. Fix the property or the generator, not the code.
- **Wrong model** if a precondition fails during replay, or the model
  disagrees with clearly reasonable behavior. Calling something a
  specification does not make it right.
- **Wrong code** if the counterexample is minimal and clearly valid.
  Paste it into a unit test only to lock a real regression; otherwise the
  property test already covers it.

After the first fix, keep running: one property routinely finds several
distinct bugs (Hughes' circular buffer yielded three in a row — most
developers stop at the first green).

Read the shrunk case precisely. What it *doesn't* contain is information
(no two equal values ⇒ distinctness is required to trigger). Shrinking is
a local minimum with a floor set by the smallest value the generator can
produce, so residual boilerplate means nothing — and the same bug
shrinking to the same canonical case every time is normalization, a
feature.

## References

- Hughes, [How to Specify It!](https://research.chalmers.se/publication/517894/file/517894_Fulltext.pdf) — five property kinds, with measured bug-finding power
- Hughes, [Experiences with QuickCheck: Testing the Hard Stuff and Staying Sane](https://www.cs.tufts.edu/~nr/cs257/archive/john-hughes/quviq-testing.pdf) — the state-machine recipe; serializability via interleaving search
- Groce et al., [Swarm Testing](https://users.cs.utah.edu/~regehr/papers/swarm12.pdf) — per-run feature omission beats the tuned configuration
- MacIver & Donaldson, [Test-Case Reduction via Test-Case Generation](https://www.doc.ic.ac.uk/~afd/papers/2020/ECOOP_Hypothesis.pdf) — how choice-sequence shrinking works
- de Vries, [Integrated versus Manual Shrinking](https://www.well-typed.com/blog/2019/05/integrated-shrinking/) — why dependent draws and filters defeat shrinking
- MacIver, [Rule-Based Stateful Testing](https://hypothesis.works/articles/rule-based-stateful-testing/) — rules, preconditions, and the bundles pool pattern
- Claessen et al., [Finding Race Conditions in Erlang with QuickCheck and PULSE](https://publications.lib.chalmers.se/records/fulltext/125252/local_125252.pdf) — linearizability checking, `?ALWAYS`, seeded scheduling
- Le et al., [Compiler Validation via Equivalence Modulo Inputs](https://web.cs.ucdavis.edu/~su/publications/emi.pdf) — dead-zone mutations as metamorphic relations
- Shi et al., [Etna](https://lemonidas.github.io/pdf/ETNA.pdf) — mutation-based comparison of generation strategies; correlated inputs beat bigger inputs
- Goldstein et al., Property-Based Testing in Practice (ICSE 2024) — 30 Jane Street interviews
- Goldstein et al., [Tyche](https://dl.acm.org/doi/10.1145/3654777.3676407) — observability for generated-input distributions
- Corgozinho et al., How Developers Implement Property-Based Tests (ICSME 2023) — 86 Hypothesis tests in the wild
- Ravi & Coblenz, [An Empirical Evaluation of Property-Based Testing in Python](https://cseweb.ucsd.edu/~mcoblenz/assets/pdf/OOPSLA_2025_PBT.pdf) — ~50× mutants per test vs. unit tests; 76% within 20 inputs
- Hughes et al., [Mysteries of Dropbox](https://www.cis.upenn.edu/~bcpierce/papers/mysteriesofdropbox.pdf) — testing nondeterministic systems
- Bornholt et al., [Using Lightweight Formal Methods to Validate a Key-Value Storage Node in Amazon S3](https://assets.amazon.science/07/6c/81bfc2c243249a8b8b65cc2135e4/using-lightweight-formal-methods-to-validate-a-key-value-storage-node-in-amazon-s3.pdf) — industrial model-based playbook: crash ops, failure injection, argument bias
- [Dropbox: Rewriting the heart of our sync engine](https://dropbox.tech/infrastructure/rewriting-the-heart-of-our-sync-engine), [Antithesis: Is something bugging you?](https://antithesis.com/blog/is_something_bugging_you/) — deterministic simulation testing
- Wlaschin, [Choosing properties for property-based testing](https://fsharpforfunandprofit.com/posts/property-based-testing-2/)
- Chen et al., METRIC (MR identification via category-choice)
- Segura et al., [A Survey on Metamorphic Testing](https://eprints.whiterose.ac.uk/id/eprint/110335/1/segura16-tse.pdf)
- [rapid docs](https://pgregory.net/rapid)
