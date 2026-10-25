# Stateful, concurrent, and eventually consistent systems

## The idiom: `Repeat`

`(*rapid.T).Repeat(map[string]func(*rapid.T))` runs a random action sequence
against the system and a trivial model.

- `actions[""]` is the invariant check, run before and after every action.
  Put only checks in it.
- `t.Skip` inside an action is a precondition; rapid retries up to 100 times
  to find a non-skipped action.
- `rapid.StateMachineActions(sm)` builds the map from the methods of a type
  with a `Check(*rapid.T)` method (each other method `Name(*rapid.T)` is an
  action).
- `-rapid.steps` sets the average actions per run; halved under `-short`.
- Action order for shrinking: names are sorted and picked via `SampledFrom`,
  which shrinks toward index 0. The **alphabetically first** name is the one
  shrinking prefers — name cheap ops first (`a_add`, `b_has`, `c_union`).

Operations that return handles need a pool (Hypothesis's bundles): creating
ops append to a slice of live handles, consuming ops `SampledFrom` it. That
makes a merge of two merged heaps — a failure no human would construct —
reachable.

## Complete example

A sorted-slice set as the system, `map[int]bool` as the model, a pool of
live sets, an invariant in `""`. Compiles and passes against rapid v1.3.0;
planting a dedup bug in `Add` fails it with a two-action counterexample.

```go
package ex

import (
	"slices"
	"sort"
	"testing"

	"pgregory.net/rapid"
)

// IntSet is the system under test: a sorted, duplicate-free slice.
type IntSet struct{ xs []int }

func (s *IntSet) Add(x int) {
	i := sort.SearchInts(s.xs, x)
	if i < len(s.xs) && s.xs[i] == x {
		return
	}
	s.xs = slices.Insert(s.xs, i, x)
}

func (s *IntSet) Has(x int) bool {
	i := sort.SearchInts(s.xs, x)
	return i < len(s.xs) && s.xs[i] == x
}

func Union(a, b *IntSet) *IntSet {
	u := &IntSet{}
	for _, x := range a.xs {
		u.Add(x)
	}
	for _, x := range b.xs {
		u.Add(x)
	}
	return u
}

func TestIntSetModel(t *testing.T) {
	rapid.Check(t, func(rt *rapid.T) {
		type pair struct {
			sys   *IntSet
			model map[int]bool
		}
		pool := []pair{{&IntSet{}, map[int]bool{}}} // live handles
		pick := func(label string) pair {
			return rapid.SampledFrom(pool).Draw(rt, label)
		}
		// small range so Add/Has collide with existing members
		elem := rapid.IntRange(-8, 8)

		rt.Repeat(map[string]func(*rapid.T){
			"a_add": func(rt *rapid.T) {
				p := pick("set")
				x := elem.Draw(rt, "x")
				p.sys.Add(x)
				p.model[x] = true
			},
			"b_has": func(rt *rapid.T) {
				p := pick("set")
				x := elem.Draw(rt, "x")
				if got := p.sys.Has(x); got != p.model[x] {
					rt.Fatalf("Has(%d)=%v, model says %v", x, got, p.model[x])
				}
			},
			"c_union": func(rt *rapid.T) {
				a, b := pick("a"), pick("b")
				m := map[int]bool{}
				for k := range a.model {
					m[k] = true
				}
				for k := range b.model {
					m[k] = true
				}
				pool = append(pool, pair{Union(a.sys, b.sys), m})
			},
			"": func(rt *rapid.T) { // invariant, before and after every action
				for _, p := range pool {
					if !sort.IntsAreSorted(p.sys.xs) || len(p.sys.xs) != len(p.model) {
						rt.Fatalf("invariant broken: %v vs model %v", p.sys.xs, p.model)
					}
				}
			},
		})
	})
}
```

## When `Repeat` is not enough

`Repeat` cannot express parallel batches. For serializability checking, draw
an op slice (`prefix`, `branchA`, `branchB`) and replay by hand (below).

## Patterns for harder systems

From testing deployed synchronizers (Dropbox, ownCloud), deterministic
simulators (FoundationDB, Antithesis), and model-checked storage (S3
ShardStore):

- **Hidden background work.** When the system acts on its own (sync daemons,
  retries), track a *set* of possible model states; a run is valid if some
  interleaving of conjectured internal events explains the observations.
- **Stabilize step.** For eventually consistent systems add a
  wait-for-quiescence action and assert agreement only there; mid-flight
  divergence is legitimate. Never stabilizing is a failure.
- **Background work belongs in the action set.** Include compact, flush,
  reclaim, reboot even when they are no-ops in the model — that proves they
  don't corrupt observable state.
- **Crash is an action.** For durable systems add a reboot action and check
  persistence (whatever was durable before the crash is readable after) and
  forward progress (after clean shutdown, everything is durable).
- **Failure injection relaxes, never inverts.** Add fault actions (drop a
  write, fail a disk once); afterward an operation may fail but may never
  return wrong data. Run the suite with and without injection.
- **Flaky failures count.** Repeat each candidate (3× while searching, 20×
  while shrinking) and fail if any repetition fails.
- **Serializability via interleaving search.** Run a sequential prefix, then
  two op batches in parallel, then search for any interleaving of the
  parallel results consistent with the model; none ⇒ definite bug. Two
  branches suffice for most bugs and keep the search feasible (~10k
  interleavings for 16 ops). Validate the model sequentially first. When
  shrinking, move ops from the parallel branches into the prefix wherever
  possible, so the counterexample is *minimally parallel*: what stays
  parallel truly races.
- **Determinism is a feature you build.** Go randomizes map iteration;
  iterating a map inside a property makes runs non-reproducible and defeats
  shrinking. Sort keys first.
- **Deterministic simulation.** Single-thread the core, inject seeded
  randomness and fake clock/network/disk so every failure replays from a
  seed. Unit-test-scale version: a user-level scheduler whose decisions come
  from a seed.
- **Tight data model.** Strong invariants make validity checks cheap; a
  permissive model leaves "undesirable but legal" outcomes you can't assert
  against.
- **Count the action mix.** Tally which actions actually run; a never-chosen
  or always-skipped action is untested.
