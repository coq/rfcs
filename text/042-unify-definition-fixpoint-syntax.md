- Title: Unifying the syntax of Theorem, Definition and Fixpoint

- Drivers: Théo Zimmermann

----

# Summary

This CEP proposes to unify the syntax of commands of the `Theorem`
family, of the `Definition` family and the `Fixpoint` and `CoFixpoint`
commands.  All the currently supported syntax forms would be kept.
The different families of commands would become variants whose
semantics differ only on specific and clearly documented points which
could alternatively be set using attributes.  For instance, `Theorem`
would become a strict equivalent of `#[ sealed ] Definition` and
`Fixpoint` a strict equivalent of `#[ recursive ] Definition`.

Because of the new `sealed` and `unsealed` attributes introduced in
this CEP which allow to declare the transparency of an object (in the
sense of `Defined` / `Qed`) at the declaration point, this CEP also
contains a second, optional, part about a plan to slowly move away
from the idea that the `Defined` / `Qed` commands are the ones which
determine transparency.

# Motivation

There are several motivations to unify the syntax of these commands:

- Supporting `Theorem @ident {? @type } := @term`:

  - Currently there is no way to define an opaque constant by
    providing only its body.

  - In the upcoming refactoring of the reference manual, we would like
    the first chapter to be about the core language that the kernel
    understands.  We couldn't give examples using `Theorem` because
    this command always open the proof mode.  Yet, it is important to
    be able to explain that the kernel can define opaque
    constants, i.e. constants with a body which behave like axioms.

  - It would allow deprecating the akward `Proof @term` syntax.

- Fixing the inconsistent semantics of `Theorem` and `Theorem
  ... with`.  The first syntax form behaves like `Definition` but the
  second one behaves like `Fixpoint`.

  A previous attempted solution was simply to deprecate the `Theorem
  ... with` syntax in favor of `Fixpoint ... with` in proof mode
  ([coq/coq#8265](https://github.com/coq/coq/pull/8265)).  However,
  this syntax is used in CompCert and Xavier Leroy argued in favor of
  keeping it, in particular on the basis that people who read and
  review formalizations might appreciate the distinction of intent
  between a `Theorem` and a `Fixpoint`.  Still, there is no reason why
  we should explicitly distinguish between a `Definition` and a
  `Fixpoint` but not between a non-recursive and a recursive
  `Theorem`.

  The proposed solution is to keep the existing syntax `Theorem
  ... with` but encourage that it comes with an explicit `recursive`
  annotation (in the form of an attribute).  This also opens the way
  to supporting (non-mutually) recursive `#[ recursive ] Theorem`, and
  `#[ recursive ] Definition` as an equivalent to `Fixpoint` to define
  (possibly mutually) recursive objects.

The motivation for the second part of the CEP (separating the
transparency status from the `Defined` / `Qed` commands) is that
`Defined` is naturally associated with the notion of definition and
`Qed` with the notion of proof.  The use case that it has taken
(defined theorems to mean non-opaque ones) was not anticipated and now
we would have other ways of specifying it (the attribute).
Furthermore, having the transparency status of an object always be
determined at declaration time would remove an inconsistency between
the case where a body is provided and the case where the body is built
in interactive proof mode.

# Detailed design

The current syntax is:

- for `Definition` / `Example`:
  ```
  {| Definition | Example } @ident_decl @def_body

  def_body ::= {* @binder } {? : @type } := {? @reduce } @term
             | {* @binder } : @type
  ```

- for `Fixpoint`:
  ```
  Fixpoint @fix_def {* with @fix_def }

  fix_def ::= @ident_decl {* @binder } {? @fixannot } {? : @type } {? := @term } {? @decl_notations }
  ```

- for `CoFixpoint`:
  ```
  CoFixpoint @cofix_def {* with @cofix_def }

  cofix_def ::= @ident_decl {* @binder } {? : @type } {? := @term } {? @decl_notations }
  ```

- for `Theorem` and variants:
  ```
  @thm_token @ident_decl {* @binder } : @type {* with @ident_decl {* @binder } : @type }

  thm_token ::= Theorem
              | Lemma
              | Fact
              | Remark
              | Corollary
              | Proposition
              | Property
  ```

The new syntax would be:

```
@decl_token @decl_def {* with @decl_def }

decl_token := Definition
            | Example
            | Fixpoint
            | CoFixpoint
            | Theorem
            | Lemma
            | Fact
            | Remark
            | Corollary
            | Proposition
            | Property

decl_def := @ident_decl {* @binder } {? @fixannot } @decl_body {? @decl_notations }

decl_body := {? : @type } := {? @reduce } @term
           | : @type
```

with the following newly supported attributes:

- `recursive` and `corecursive` to declare a fixpoint / cofixpoint:

  - attribute `recursive` is implicit for `Fixpoint` and rejected by
    `CoFixpoint`;
  - attribute `corecursive` is implicit for `CoFixpoint` and rejected
    by `Fixpoint`;
  - attribute `recursive` or `corecursive` is compulsory to use the
    `with` variant of `Definition`;
  - using the `with` variant of `Theorem` without attribute
    `recursive` or `corecursive` raises a deprecation message and is
    interpreted as if `recursive` had been used;
  - the `recursive` and `corecursive` attributes are incompatible;
  - the `@fixannot` clause requires the `recursive` attribute
    being used (or implicit in the case of `Fixpoint`).

- `sealed` and `unsealed` to declare an object opaque or transparent
  like `Qed` and `Defined` currently do (note that this is not the
  same as what the `Opaque` and `Transparent` commands do, and the
  choice of new words `sealed` / `unsealed` was made to avoid any
  confusion with tactic-level transparency):

  - `unsealed` is assumed by default for `Definition`, `Example`,
    `Fixpoint`, `CoFixpoint` but `sealed` is accepted;
  - `sealed` is assumed by default for `Theorem` and variants but
    `unsealed` is accepted.

The second part of the CEP is about the interaction of the `sealed` /
`unsealed` attributes with the interactive proof mode.  The end goal
is that the transparency status is only determined by the attributes
and that `Qed` and `Defined` become synonyms of each other (`Qed`
being encouraged in combination with `Theorem` and `Defined` being
encouraged in combination with `Definition`).  The steps to reach this
goal are the following:

1. When the new `sealed` / `unsealed` attributes are introduced,
   deprecate the use of `Defined` to close a theorem without an
   explicit `unsealed` attribute (similarly deprecate the use of `Qed`
   to close a definition without an explicit `sealed` attribute).  In
   addition to the attributes, flags are provided to serve the needs
   of users who are frequently declaring transparent theorems.  At
   this point, transparency in proof mode is still determined by the
   closing command, unless an attribute is explicitly provided or a
   flag has been set.

2. Turn the deprecation into a hard error.  This could happen several
   releases after the deprecation was introduced to give plenty of
   time to users to be aware and change their code.  This step is only
   there to ensure that the last remaining projects that are still
   relying on `Defined` to make theorems transparent or on `Qed` to
   make definitions opaque do use the attributes or the flags.

3. Remove the error and make `Defined` and `Qed` true synonyms.  Now
   the transparency status is only determined by the attributes or the
   flags.  Again, this could happen only several releases after the
   error was introduced to make sure that no projects remain unported.

Some people might want to change the default transparency only for a
subset of commands.  For instance, some people might want to make
`Example` opaque by default.  Or some people might want to adopt a
convention of always using the same command name (e.g. `Fact` or
`Property`) for transparent theorems.  Therefore it might be useful to
provide both a generic flag `No Sealed Declarations` and some
command-specific ones.  When set, the generic flag would override the
command-specific settings, and when unset, the command-specific
settings would apply.  The following table summarizes the default
transparency statuses and the command-specific flags to change them:

| Command name | Default transparency | Change default with |
|:-:|:-:|:-:|
| `Definition` | transparent | `Set Sealed Definitions` |
| `Example` | transparent | `Set Sealed Examples` |
| `Fixpoint` | transparent | `Set Sealed Fixpoints` |
| `CoFixpoint` | transparent | `Set Sealed CoFixpoints` |
| `Theorem` | opaque | `Unset Sealed Theorems` |
| `Lemma` | opaque | `Unset Sealed Lemmas` |
| `Fact` | opaque | `Unset Sealed Facts` |
| `Remark` | opaque | `Unset Sealed Remarks` |
| `Corollary` | opaque | `Unset Sealed Corollaries` |
| `Proposition` | opaque | `Unset Sealed Propositions` |
| `Property` | opaque | `Unset Sealed Properties` |

# Drawbacks

- This creates more ways of doing the same thing.  People could decide
  to write things like:

  ```coq
  #[ unsealed ]
  Theorem negb (b : bool) :=
    match b with true => false | false => true end.

  #[ sealed ]
  Definition negb_idem (b : bool) : negb (negb b) = b :=
    match b with true => eq_refl | false => eq_refl end.
  ```

  But it seems consistent with the current philosophy of the language
  to give many synonyms for doing the same thing and trusting the user
  to produce readable scripts in the end.

- The first part of the CEP will impose users of the existing `Theorem
  ... with` syntax to add a `recursive` attribute to their
  declarations.

- The second part of the CEP will impose users of the `Theorem` /
  `Defined` combo to add an `unsealed` attribute to their declarations
  or to set some of the new flags.  Thanks to the flags, this porting
  step should be pretty uninvasive.

# Alternatives

- We could keep the syntax separate and keep waiting for users asking
  for this or that extension to implement it, but it is unlikely that
  the syntax would ever converge, and it would remain inconsistent and
  confusing.  The first chapter of the refactored reference manual
  could rely on the unsatisfactory `Proof @term` syntax to avoid
  bringing the tactics in.

- We could implement this plan without the second part.  This would
  not be a big problem, but it would leave a small inconsistency
  between the way transparency is set when the body is provided or
  not.  In this case, the `sealed` and `unsealed` attributes would
  raise an error when the body is not provided.

- We could implement this plan without providing the `sealed` and
  `unsealed` attributes at all.  This means that when a body is
  provided, `Theorem` and `Definition` would not be interchangeable
  (but they already mostly are when in proof mode because the `Qed`
  and `Defined` commands are not forced upon the user, so this would
  not be consistent).

- We could also decide to implement this plan but make `Definition`
  and `Theorem` fully interchangeable.  A `Theorem` with a body would
  be transparent by default, a `sealed` attribute could be used on
  `Definition` or `Theorem` to make the body opaque.  In this case, we
  would not implement the second part of the CEP either.

# Unresolved questions

- The current `Theorem ... with` syntax has been criticized for
  allowing proofs that are rejected at `Qed` time (unless the
  `Guarded` command is used).  The problem is exactly the same as when
  using the `fix` or `cofix` tactics.  When using `Fixpoint` in proof
  mode, it would seem that the `@fixannot` clause is ignored.  We
  could look into making this clause being used and the use of tactic
  becoming safer, but IMHO as a first step it suffices to properly
  document this behavior and possibly warn when using the `@fixannot`
  clause that it is ignored.

- Another command that should be merged with `Definition` is
  `Instance` (an `instance` attribute could be supported on
  `Definition` and variants).  However, given that `Instance` itself
  has several additional peculiarities (supports `{ }` syntax, accepts
  auto-generating a name for the instance), this could also happen as
  a later step.

  The current syntax of `Instance` is the following:

  ```
  Instance {? @ident_decl {* @binder } } : @type {? @hint_info } {? := @instance_body }

  hint_info ::= | {? @num } {? @term }

  instance_body ::= { {*; field_def } }
                  | @term
  ```
