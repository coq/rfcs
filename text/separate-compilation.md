- Title: Separate Compilation in Coq

- Drivers: David Swasey, Paolo Giarrusso, Gregory Malecha

# Summary

Coq provides a module system that can be used explicitly through commands such
as `Module` and `Module Type`. These can be quite heavyweight in many instances,
and have some limitations when it comes to separately compiling files and
building generic libraries.
To address these problems, and support information hiding and Cardelli-style separate
compilation, in this CEP we propose a notion of Coq interface files, which we
will call a `.vi` file, and which are inspired by OCaml's `.mli` files.
Intuitively, a Coq interface file called `module.vi` defines the public
interface for `module.v`. The `module.vi` interface shall enable developing and
typechecking clients, even before `module.v` has been implemented.

If both `module.vi` and `module.v` are present, `module.vi` shall act as an
opaque ascription for the top-level module defined by `module.v`; this opaque
ascription ensures that clients that typecheck against `module.vi` shall still
typecheck against the combination of `module.vi` and `module.v`, regardless of
the implementation details of `module.v`, including any non-logical side effects
such as hints (but excluding [universes](#universes)). Instead, without
interfaces, adding implementations is sufficient to break clients, and changing
implementations can break clients again, hindering modular development.

This has a few advantages compared to Coq's state of the art:
- It enables separate development: after agreeing on `module.vi`, `module.v` and
  its clients can be developed independently. To ensure this, unlike today's
  opaque ascription, `.vi` files can even hide side effects due to `Require`.
- It reduces compile-time dependencies and improves compile times, even compared
  to today's `vos` builds (initial builds can be more parallel, and incremental
  builds need to recompile fewer files).

NOTE: For concreteness, in this CEP we use the `.vi` extension for for interface
source files, and the `.vio` extension for interface object files. However,
`.vio` files should not be confused with `.vio` files produced by `-quick`
builds: to avoid confusion, we could choose other file extensions or remove
`-quick` builds entirely.

# Proposed Semantics

In this section, we describe the meaning of our Coq extension: instead of
suggesting an implementation strategy, we sketch a semantics as a
source-to-source transformation, to be taken as an informal and _approximate_
specification. However, this is an approximate specification, because the exact
semantics cannot be expressed via source-to-source transformation.

As a key principle: `.vi` interfaces are meant to hide implementations and
support separate compilation in Cardelli's sense. Hence, compiling a module
`consumer.v` that consumes the interface of `producer.vi` shall not depend
(directly or indirectly) on the existence of `producer.v` or its contents. As a
consequence, no change to `producer.v` can affect whether `consumer.v`
typechecks.

More concretely, our semantics for compiling top-level module (say, `lib`)
distinguishes three scenarios, depending on the existence of:
- both `lib.vi` and `lib.v`;
- only `lib.vi`;
- or only `lib.v`.

## Both a `.vi` and `.v` File

For concreteness, consider the following example:

```coq
(* lib.vi *)
Global Open Scope Z_scope.
Parameter answer : nat.
Axiom answer_is_42 : answer = 42%nat.

(* lib.v *)
Require Import stdpp.prelude.
Global Open Scope N_scope.

Definition answer : nat := 42.
Definition answer_is_42 : answer = 42%nat := ltac:(reflexivity).

Definition other_answer : N := 42 + 1.
```

Here, `lib.v` contains both the implementation of `lib.vi` and some commands
that perform global side effects and that should be hidden from clients:
`stdpp.prelude` modifies many Coq settings aggressively, even when simply
`Require`d, so adding this `Require` is a significant breaking change. 

In this example, the semantics of `lib` would resemble the semantics of the
following Coq source:

```coq
(* lib_composed.v: *)
Module Type __LIB.
  Global Open Scope Z_scope.
  Parameter answer : nat.
  Axiom answer_is_42 : answer = 42.
End __LIB.

Module __lib : __LIB.
  Require Import stdpp.prelude.
  (* ^ Require inside a Module is strongly discouraged. What is relevant is
     that side effects of the Module are not visible outside of this Module
     *even* when the Module is Import-ed *)
  Global Open Scope N_scope.

  Definition answer : nat := 42.
  Definition answer_is_42 : answer = 42%nat := ltac:(reflexivity).
End __lib.

Export __lib. (* make the declarations from lib available from [Import]ing lib (rather that lib.lib *)
```

In this example translation, we use the `\_\_` prefix for generated identifiers
used as a translation device; these identifiers should be hidden from clients.

This translation turns the `.vi` interface into a module type, the `.v`
implementation into a module, and uses opaque module ascription to hide the
module contents.
The only visible side effects are those that appear in the interface: here,
only `Global Open Scope Z_scope`. As we only `Require` stdpp.prelude in the
module body, we intend this to be **hidden** from clients that perform `Require
lib`, even if this might not be guaranteed by `lib_composed.v`.

## Only a `.vi` File

When only an interface file exists, there is (potentially) no underlying
implementation, but the existance of the interface should still provide definite
references to an implementation. Casting this in the feature set of the current
Coq ecosystem, this essentially translates to a `Declare Module`
[^full-functorization]. Concretely,

```coq
(* lib.vi *)
Global Open Scope Z_scope.
Parameter answer : nat.
Axiom answer_is_42 : answer = 42.
```

would be "translated" to:

```coq
(* lib.v *)
Module Type __LIB.
  Global Open Scope Z_scope.
  Parameter answer : nat.
  Axiom answer_is_42 : answer = 42.
End __LIB.

Declare Module __lib : __LIB.
Export __lib.
```

## Only a `.v` File

If only a `.v` file is available, Coq's semantics should be the same as today:
the entire implementation shall be exposed in full builds, and it shall be
hidden in `vos` builds.

## Universes

As some readers will anticipate, universe checks do not admit fully separate
compilation. Module bodies might add universe constraints that are note explicit
(or even visible) from its interface. This is already an issue with `.vos`
builds today, and is a problem inherent to Coq's global universe graph and
(generally) implicit treatment of universes in the surface syntax, so any
solution to this problem could be shared.

We consider `vos` builds a special case of this proposal, where interfaces are
inferred as the strictest possible ones for the given implementation; `.vi`
files enable hiding more implementation details. In both cases, the interface
omits universe constraints that are derived from hidden bodies (for `vos`
builds, just `Qed` bodies). While some universe checks are performed anyway,
omitted constraints might make the universe graph unsatisfiable.

To remedy this problem, we propose an additional "global" check. By analogy with
separate compilation in other languages, we call this "link-time" universe
checking. Such a link-time would be effectively the same as `Require`ing all of
the modules in a library preferring their *implemenation* (`.v` file) rather
than their interface (`.vi` file). If the combined set of constraints is
satisfiable, then there is no problem.

If the combined set of constraints is *not* satisfiable, then the problem must
arise from an interface not exposing a constraint that an implementation
requires. Diffing the universe constraints between an interface and an
implementation would provide the right information necessary to diagnose and fix
the problem.

### "Full compilation" Semantics

The notion of full compilation semantics, i.e. a sound full-build semantics akin
to a "vo-style" build can be achieved (at the cost of build parallelism) by
introducing a dependency between the `.vio` file and the `.vo` file and
elaborating the resulting `.vio` file with universe constraints introduced by the
implementation. It is important that nothing but universe constraints (say
hints, tactics or plugin requirements) leak from `.vo` files into `.vio` files.

Alternatively, the extra universe constraints could be fetched directly from
`.vo` files without altering the `.vio` files: again, nothing but universe
constraints should leak from implementations to clients.

In this case, we could maybe ensure that `.vio` files coincide across separate
builds and "full" builds.
To this end, processing `Require library` when building `client.vio` file might
have to not load constraints from `library.vo`, even in a full build.

# Implementation

The implementation would require (at least) the following:

1. Extending the build infrastructure to support `.vi` compilation.
2. Modifying the implementation of `Require` to search for `.vio` files in
   addition to `.vo` files. For backwards compatibility, we believe it is
   necessary to search for both `.vio` and `.vo` files *simultaneously* rather
   than first searching for a `.vio` everywhere and then for a `.vo` everywhere
   because the later would mean that adding a `.vi` files could change the
   library that is used.
3. We believe that the bit-level representation of `.vio` could be the same as
   `.vo` files, though an alternative would be to leverage the representation of
   `.vos` files (which might be the same).
4. Build support for the new mode; ideally, some code could be shared with
   support for `vos` builds.


# Footnotes

[^full-functorization]: An alternative characterization of `Declare Module` is
    to implicitly functorize all dependent modules over the module type of the
    declared module. This understanding of `Declare Module` does enable some
    additional flexibility at "link-time" but is beyond the scope of this
    proposal.
