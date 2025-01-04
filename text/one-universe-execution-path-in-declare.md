- Title: A unique execution path for the treatement of universes in `declare.ml`

- Drivers: Hugo Herbelin

----

# Summary

We propose a restructuration of `declare.ml` that ensures that all execution paths apply the same treatment and do it without redundancies. For the most part this means delaying the treatment of universes in `close_proof` and splitting `declare_definition` into a first part checking evars and a second part processing universes that becomes factorizable with other execution paths.

# Motivation

Some of the objectives are:
- to have a clearer code, with uniform naming, and structure as easy to understand as possible for new readers
- that future changes in the treatment of universes get applied to all execution paths uniformly at the same time
- to remove some redundancies in the code

# Overview of the main declaration tasks, especially regarding universes

After completion of elaboration (and a first universe minimization) we have three ways to go from `econstr` to a new constant in the environment depending on how we deal with evars:
- A1. Non-interactive mode: `Definition`, `Parameter`, non-interactive `Fixpoint` ...

  It errors on unresolved evars and if no error, it (trivially) embeds the body as a `(constr * UState.t) Future.computation` and continue.

- A2. Goal mode: `Theorem`, body-free `Definition`, body-free `Fixpoint`, ...

  It open subgoals for evars, and when fully resolved, it continues with the proof seen as a `(constr * UState.t) Future.computation`.

- A3. Obligation mode: `Program Definition`, `Program Fixpoint`, ...

  It open obligations for unresolved evars, and when all resolved, it continues.

Here, "to continue" means going from `(constr * UState.t) Future.computation` to either a `constant_entry` (case of `Definition` in A1, of `Qed` in A2, of final `Admitted` or `Qed` in A3) or a `parameter_entry` (case of `Parameter` in A1, of `Admitted` in A2, of an initial `Program Parameter` in A3). In the first case, it means (roughly in sequence):
- B1. minimizing body universes;
- B2. restricting universes;
- B3. checking universe declaration;
- B4. computing universe entry;
- B5. computing the "using" clause;
- B6. registering the resulting entry to safe_typing, doing auxiliary registrations (impargs, etc.);
- B7. registering "where"-style notations.

So, morally, it seems we should have only three functions for dealing with evars (possibly multiplied by 2 if we want to treat recursive definition differently from non-recursive ones, and possibly adding 2 functions for `Parameter` and `Program Parameter`) + two functions to take the relay once the original declaration is either fully proved or admitted.

But currently, we have different ways to do so:

- there are two variants of B3 and B4, one used by A1 (say B3'-B4') and one used by A2 (say B3''-B4''), with A3 calling both
- in the non-interactive mode A1, we do B1-B4 as part of A1 and resume at the level of doing B5, B6, B7;
- in the goal mode A2, we resume with B1, B2''-B4'' then sharing only B6;
- in the obligation mode A3, we do B1, B2''-B4'' then:
  - for obligations, we redo B2'-B4' then jump in the middle of B6
  - for the main declaration: we resume towards the beginning of A1, also redoing B2'-B4'
- the treatment for non-recursive and recursive declarations is shared for the last part of A2 and A3 and rather different for A1

# The proposal

The proposal is to move from the computation flow described in the next subsection to the computation flow described in the subsection following the next subsection.

## Current computation flow

#### Non recursive case

- A1 (declare_definition / declare_parameter)
  - prepare_definition (if a definition)
    - evars check
    - finalization
      - minimization (B1)
      - restriction (B2')
    - universe declaration + universe entry (B3'+B4')
    - using (B5)
  - declare_entry / declare_parameter (B6)
  - notations (B7)
- A2 (start_definition_with_initialization)
  - using (B5)
  - open a proof
  - wait for the proof being complete
- A3 (Obls.add_definition)
  - open n obligations + main definition
  - wait for the obligations to be proved

#### Mutually recursive case

- A1 mutual (declare_mutual_definition)
  - (note: minimization and evar checks are supposed to be done beforehand)
  - restriction (B2')
  - universe declaration + universe entry (B3'+B4')
  - using (B5)
  - declare_entry (B6)
  - notations (B7)
- A2 mutual (start_mutual_definition_with_initialization)
  - using (B5)
  - open a multi-goal proof
  - wait for the proof being complete
- A3 mutual (Obls.add_mutual_definition)
  - open n obligations + main definition
  - wait for the obligations to be proved

#### Resumption from the proving process (common to both non-recursive and recursive cases)

- Qed
  - prepare_proof
  - close_proof
    - minimization (B1)
    - restriction (B2'')
    - universe declaration + universe entry (B3''+B4'')
  - dispatch on auxiliary obligations / main proof (finalize_proof)
    - if proof complete (declare_possibly_mutual_definitions)
      - B6 (declare_parameter)
    - if an obligation (obligation_terminator, then declare_obligation)
      - redo B2, B3' (in current_obligation_uctx)
      - redo B4'
      - B6 (skipping declare_entry, going directly to declare_constant)
      - if was last obligation: go to A1 recursive for main declaration (declare_mutual_definition)
- Admitted
  - universe declaration + universe entry (B3+B4)
  - dispatch on auxiliary obligation / main proof (finish_admitted)
    - if was a proof (declare_possibly_mutual_parameters)
      - B6 (declare_parameter)
    - if an obligation (obligation_admitted_terminator)
      - B2, redo B3' (in current_obligation_uctx)
      - redo B4'
      - B6 (declare_parameter via declare_possibly_mutual_parameters)
      - if was last obligation: go to A1 recursive for main declaration (declare_mutual_definition)

## Proposed computation flow

The plan is that, for each of A1-A3, after obtention of a `(constr * UState.t) Future.computation` (or `(constr list * UState.t) Future.computation` in the mutual case), one continues with a common B1, B2-B4 (relying on the more general B2''-B4'' form), B5, B6, B7.

#### Initial part

- A1 (declare_definition / declare_parameter)
  - prepare_definition (if a definition)
    - evar check
    - using (B5)
  - preparing the body as a future computation or preparing the non body
- A2 (with new name start_definition)
  - prepare proof
  - using (B5)
  - wait for the proof being complete
- A3 (Obs.add_definition)
  - try to solve what is possible
  - using (B5)
  - wait for all obligations proved

#### Asynchronous part

- Qed
  - prepare_proof
  - dispatch on auxiliary obligations / main proof (renamed finish_proof)
    - if proof complete: go to common B
    - if an obligation
      - go to common B for the obligation
      - if was last obligation: go to common B for the main declaration
- Admitted
  - dispatch on auxiliary obligation / main proof (finish_admitted)
    - if an obligation
      - go to common B for the obligation
      - if was last obligation: go to common B for the main declaration
    - if was a proof: go to common B

- Common B (with new name process_declaration, recursive or not, with body or not)
  - minimization (B1)
  - restriction (B2'')
  - universe declaration + universe entry (B3''+B4'')
  - declare_entry / declare_parameter (B6)
  - notations (B7)

## Other behaviors to consider

There are other behaviors in the wild:

- funind calls `build_by_tactic` which produces a constr and do B1, B2''-B4'' and B5, then calls declare_entry to do B6; so, we would need to remove B1 and B2''-B4'' from `build_by_tactic` and call `process_declaration` instead of `declare_entry` (or even propose a variant of `declare_definition` that computes the body with tactics???)

## Remarks

About minimization: one needs a first minimization of the type for opaque constants. Should it be generally done by `declare.ml` (as it is the case for `Definition`) or beforehand (as it is done in `Fixpoint`)?

About evar check: `declare_definition` does its own evar check (which has its own error message) but some callers (e.g. classes) do also their own evar check call (using pretyping which has its own error message too). What would be the most convenient policy (in particular regarding plugins)?

There is no "using" (B5) for obligations and actually no syntax for it (maybe `Next Obligation` could accept a "using" attribute). No "where"-like notation (B7) either.

Would a minimization be useful on `Admitted` obligations? Would it give more than the minimization made before declaring the type of obligations?

Is there a reason why proofs of obligations are not delayed? Or even why transparent proofs are not delayed until we really need them?

## Open questions

- Sometimes there is also an `inline_private_constants` (for obligations and build_by_tactic). I don't know why it is done only sometimes.
- In the `Theorem` execution path, there is a `fix_undefined_variables`. I don't know its role.
