- Title: Redesign of `abstract` / "side-effect" mechanism

- Drivers: Gaëtan Gilbert (@SkySkimmer on github)

----

# Summary

We redesign side effects (constants produced by the `abstract` tactic)
to live in the evar map and not in the global environment.

# Motivation

The current design of side effects leads to bugs, for instance not
respecting backtracking or leaving leftover constants that were
supposed to be inlined (https://github.com/coq/coq/issues/13324).

It also makes more sense semantically to have the side effects in the
proof state instead of the global environment.

See also [general issue about abstract](https://github.com/coq/coq/issues/9146).

# Current system

The following are notable behaviours in the current system on which we
need to decide if they are important to keep, not important, or undesired.

1. the cost of checking a side effect is paid when it is created, not at Qed time
2. side effects may be transparent (automatic schemes, transparent_abstract) or opaque (abstract)
3. side effects in a Defined proof become global constants. They are sealed according to their opacity.
4. side effects in a Qed proof or a tactic-in-term (`ltac:(abstract ...)`) get inlined:
  without universe polymorphism the term `C[subproof]` where `C` is a term context
  and `subproof` is the side effect constant becomes
  `let subproof := (proof term) in C[subproof]` for transparent side effects
  and `(fun subproof => C[subproof]) (proof term)` for opaque side effects.
  With universe polymorphism it becomes `C[(proof term)]` (this breaks point 1!)
5. side effects behave as normal constants during the proof. For instance in `Lemma bar : nat * nat`,
  `split;[abstract exact 1 | exact bar_subproof].` works,
  after which `Print bar_subproof.` prints `bar_subproof = 1`,
  `Show Proof` prints `(bar_subproof, bar_subproof)`, etc.
6. side effects for a universe polymorphic surrounding constant must be universe polymorphic.
7. `abstract` garbage collects evars which are created and defined during the proof
   (before 8.20 by running the tactic in a separate evar map which only has its universes merged back to the surrounding proof's evar map,
   after by using `Evd.drop_new_defined`)

# Detailed design

TODO

# Drawbacks

TODO

# Alternatives

TODO

# Unresolved questions

TODO
