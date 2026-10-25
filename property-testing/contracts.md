# Contracts, equality, and side effects

Before choosing a property, identify the valid input domain, observable
contract, equivalence relation, and permitted side effects. Separate API
guarantees from implementation accidents. Invalid inputs belong in separate
properties when their rejection or other behavior is specified.

## Decide what equal means

- **Roundtrips.** Canonicalizing encoders may preserve meaning without
  preserving representation. Compare decoded values using semantic equality,
  or with a specified canonical form; do not require byte identity unless
  the format promises it.
- **Floating point.** Decide how to handle NaN (not equal to itself under
  `==`), signed zero (equal under `==` but distinguishable), rounding, and
  overflow/infinities where relevant. Derive tolerances from the numerical
  contract, not from whichever tolerance makes the test pass.
- **Collections.** Specify whether nil and empty are equivalent, whether
  ordering matters, and whether duplicates count (sequence, multiset, or set).
  Normalize only distinctions the contract says are irrelevant. If sorted,
  duplicate-free output is promised, converting it to a set before comparing
  would hide ordering and duplicate bugs.

## Check permitted side effects

An error does not imply rollback or no partial state. Assert unchanged state
on failure only if the API promises atomic failure; otherwise check the
specified partial effects and any invariants that must survive rejection.

Correct return values do not prove that inputs stayed unchanged. When the
contract requires preservation, snapshot inputs before the call and compare
afterward. Copy nested mutable data too: a shallow slice or map copy can let
the implementation mutate the expected value along with the actual one.

## Choose independent, complementary checks

Prefer an independent reference implementation or a simple model. Repeating
the implementation's algorithm in the model can repeat its defects. Model
only the observable guarantees, not incidental internal representation.

Combine agreement with structural invariants, specified error behavior, and
input preservation. A roundtrip checks consistency between two operations,
not conformance to an independent format: `encode(x) = x + 1` and
`decode(x) = x - 1` roundtrip even if both violate the wire specification.
Use independent format examples or a reference codec for that guarantee.
