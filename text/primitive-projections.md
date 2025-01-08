- Title: What model for primitive projections?

----

# Summary

Projections of record fields currently follow different overlapping paths:
- constructors-based vs destructors-based normal forms
- `t.(f params)` vs `f params t` concrete representation
- unfolded `Proj` vs folded `Proj` vs `App`-`Const` internal representation

This document is an attempt to propose solutions to clarify the picture.

See also [#14640](https://github.com/coq/coq/pull/14640) which tries to set up a context to stabilize a design choice.

## Issue 1: the relative status of `Const`-based projections and `Proj`-based projections

It is common to write `f param t` to mean a compact `Proj` kernel node. It is also useful to have a non compact representation for partial application of a projection or for an unapplied projection. What to do with it?

*Design choice 1*: The kernel provides only compact `Proj` nodes for projections in primitive record types. It is the responsibility of the user (or of the vernac level) to later declare `Definition f' params x := x.(f params)` (with a new name) and to use it as an ordinary definition.
  - Advantages: a simple model
  - Drawbacks: the name `f'` is different

1. *Design subchoice 1.1*: `Proj` nodes are exclusively parsed and printed using the `x.(f)` without parameters.
   - Advantages: a direct correspondence between concrete and internal syntax
   - Drawbacks: possible confusion with the more general syntax for projections of non-primitive records which has the form `x.(f params)` and possible confusion with the "tolerance" for using at parsing time the notation `x.(f params)` for any constant `f` even if not a projection

2. *Design subchoice 1.2*: Idem but using a different syntax such as `x.[t]`, or `x.((t))`, or `x.{t}`, etc.
   - Advantages: no confusion with the existing uses of the `x.(f)` syntax
   - Drawbacks: reserves a new syntactic construction

3. *Design subchoice 1.3*: `Proj` nodes are exclusively parsed and printed using the `x.(f _ ... _)` syntax where the `_` take the place of the parameters, noticing however that if the parameters are declared implicit, this reduces to the notation `x.(f)`.
   - Advantages: compatibility with the existing syntax for projections in non-primitive records
   - Drawbacks: one may still want a clearer distinction from the other uses of the syntax `x.(f params)`

4. *Design subchoice 1.4*: `Proj` nodes are exclusively printed using the `x.(f _ ... _)` syntax (reducing to `x.(f)` with implicit parameters) but it is tolerated to use `f params x` at parsing as a compatibility notation for `Proj(f,x)` (when `f` is a projection in a primitive record type).
   - Advantages: compatibility with the existing usage of often writing `f params x` for a primitive projection
   - Drawbacks: asymmetry parsing/printing (but deprecating `f params x` could eventually clarify things)

*Design choice 2*: Whenever a record is defined, a `Const f` node of same name `f` is made available at the same time (with body `fun params x => x.(f)` that is `Lambda` over a `Proj`) and `Proj (f,x)` and `App (Const f, params, x)` are considered convertible by default without needing any form of explicit unfolding. Note that the infrastructure to make them convertible basically already exists in coercions, unification, ...
  - Advantages: the two representations are fully equivalent (up to the compact representation and the loss of the exact form of parameters)
  - Drawbacks: the code to identify `Const` and `Proj` might be considered intrusive (but at least, it is alreay there, using `Projection.constant` and `expand_projection` machinery)

1. *Design subchoice 2.1*: When the syntax `f params t` is used, does it bind automatically to the compact representation (as it is done now), and thus reprinted as `t.(f params)`?
   - Advantages: compatibility with the current usages
   - Drawbacks: it does some transformation under the carpet

2. *Design subchoice 2.2*: Any application of `Const f` to enough arguments is automatically turned by `mkApp` into a `Proj` (this assumes that any constant `f` contains an information on its projection status)? In particular, `Const` is used only for non-applied or not-enough-applied projection.
   - Advantages: a simple model; no need to identify `Proj (f,x)` and `App (Const f, params, x)` in conversion
   - Drawbacks: maybe a few incompatibilities due to the dropping of arguments?

3. *Design subchoice 2.3*: delta-reduction is refined so that in the case of `Const f`, the reduction can happen only when `f` is applied to at least the number of parameters + 1. We arrange at parsing time that `Const` is used only for non-applied or not-enough-applied projection (even if it can be changed later by reduction).
   - Advantages: the natural expectation
   - Drawbacks: maybe a bit ad hoc, though controlling reduction with properties of the arguments is not new (already done for `simpl`)

Related: [#14084](https://github.com/coq/coq/pull/14084) (`match goal` involving primitive projections) and [#11366](https://github.com/coq/coq/issues/11366) (which requires to be able to declare partially applied projections as coercions).

## Issue 2: the relative status of "folded" vs "unfolded" `Proj` node

*Design choice 1*: The "folded" vs "unfolded" boolean is dropped. Unfolding a `Proj` node is a no-op. In scenario 2.1 of issue 1, unfolding a `Const` denoting a projection reveals a `Proj` node. In scenario 2.2, it is a no-op.

*Design choice 2*: Idem but unfolding a primitive projection applied to the constructor of the record performs the iota-reduction (like in the positive case) instead of a being a no-op.

Related: [#5250](https://github.com/coq/coq/issues/5250), [#5698](https://github.com/coq/coq/issues/5698), [#5699](https://github.com/coq/coq/issues/5699) (effect of unfolding tactics).

See associated [Zulip discussion](https://coq.zulipchat.com/#narrow/stream/237656-Coq-devs.20.26.20plugin.20devs/topic/Primitive.20Projection.20mode).

Note: effect of various reduction commands on the diverse representations of projections:

|                                | non-primitive | prim Const        | prim folded Proj | prim unfolded Proj | expected
| ------------------------------ | ------------- | ----------------- | ---------------- | ------------------ | --------
| unfold on non-`Build`          | to `match`    | to unfolded Proj  | to unfolded Proj | nop                | error (2) or nop (1)
| unfold at on non-`Build`       | to `match`    | to unfolded Proj  | error (bug?)     | error              | error or nop?
| unfold on `Build`              | contract      | contract          | contract         | contract           | contract
| unfold at on `Build`           | contract      | contract          | error (bug?)     | error (bug?)       | contract
| simpl on arg non-red `Build`   | nop           | nop               | nop              | nop                | nop
| simpl on arg red `Build`       | contract      | contract          | contract         | contract           | contract
| simpl never on arg red `Build` | nop           | nop               | nop              | contract (!)       | nop
| simpl nomatch non-red `Build`  | nop           | nop               | nop              | nop                | nop
| delta                          | to expanded `match` | to expanded unfolded Proj | nop (?!) | nop          | nop
| delta+beta                     | to `match`    | to unfolded Proj  | to unfolded Proj | nop                | nop
| iota on `Build`                | contract      | nop               | nop              | contract           | contract

<details>
<summary>Coq File used to test the reduction commands</summary>

```coq
Module NotPrim.
(* Test effect on non-primitive Projections *)
Record F A := { a : A }.

Module NotBuild.

Goal forall x, x.(a nat) = 0.
cbv beta.
(* Now an applied Const *)
unfold a. Undo.
unfold a at 1. Undo.
simpl. Undo.
Local Arguments a : simpl never.
simpl. Undo.
cbv delta [a]. Undo.
cbv beta delta [a]. Undo.
Abort.

End NotBuild.

Module OnBuild.
Goal {|a:=0|}.(a nat) = 0.
unfold a. Undo.
unfold a at 1. Undo.
simpl. Undo.
Local Arguments a : simpl never.
simpl. Undo.
cbv delta [a]. Undo.
cbv beta delta [a]. Undo.
Abort.
End OnBuild.

Module RedToBuild.
Goal (id {|a:=0|}).(a nat) = 0.
cbn beta.
(* Now an applied Const *)
unfold a. Undo.
unfold a at 1. Undo.
simpl. Undo.
Arguments a : simpl nomatch.
simpl. Undo.
cbv delta [a]. Undo.
cbv beta delta [a]. Undo.
Abort.
End RedToBuild.

End NotPrim.

(**********************************)

Module ConstPrim.
(* Test effect on Const in Primitive Projections *)
Set Primitive Projections.
Record F A := { a : A }.

Module NotBuild.

Goal forall x, (fun A => a A) nat x = 0.
cbv beta.
(* Now an applied Const *)
unfold a. Undo.
unfold a at 1. Undo.
simpl. Undo.
Local Arguments a : simpl never.
simpl. Undo.
cbv delta [a]. Undo.
cbv beta delta [a]. Undo.
Abort.

End NotBuild.

Module OnBuild.
Goal (fun A => a A) nat {|a:=0|} = 0.
cbv beta.
(* Now an applied Const *)
unfold a. Undo.
unfold a at 1. Undo.
simpl. Undo.
Local Arguments a : simpl never.
simpl. Undo.
cbv delta [a]. Undo.
cbv beta delta [a]. Undo.
cbv iota. Undo.
Abort.
End OnBuild.

Module RedToBuild.
Goal (fun A => a A) nat (id {|a:=0|}) = 0.
cbn beta.
(* Now an applied Const *)
unfold a. Undo.
unfold a at 1. Undo.
simpl. Undo.
Arguments a : simpl nomatch.
simpl. Undo.
cbv delta [a]. Undo.
cbv beta delta [a]. Undo.
Abort.
End RedToBuild.

End ConstPrim.

(**********************************)

Module FoldedProj.

Set Primitive Projections.
Record F A := { a : A }.

Module NotBuild.

Goal forall x, x.(a nat) = 0.
match goal with |- forall x, x.(a nat) = 0 => idtac end.
unfold a. Fail match goal with |- forall x, x.(a nat) = 0 => idtac end. Undo 2.
Fail unfold a at 1. Undo.
simpl. Undo.
cbv delta [a]. match goal with |- forall x, x.(a nat) = 0 => idtac end. Undo 2.
cbv beta delta [a]. Fail match goal with |- forall x, x.(a nat) = 0 => idtac end. Undo 2.
unfold a.
Abort.

End NotBuild.

Module OnBuild.
Goal {|a:=0|}.(a nat) = 0.
unfold a. Undo.
Fail unfold a at 1. Undo.
simpl. Undo.
Local Arguments a : simpl nomatch.
simpl. Undo.
cbv delta [a]. match goal with |- {|a:=0|}.(a nat) = 00 => idtac end. Undo 2.
cbv beta delta [a]. Fail match goal with |- forall x, x.(a nat) = 0 => idtac end. Undo 2.
cbv iota. Undo.
Abort.

End OnBuild.

End FoldedProj.

(**********************************)

Module UnfoldedProj.

Set Primitive Projections.
Record F A := { a : A }.

Module NotBuild.

Goal forall x, x.(a nat) = 0.
unfold a.
(* now unfolded *)
unfold a. Undo.
Fail unfold a at 1. Undo.
simpl. Undo.
cbv delta [a]. Undo.
Abort.

End NotBuild.

Module OnBuild.

Goal {|a:=0|}.(a nat) = 0.
cbv beta delta [a].
(* now unfolded *)
unfold a. Undo.
Fail unfold a at 1. Undo.
simpl. Undo.
cbv delta [a]. Undo.
cbv beta delta [a]. Undo.
cbv iota. Undo.
unfold a. Undo.
Local Arguments a : simpl never.
simpl. Undo.
Abort.

End OnBuild.

End UnfoldedProj.
```
</details>

Note 2: should a reduction specific name be added for projections (e.g. `proj`) which `iota` would include?

## Issue 3: the status of `match` and destructing `let` in "primitive" record types
 
See impact of desugaring `match` in terms of complexity ([#13913](https://github.com/coq/coq/pull/13913)) or unification ([#6726](https://github.com/coq/coq/issues/6726), [#9763](https://github.com/coq/coq/issues/9763)) or readability ([#6723](https://github.com/coq/coq/issues/6723)).

*Design choice 1*: keep `match` nodes as they are in primitive record types; they will behave correctly for reduction (that is like their expansion into projections); to deal with them in the conversion, it is enough to extend the conversion with eta-expansions of `match` into projections.

   - Advantages: a priori no break of compatibility for the developments already using `match` on primitive record types

*Design choice 2*: forbid `match` at all on primitive record types (as well as `fun 'pat => ...` and `let 'pat := ... in ...` and `let (x1,...,xn) := ... in ...`), even as a primitive form of sugar, making primitive and non-primitive record types two incompatible styles of formalization.

   - Advantages: model consistent with viewing primitive record types as negative types in polarized logic
   - Drawbacks: unclear how to address the compatibility with existing code

## Issue 4: The status of the `t.(f args)` notation as an alternative notation for any kind of application.

See e.g. [#11332](https://github.com/coq/coq/pull/11332).

*Design choice 1*: Accept the notation `t.(f args)` as a shorthand for `f args t` even when `f` is not a projection even if this is lost at printing time.
  - Advantages: nothing more to do
  - Drawbacks: no printing

*Design choice 2*: Give the ability to declare a table of "methods" supporting the `t.(f args)` notation. Only such registered method can use the notation. The notation is used for printing too. Internally, it is an `App` node.

*Orthogonal design choice*: same as design choices 1 and 2 but with a different notation, e.g `t.{f args}`.

## Issue 5: The representation of projections in "positive" types. Can they take benefit of a compact representation?

*Design choice 1*: We don't make a difference between positive and negative types. Both supports a `Case` and a `Proj` node which behave anyway the same wrt reduction even when we think at them as syntactic sugar. We add eta-rule for `Case` in the conversion, which are the current rules for expansion into `Proj`. The only difference is that `Proj` is mentally thought as in normal form for a negative type and as a short-hand for a `match` for a positive type. As for `Case`, it can either be mentally thought as a short-hand for its let-proj-expansion for a negative type or explicitly expanded in `cbn` and `lazy`, while `Case` for a positive type is in normal form.
   - Advantages: a simple model for the user
   - Drawbacks: not a conventional view in polarized logic

Note: consistency between positive and negative record types would require to add support for named projections of local definitions of records.

*Design choice 2*: Idem but `Proj` expands to `match` in positive types.
   - Advantages: conventional view in polarized logic
   - Drawbacks: `match`-based representation of projections are uselessly verbose
