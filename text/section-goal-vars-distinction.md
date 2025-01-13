- Title: On referring to section variables in goals

- Drivers: Hugo Herbelin

----

# Summary

There is a long-standing issue about section variables in goal being shadowed with goal variables of same name. This proposal sketches how to fix them by giving a qualified name to section variables.

To go directly to the point, see item *Proposal* below.

# Historical context

Introduced at the end of the 80's in Coq and inspired from a common practice in mathematical books, sections provide with a way to share variables across definitions and goals (see section 6.3 of _A Vernacular Syllabus_ by Gilles Dowek).

Variables declared in a section behave for the timespan of the section like axioms. They can occur in definitions (and theorems) of the section, and, thus, indirectly, in the other definitions of the section that depend on these definitions in which they directly occur. It is only at the end of the section that they are _discharged_, i.e. distributed, over all definitions (and theorems) that depend on them.

Proof mode is about statements to prove under assumptions. In proof mode, new hypotheses can be added by means of (typically) `intros`. In proof mode, one wants to control the visibility of the hypotheses of the proof, so as to guide automation, but also, simply, for visually managing the size of the proof context. This control is typically provided with the tactics `clear`, `rename` as well as subsequent `intros` possibly replacing occurrences of variables by others of same name.

Section variables are conventionally added to the initial context of an interactive proof. The fact that they behave also as global names created confusions, leading to think that it was unsafe to clear them or shadow them while the problem can alternatively be seen as the question of how to continue to refer to section variables as global names even when they are removed from the goal context.

# Some terminology

Let us call _global context_ the context of typing of a global definitions (`safe_env` in Coq). It can be represented as a sequence of named declarations separated with section markers (we skip many features not important for our purpose: universes, opacity, inductive types, ...):

```
Γ ::= ∅ | Γ; SectionMarker
   | Γ; Var a:T | Γ; Let a:=t:T
   | Γ; Ax a:T | Γ; Def a:=t:T
```

A _goal_ is an existential variable representing a hole to be filled in a given context. A goal is progressively refined into subgoals, themselves represented by other existential variables.

Existential variables have a type and are defined in what we shall call a _goal context_ for this type where a goal context is an extension of a global context as defined by this grammar:
```
Δ ::= Γ | Δ; a:T | Δ; a:=t:T
```
Names of a goal context need not to be unique. In particular, they can be thought as represented either by de Bruijn numbers or with the convention that the last occurrence of a same name shadows the previous occurrences (which requires that the shadowed variable is not used any further in the extension of the context).

Existential variables come with a filter which tells which section or goal variables are allowed to occur in the term intended to fill the hole. A filter is any removal or renaming of `Var a:T`, `Let a:=t:T`, `a:T`, or `a:=t:T` in the goal context so that the resulting goal context:
- is still well-formedness,
- preserves the typability of the type of the existential variable,
- do not have two declarations with same names.

# Typical issues to solve

This CEP is about the unability to distinguish a living section variable from a goal variable shadowing the name of the section variable. Some bugs related to this have been fixed along the years (sometimes with unnecessary restrictions which the proposal should be able to lift) but some remain due to collision between names. Here are two typical currently problematic situations:
- names mixed up causing confusions in tactics: coq/coq#11487, coq/coq#11729
- `abstract` unable to know whether a goal variable was actually a section variable: coq/coq#6673

A typical example of confusion is the following:
```coq
Section S.
Variable B : nat.
Definition C := B.
Goal nat -> True.
clear B. intro B.
assert (C = B).
unfold C. (* wrongly shows B = B *)
reflexivity.
exact I.
Fail Qed. (* The term "eq_refl" has type "B0 = B0" while it is expected to have type "C = B0". *)
```

Another one is:
```coq
Section S.
Variable a:nat.
Definition b:=a.
Goal forall n:nat, n=n.
clear a.
intro a.
change (b=b). (* wrongly accepted *)
reflexivity.
Fail Qed.
```

An additional issue is extra existing limitations in clearing section variables: Following the principle that all `Var a:t`, `Def a:=t:T`, `a:t`, `a:=t` declarations have an impact on automation and on the visual organization of the context, it should be allowed to clear all variables as soon as they do not break the well-formedness of the typing of the type of the goal. On the basis that section variables are also global, they should be clearable without restriction.

# Proposal

The core of the proposal is to make the type `Names.variable` distinct from `type variable = Id.t` so that it remembers if a variable is a section variable or a goal variable.

A first possibility is to set `type variable = KerPair.t` with the convention that section variables have names of the form `File.Mod.Sec.foo` and goal variables names of the form `foo` (i.e. with empty file name, empty module/section infix), together with a name resolution which binds `foo` to a section variable of base name `foo` when it is not shadowed by another name.

A second possibility is to set `type variable = {issecvar:bool; basename:Id.t}`, leaving to the name table the work of disambiguating when a section variable needs to be qualified to be interpresed as such. Here is a tentative possible API:
```ocaml
module Var :
sig
  type t
  val base : t -> Id.t
  val is_secvar : t -> bool
  val secvar : DirPath.t -> Id.t -> t
end
```

Remark: the proposal is thus a variation about what was discussed at [coq/coq#6254](https://github.com/coq/coq/pull/6254) (future of sections), especially [this comment](https://github.com/coq/coq/issues/6254#issuecomment-480886055) and [this comment](https://github.com/coq/coq/issues/6254#issuecomment-481168862) (distinguishing section variables and goal variables) for the first part, and [this comment](https://github.com/coq/coq/issues/6254#issuecomment-481171412) (representing section variables with constants) for the second part.

Then, in addition to the `Id.t`-based `Context.Named.Declaration` and `named_context`, there would be `variable`-based `Context.Goal.Declaration` and `goal_context`. Similarly, some functions that refer to `Id.t` would instead refer to the new `variable` type (e.g. `mkVar`, `destVar` would be about `variable` but `Environ.vars_of_global`, which is only about section variables, would remain in `Id.t`).

It is not clear yet how easy these changes could be propagated to the whole code basis (here is a very preliminary [attempt](https://github.com/herbelin/github-coq/tree/master+distinguish-secvar)). Ideally, this should be reducible to involving the nametab at declaration/locating/printing time of section variables, and to make precise when a function is about a goal context (most of the time) and when it is about a section context (mostly in the kernel data-structures).

# Remaining design questions

- Name to use to refer to a section variable cleared in the current goal ([coq/coq#12250](https://github.com/coq/coq/pull/12250)).

  If a section variable is cleared, do we force to refer to its name only in a qualified way or do we still accept to refer to its name without qualification. I.e., in:

  ```coq
  Section S.
  Variable a:nat.
  Goal True.
  clear a.
  Locate a.
  ```
  Do we say that `a` can be accessed (only) as `S.a` or also as `a`.

- Tactics `induction`/`destruct` not clearing section variables ([coq/coq#2901](https://github.com/coq/coq/issues/2901), [coq/coq#5220](https://github.com/coq/coq/issues/5220), [coq/coq#7518](https://github.com/coq/coq/issues/7518))

  This is actually relatively independent and is addressed by [#374](https://github.com/coq/coq/pull/374) (pending).

- Renaming section variables

  This is currently allowed but it raises problems:
  ```coq
  Section S.
  Variable a:nat.
  Definition b:=a.
  Goal b=b.
  rename a into c.
  unfold b. (* a = a *)
  reflexivity. (* anomaly *)
  ```

  A section variable is a global object. Either it should not be renamed or its renaming should be interpreted as setting `(c:=a)` and clearing the body of `c`. Not clear what would be the most intuitive.

- Lifting existing limitations

  It is currently forbidden to clear a section variable if the goal refers to a global definitions which depends on this section variable. With qualified section variables, this limitation can be lifted.

  Actually, one may also allow to clear a section variable even if it occurs in the goal. The remaining occurrences would simply be reinterpreted as reference to the global name. This is a design choice.

- Impact on `Proof using`

  The resolution of problems with `clear` should open the way to support `Proof using` clearing variables that are explicitly not wanted ([coq/coq#883](https://github.com/coq/coq/issues/883)).

  If a variable has been explicitly excluded with a `Proof using`, the use of any constant referring to it should also be rejected.

# Other related issues

Another case where it is needed to know when a variable is a copy of a section variable or a variable which shadowed it: [coq/coq#10812](https://github.com/coq/coq/issues/10812) (`subst` on section variables).
