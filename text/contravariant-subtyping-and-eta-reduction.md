- Title: Contravariant subtyping and support for η-reduction

- Drivers: Hugo Herbelin

----

# Summary

We propose to turn subtyping contravariant on the domain of dependent products and to include η-reduction as a standard reduction rule.

# History

Subtyping was introduced in Coq version 4.8 (1988) where it was contravariant on the domain side of dependent products. In version 4.9 (1989) it was however moved to equivariance on the domain side of dependent products.

The story I recalled is that the set-theoretic model of the Extended Calculus of Constructions which Luo studied in his PhD (defended 1990) was not able to model the contravariance. More precisely, granting a contravariant interpretation of `Πa:A.B` in its domain is easy, but, in a setting where the conversion is untyped, there is no way to give the same interpretation to `f` and to an η-expansion `λx:A.fx` restricting its domain.

Since then, Martin-Löf's typed approach to conversion learnt us that we can interpret typed conversion instead, and, by results from Adams, Siles and the author [[1](https://www.cs.ru.nl/R.Adams/ptseq8.pdf),[4](http://pauillac.inria.fr/~herbelin/articles/jfp-SilHer11-pts-typed-conv-all.pdf)] who proved the equivalence of typed and untyped conversion in the setting of Pure Type Systems, there is a strong presumption that untyped conversion can be equipped with types also in the presence of subtyping.

Additionally, η-expansions can precisely be used as contravariant/covariant coercions to emulate contravariant/covariant subtyping on top of a subtyping rule which is only equivariant on both sides of the dependent product. It has also be shown in the context of the Calculus of Constructions that systematic η-expansion is possible (see e.g. Dowek and Werner [[3](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.48.273&rep=rep1&type=pdf)]). Thus, contravariant/covariant subtyping domain can conversely be rethought as an implicit way to reason in the language of η-expanded terms.

As a consequence, it is unclear to me that the absence of contravariance of the domain is still defendable, even, if, stricto sensu, the exact formulation of the metatheory of Coq is only in progress (see MetaCoq) and, in particular, we have neither a formal proof that Coq subtyping between well-formed types is equivalent to a typed one nor a precise formulation that η-expanded judgements using only an equivariant subtyping could be systematically reconstructed from a judgement using contravariant/covariant subtyping.

# Motivations

On the theoretical side, contravariant subtyping of the domain and η-reduction are intuitive and convenient. Why do without?

Also, transparently supporting η-reduction allows to manipulate shorter expressions. Users have already suggested we may have it (e.g. [#2860](http://github.com/coq/coq/issues/2860)).

Additionally, in unification, in order to factorize the code between first-order unification and Miller-Pfenning pattern unification, it is convenient to resolve a Miller-Pfenning pattern `x ⊢ ?f x == t` into `?f := λx.t` even though that in the case `t` had been some `g x` with `x`∉FV(`g`), first-order unification would have instead produced the simpler form `?f := g`. So, if we had a way to be certain that η-reducing unification solutions afterwards is safe, we could stop having a distinct unification path for Miller-Pfenning pattern unification and first-order unification.

# Detailed design

### A different way to compare the domains of λs in the conversion

In the presence of contravariant subtyping, requiring the domains of λs to be convertible for two functions to be convertible is too strong. Indeed, consider e.g. the following valid example where two occurrences of a function only need to coincide on the domain of observation (which here is the one given by the type of `E`):
```
Axiom E : (Set -> Set) -> Set.
Axiom C : forall X, E X.
Check C (fun X : Type => True) : E (fun X : Set => True).
```
Converting the domains of λs is redundant from a theoretical point of view as shown by Barras and Grégoire [[2](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.61.2666&rep=rep1&type=pdf)]. However, as shown by Gilbert, comparing the domains of λs is useful for [efficiency](https://github.com/coq/coq/wiki/Performance-experiments#skipping-conversion-of-parameters-of-constructors-and-of-the-domain-of-lambdas-failed). So, a solution would be to compare the domain up to the identification of all sorts.

Comparing the domain is also useful in unification since it provides unification information. But this would require a third form of unification problems in addition to `CONV` and `CUMUL`, say `BICUMUL` (??).

As for the convertibility algorithms used in `vconv.ml` and `nativeconv.ml`, they already skip the domains of λs, so nothing needs to be changed.

### Implementing contravariant subtyping in the domain

Contravariant subtyping itself is obtained by changing the subtyping rules in a couple of places:

- in `reduction.ml`, where the basic kernel subtyping is implemented
- in `vconv.ml`, where the vm-based subtyping is implemented
- in `nativeconv.ml`, where the native-compute-based subtyping is implemented
- in `evarconv.ml`, where the unification subtyping is implemented
- am I missing something?

### Supporting η-reduction

Some support for η-reduction already exists (see flag `ETA` in `cClosure.ml` and `shrink_eta` in `reductionops.ml`). Providing support for `eta` in `cbv`, `cbn`, `lazy` can be obtained by working at the level of the following files:

- adding support for an `eta` flag in parsing (`g_tactic.mlg` but ideally in `g_vernac.mlg` so that it is part of the specification language, see [#11679](http://github.com/coq/coq/issues/11679)) and printing.
- in `cClosure.ml`: adding a function to `whd_eta`-normalize `constr` and call it from `norm_head` at the time of reifying
- similarly for `cbv_norm_value` in `cbv.ml`
- for `cbn`, the code is already there and it should work automatically from the moment the `eta` flag is parsed
- similarly for internal reduction strategies in `reductionops.ml`

An open question is whether we should optimize the computation that `x` does not occur in `t` in the rule `λx:A.tx` → `t`, so as to avoid traversing the whole term at each new λ. For instance, we could collect the number of occurrences of each variables in the normal form at reification time so that there is no more traversals of the term than the one done by the reification code.

### Usefulness in unification

In unification, we may experiment applying `whd_eta` at the end of `solve_simple_eqn` (which itself is used to resolve Miller-Pfenning problems) so that it produces similar solutions as if first-order unification had been used.

# Further directions

- support for η in `rewrite` ([#11719](http://github.com/coq/coq/issues/11719))
- MetaCoq-justified subtyping
- ...

# Formal bibliography

[1] Robin Adams, _Pure type systems with judgemental equality_, 2006.

[2] Bruno Barras, Benjamin Grégoire, _On the Role of Type Decorations in the Calculus of Inductive Constructions_, 2005.

[3] Gilles Dowek, Benjamin Werner, _On the Definition of the Eta-long Normal Form in Type Systems of the Cube_, 1993.

[4] Vincent Siles, Hugo Herbelin, _Pure Type System conversion is always typable_, 2012.
