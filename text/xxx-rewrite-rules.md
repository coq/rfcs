- Title: User-defined rewrite rules
- Drivers: Gaëtan Gilbert, Nicolas Tabareau, Théo Winterhalter

----

# Summary

We want to add user-defined rewrite rules to Coq. They would complement the
axiom mechanism of Coq by allowing users to define their own computation rules.

# Motivation

At the moment users can extend the theory of Coq by assuming extra constructions
using `Axiom` but they cannot assume any computational behaviour.
With rewrite rules, users could also postulate new reduction rules.
This could have various use cases:
- Adding new reduction rules to neutral terms (eg. allowing both `0 + n` and
`n + 0` to reduce to `n`).
- Postulating otherwise rejected inductive types with computation rules for
their eliminator (eg. quotient types or higher inductive types).
- Defining functions using a mechanism more powerful that the usual
pattern-matching.
- Define extensions of CIC such as Exceptional Type Theory or Setoid Type Theory

# Detailed design

## Overview

To ensure confluence and preservation of typing (subject reduction) we enforce
constraints on rewrite rules. As such they have to be defined by blocks
consisting of fresh symbols (axioms) and rewrite rules involving them.
A tentative syntax is as follows:

```coq
Rewrite Block :=
  s₁@{u₁} : T₁
        ⋮
  sₙ@{uₙ} : Tₙ
with rules
  lhs₁ := rhs₁
        ⋮
  lhsₙ := rhsₙ.
```
where terms `sᵢ` are called symbols, quantifying over universe declaration `uᵢ`
and of type `Tᵢ`; the `rhsᵢ` are arbitrary terms and `lhsᵢ` obey the following
syntax:
```
lhs ::= sₖ                                       one of the symbols
      | lhs p                                    lhs applied to a pattern
      | lhs .proj                                projection of a lhs
      | match lhs with p₁ ... pₙ end             pattern-matching of a lhs
```
Note that in particular, a rewrite rule may only rewrite symbols that are
declared together with it.

In the case of pattern-matching, all the branches are patterns as well.
Patterns are given by the following syntax (extending the already existing
notion of patterns):

```
p ::= ?x y₁ ... yₙ         pattern variable (applied to all bound variables)
    | C@{u} p₁ ... pₙ      constructor applied to patterns
    | I@{u} p₁ ... pₙ      Inductive type constructor applied to patterns
    | λ x : p₁, p₂         λ-abstraction over patterns
    | ∀ x : p₁, p₂         Π-type of patterns
    | !{t}                 type-forced term
```

Here we mention pattern variables. They are implicitly quantified over when
defining a rewrite rule. Each variable must appear exactly once on the left-hand
side (`lhs`) and can appear on the right-hand side (`rhs`) any number of times
(including zero).

The reduction of Coq is then extended with rules
```
⟦lhsᵢ⟧[σ] → rhsᵢ[σ]
```
where `σ` instantiates all of the pattern variables (as well as the universe
levels) and where `⟦lhsᵢ⟧` traverses the whole `lhsᵢ` to interpret type-forced
terms as fresh pattern variables (that are therefore unused in the rhs).

Type-forced terms are there to give information to the pre-typer and to ensure
that the rule is indeed type-preserving.

Once a rewrite block is defined, a confluence checker based on parallel
reduction and shown to be sound in MetaCoq will verify that the block
introduction preserves confluence of the whole system, in a modular way
(it doesn't have to recheck previously defined rewrite blocks).

Rewrite rules are triggered when a term is stuck with respect to the regular
rules of Coq.

## Examples

### Parallel plus

One popular example is the definition of a parallel version of addition of
natural numbers:

```coq
Rewrite Block :=
  plus : nat -> nat -> nat ;
with rules
  plus 0 ?n := n ;
  plus ?n 0 := n ;
  plus (S ?n) ?m := S (plus n m) ;
  plus ?n (S ?m) := S (plus n m).
```

### (Non-)linearity in practice

To illustrate the use of type-forced terms (and explain why left-linearity of
pattern variables is not such a restriction) we can show the artificial example
of the `J`-eliminator.

```coq
Rewrite Block :=
  J :
    ∀ A (x : A) (P : ∀ (y : A), x = y → Type),
      P x (eq_refl x) →
      ∀ y (e : x = y), P y e ;
with rules
  J ?A ?x ?P ?p !{x} (@eq_refl !{A} !{x}) := p.
```

### Higher-inductive types

HoTT users might want to use rewrite rules to define HIT rather than use the
current notion of private inductive types.

```coq
Rewrite Block :=
  S¹ : Type ;
  base : S¹ ;
  loop : base = base ;
  elimS¹ : ∀ (P : S¹ → Type) (b : P base) (l : loop # b = b) x, P x ;
  elimS¹ₗₒₒₚ : ∀ (P : S¹ → Type) (b : P base) (l : loop # b = b),
                apD (elimS¹ P b l) loop = loop
with rules
  elimS¹ ?P ?b ?l base := b.
```

### Quotients

We can also postulate quotients with a way to lift functions to the quotient.
This example also illustrates how we could combine this syntax with notations.

```coq
Rewrite Block :=
  Quotient : ∀ A, (A → A → Prop) → Type ;
  proj : A → A // R ;
  quot : ∀ x y, R x y → proj x = proj y ;
  rec : ∀ {A R B} (f : A → B), (∀ x y, R x y → f x = f y) → A // R → B
with rules
  rec ?f ?q (proj ?x) := f x
where
  "A // R" := (Quotient A R).
```

## Possible extensions

Coq [CEP#45]'s integration might make it possible to add rewrite rules to
already defined symbols, such as addition.

[CEP#45]: https://github.com/coq/ceps/pull/45

## About trusted code base

The addition of rewrite rules should not make the user fear about consistency
of the system as a whole.
Similarly to axioms and other consistency-breaking assumptions such as
`Type : Type`, the use of rewrite rules should be reported by the command
```
Print Assumptions
```

Rewrite rules are not so different from axioms in that they might break
consistency and canonicity, except that they introduce new computation rules
and as such might also break termination. Any proof completed without them would
still be entirely safe.

# Drawbacks

This proposal concerns unsafe rewrite rules. As such, uncareful rewrite rules
could lead to infinite loops in the reduction machine or worse to anomalies
(by breaking subject reduction for instance).
More generally, rules can break confluence and result in incompleteness from
the type checker which might reject conversion of two convertible terms in
the extended theory.
These drawbacks can be in part alleviated by implementing a modular confluence
checker as advocated in the paper [The Taming of the Rew] and as is currently
implemented in Agda by Jesper Cockx.

[The Taming of the Rew]: https://hal.archives-ouvertes.fr/hal-02901011

# Alternatives

The Agda implementation of rewrite rules is very much related to this.
There were of course many other attempts to integrate rewrite rules in the past
like Coq Modulo Theory and the work of Bruno Barras.

# Unresolved questions
