# References and measurements

Evidence behind the advice in `SKILL.md`. None of it changes what to do;
it is here for judging how hard to push.

## Measurements

- Roundtrip and oracle/model-based dominate real usage (~71% of an 86-test
  Hypothesis corpus; the most-used pattern in Jane Street interviews).
- Model-based properties found every bug in 8 buggy BST variants and failed
  fastest (~8 vs. ~50 tests for an equivalent postcondition).
- Across 40 projects under mutation testing, each property test killed ~50×
  the mutants of the average unit test; exception-raising and
  collection-inclusion checks were the most effective (>19×).
- 55% of mutants were caught by the first generated input, 76% within 20 —
  so cheap CI runs find most killable bugs; raise `-rapid.checks` for the tail.
- Over a third of experienced PBT users at Jane Street admitted not
  interrogating passing properties.
- Hughes measured 80% of runs hitting the trivial branch (key absent from the
  tree); frequency statistics once revealed an API operation never generated.
- Etna: bespoke correct-by-construction generators solved every mutation task
  while generate-and-filter missed 43; tasks solvable in milliseconds at size
  3 timed out at size 30 without correlated inputs.
- Swarm testing found 42% more distinct C-compiler crashes than the
  hand-tuned all-features configuration; a 50/50 push/pop mix triggers a
  stack overflow once per ~370,000 tests vs. once per 16 with pushes only;
  100 random configs cover any 5-feature combination with 95% probability.
- EMI dead-zone mutation found 147 confirmed GCC/LLVM bugs.
- Klarna's database corruption was found in ~10 minutes (shrunk to 5–6
  calls) after six weeks of human hunting.
- Dropbox study: repeat 3× while searching, 20× while shrinking.

## Sources

- Hughes, [How to Specify It!](https://research.chalmers.se/publication/517894/file/517894_Fulltext.pdf) — five property kinds, with measured bug-finding power
- Hughes, [Experiences with QuickCheck: Testing the Hard Stuff and Staying Sane](https://www.cs.tufts.edu/~nr/cs257/archive/john-hughes/quviq-testing.pdf) — the state-machine recipe; serializability via interleaving search
- Groce et al., [Swarm Testing](https://users.cs.utah.edu/~regehr/papers/swarm12.pdf) — per-run feature omission beats the tuned configuration
- MacIver & Donaldson, [Test-Case Reduction via Test-Case Generation](https://www.doc.ic.ac.uk/~afd/papers/2020/ECOOP_Hypothesis.pdf) — how choice-sequence shrinking works
- de Vries, [Integrated versus Manual Shrinking](https://www.well-typed.com/blog/2019/05/integrated-shrinking/) — why dependent draws and filters defeat shrinking
- MacIver, [Rule-Based Stateful Testing](https://hypothesis.works/articles/rule-based-stateful-testing/) — rules, preconditions, and the bundles pool pattern
- Claessen et al., [Finding Race Conditions in Erlang with QuickCheck and PULSE](https://publications.lib.chalmers.se/records/fulltext/125252/local_125252.pdf) — linearizability checking, `?ALWAYS`, seeded scheduling
- Le et al., [Compiler Validation via Equivalence Modulo Inputs](https://web.cs.ucdavis.edu/~su/publications/emi.pdf) — dead-zone mutations as metamorphic relations
- Shi et al., [Etna](https://lemonidas.github.io/pdf/ETNA.pdf) — mutation-based comparison of generation strategies
- Goldstein et al., Property-Based Testing in Practice (ICSE 2024) — 30 Jane Street interviews
- Goldstein et al., [Tyche](https://dl.acm.org/doi/10.1145/3654777.3676407) — observability for generated-input distributions
- Corgozinho et al., How Developers Implement Property-Based Tests (ICSME 2023) — 86 Hypothesis tests in the wild
- Ravi & Coblenz, [An Empirical Evaluation of Property-Based Testing in Python](https://cseweb.ucsd.edu/~mcoblenz/assets/pdf/OOPSLA_2025_PBT.pdf)
- Hughes et al., [Mysteries of Dropbox](https://www.cis.upenn.edu/~bcpierce/papers/mysteriesofdropbox.pdf) — testing nondeterministic systems
- Bornholt et al., [Using Lightweight Formal Methods to Validate a Key-Value Storage Node in Amazon S3](https://assets.amazon.science/07/6c/81bfc2c243249a8b8b65cc2135e4/using-lightweight-formal-methods-to-validate-a-key-value-storage-node-in-amazon-s3.pdf) — crash ops, failure injection, argument bias
- [Dropbox: Rewriting the heart of our sync engine](https://dropbox.tech/infrastructure/rewriting-the-heart-of-our-sync-engine), [Antithesis: Is something bugging you?](https://antithesis.com/blog/is_something_bugging_you/) — deterministic simulation testing
- Wlaschin, [Choosing properties for property-based testing](https://fsharpforfunandprofit.com/posts/property-based-testing-2/)
- Chen et al., METRIC (MR identification via category-choice)
- Segura et al., [A Survey on Metamorphic Testing](https://eprints.whiterose.ac.uk/id/eprint/110335/1/segura16-tse.pdf)
- [rapid docs](https://pgregory.net/rapid)
