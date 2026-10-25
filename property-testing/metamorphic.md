# Metamorphic relations

When no oracle exists, transform the input and predict a relation on the
outputs. Walk each family against each public operation; the non-vacuous
ones are tests.

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
  beat whole-system.
- n operations × n input modifiers = O(n²) candidates. Walk the
  cross-product per operation, not just the families.

**Dead-zone recipe.** Find the part of the input that cannot affect the
result for this operation — fields a parser ignores, elements beyond a
limit, key order in a map — mutate only it, and demand a bit-identical
output. This is differential testing with one implementation: the oracle is
the unchanged output.

Both sides must have content: a subset relation is vacuous if generated
inputs rarely make both sides non-empty. Shape the generator, don't weaken
the relation.
