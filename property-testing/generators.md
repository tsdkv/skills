# Generators: validity, shrinking, swarm

## Test the generator

If a generator builds values via the code under test (folding `insert` to
build a tree), an invalid result can come from the generator, implementation,
or assumed invariant. Write one property asserting every drawn value
satisfies the datatype invariant:

```go
rapid.Check(t, func(rt *rapid.T) {
    v := treeGen.Draw(rt, "v")
    if !valid(v) {
        rt.Fatalf("generator produced invalid value: %v", v)
    }
})
```

If this fails, determine which contract was violated:

- The generator violated a constructor precondition: fix the generator.
- The constructor violated its contract on valid inputs: fix the
  implementation. Valid inserts producing an invalid tree are a real bug.
- The invariant or assumed contract is wrong: fix the test specification.

Fix the responsible component; do not repair generated values in a way that
hides implementation bugs. Because rapid shrinks inside the generator, this
property also checks validity of the values produced during shrinking.

## How rapid shrinks

rapid reduces the underlying sequence of random choices (shorter first, then
earlier draws) and re-runs the generator, so every shrunk value is one the
generator can produce. There is no shrink function to write. Help it:

- **Few, independent draws.** A drawn length followed by that many element
  draws forces two coupled regions to shrink in lockstep; self-delimiting
  structures (`SliceOf`, draw-until-stop-bit) shrink cleanly.
- **Independent beats dependent.** A later draw computed from an earlier one
  shrinks only after the earlier is fixed; the shrinker never revisits a
  committed dependency. Accept that genuinely dependent draws shrink
  one-directionally.
- **Filtering stalls shrinking.** A filter that rejects shrunk values is a
  dead end (a filtered even-number generator stops at 22). Generate by
  construction: draw any number, multiply by two.
- **Order alternatives simplest-first** in `SampledFrom` / `OneOf` — shrinking
  moves toward index 0.

## Correlated inputs

Independent draws are almost never related, and bigger inputs make it
worse: the chance of drawing a key that is present, two equal keys, or a
value at a boundary falls as the structure grows. Manufacture the
relationship (reuse existing keys, size near a page boundary), keeping the
miss case reachable. Add a bias only with evidence it pays; mirroring
production distributions bought nothing in S3's ShardStore.

## Swarm testing: vary the configuration per run

One tuned configuration visits the same region of the test space every run.
Draw a random configuration per run instead — each feature or op type
coin-tossed off in some runs. Features suppress each other (pops keep a stack
shallow, hiding an overflow) and crowd each other out of finite tests.

```go
allowDelete := rapid.Bool().Draw(rt, "allowDelete")
allowResize := rapid.Bool().Draw(rt, "allowResize")
ops := rapid.SliceOfN(opGen(allowDelete, allowResize), 1, 100).Draw(rt, "ops")
```

Keep the all-features configuration in the rotation — some bugs need many
features at once. If one feature dominates the system's complexity (file
rename), enable it with probability above 50%.
