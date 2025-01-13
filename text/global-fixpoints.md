- Title: Named fixpoints and cofixpoints

- Drivers: Hugo Herbelin

----

# Summary and motivation

The Calculus of Inductive Constructions (CIC) originally comes with anonymous fixpoints of the form `fix f args := body(f)` and cofixpoints of the form `cofix f args := body(f)` with reduction rules which - when allowed to be triggered - lead respectively to `body(fix f args := body(f))` and `body(cofix f args := body(f))`.

It is however common to want to hide the size or complexity of such expression behind a name.

This can easily be done with a definition, and this is the way it is done since quite the beginning of the CIC (Coq version 5). Hiding a fixpoint or cofixpoint expression behind a name has however some drawbacks with respect to reduction:

- reduction, such as defined by the CIC, discards the original name and a "refolding" step, such as `simpl` or `cbn` do is necessary to express again the fixpoint or cofixpoint in term of its name
- moreover, this reduction involves quite a number of steps, see an example below (*)

The objective of this Coq enhancement proposal is to provide "natively" named fixpoints and cofixpoints whose reduction rules directly produce what we intuitively expect, i.e. in, the case (*) below, directly go from `add (S t) m` to `S (add t) m`.

The idea is well-known and used in other proof assistants (e.g. Agda): it is that the body of the fixpoint or cofixpoint refers to the global name associated to the fixpoint rather than to the root of the anonymous `fix` or `cofix` expression.

(*) Examples of refolding such as it is done today:

```
Fixpoint add n m :=
  match n with
  | 0 => m
  | S n => S (add n m)
  end.
```
is syntactic sugar for:
```
Definition add :=
  fix f n {struct n} := fun m =>
  match n with
  | 0 => m
  | S n => S (f n m)
  end.
```
When applied to a constructed term, say `S t`, it reduces as follows:

```
add (S t) m
```
→_δ
```
fix f n := fun m =>
  match n with
  | 0 => m
  | S n => S (f n m)
  end (S t) m
```
→_fix
```
fun n m => match n with
  | 0 => m
  | S n => S (fix f n := fun m => match n with 0 => m | S n => S (f n m) end n m)
  end (S t) m
```
→_βɩ
```
S (fix f n := fun m => match n with 0 => m | S n => S (f n m) end t m)
```
Then, an upward δβ-reduction step can be done to reuse the original name:
```
S (add t m)
```

# Proposed design

We give the plan for fixpoints. The plan for cofixpoints is similar.

## Reduction policy

Two kinds of reduction for named fixpoints can be considered.

1. Guarded unfolding, generative names
  - such global fixpoint unfolds only when its main argument is constructed
  - the unfolding is done by substituting the arguments in the body (no substituting of the recursive call though since the recursive call is already built from the global name of the fixpoint)
  - this would typically be governed by the δ flag for this constant (no need a priori to expect also the `fix` flag)
  - equality of such named fixpoints is only by name: two distinct definitions of `add` would be different; such fixpoint would also be different from an anonymous fixpoint

2. Guarded unfolding, structural equality
  - same reduction policy as above
  - equality is considered structurally: i.e. the comparison of two such fixpoints is done by comparing their body in a context where their names are considered equal; similarly, they are compared to anonymous fixpoints by looking at the body

One could wonder if unfolding based on the δ flag for this constant is ok, even if the constant is not applied to a constructor in its main position, but this would require to control the reduction a way or another since otherwise, one could unfold the new occurrence of the constant in its body indefinitely often.

We believe that the guarded unfolding with structural equality is intuitive enough and would provide a better compatibility.

## Behavior with respect to the guard condition check

The conservative approach is to turn named fixpoint as anonymous fixpoints for the purpose of the guard checker algorithm. (Maybe better can be done though.)

## Kernel declaration
 
For fixpoints, we plan to add a new constructor to `Declarations.constant_def`:
```
type ('a, 'opaque) constant_def =
 | Undef of inline
 | Def of 'a
 | OpaqueDef of 'opaque
 | Primitive of CPrimitives.t
 | Fix of ('a,'a) Context.Rel.Declaration.pt list * 'a
```
where the `'a` is intended to be `constr` or `fconstr` and the decreasing argument is the last one of the context (which has to be an assumption).

The `rel_context` here (i.e. `('a,'a) Context.Rel.Declaration.pt list`), is to pally the limitations of the guardness checker which may fail to recognize as guarded a lambda-lifted fixpoint (see this [Coq wish](https://github.com/coq/coq/wiki/Wishes#recognizing-uniform-parameters-in-fixpoints) for an explanation of the problem). Thus, we need to remember the prefix of uniform parameters of a fixpoint.

We plan to add a `Safe_typing` function to register a (mutual) global fixpoint in the safe environment:
```
val add_mutual_fixpoint :
  Id.Set.t option -> Constr.rel_context ->
  Entries.universes_entry -> Constr.fixpoint ->
  safe_environment -> Names.Constant.t array * safe_environment
```
The effect of `add_mutual_fixpoint ids Γ univs (fix f₁ Γ₁ := body₁ ... with fn Γn := bodyn) env` is to check that the term `fun Γ => fix f₁ Γ₁ := body₁ ... with fn Γn := bodyn` is well-typed and to add the following (recursive) constants to the environment:
```
ModulePath.f1 := ... { const_body = Fix (ΓΓ₁, body1[fi:=ModulePath.fi Γ]) } ...
...
ModulePath.fn := ... { const_body = Fix (ΓΓn, body1[fi:=ModulePath.fi Γ]) } ...
```
and to return the new certified environment and the names of the mutually defined constants.

This function will follow the same pattern as `add_constant`. In particular, it will depend on an auxiliary function in `Term_typing`:
```
val translate_mutual_fixpoint :
  env -> Constant.t array -> Id.Set.t option ->
  Constr.rel_context -> Entries.universes_entry ->
  Constr.fixpoint -> 'a constant_body array
```

Other functions will be added, in parallel to `add_constant`, up to the `vernac` directory and `comFixpoint.ml`.

A preliminary kernel investigation can be found [here](https://github.com/herbelin/github-coq/pull/new/master+global-fixpoint).

## Reduction machines

The various reduction machines (lazy, cbv, reductionops.ml, ...) need to be modified as follows: when the δ flag is on for a constant whose body is a `Fix` and which satisfies the conditions for unfolding is expanded into its body, substituting the context with the actual arguments.

The conversion and unification need to be modified as follows: when comparing such global fixpoint not allowed to be unfolded to another term, the other term must be an anonymous fixpoint or a named fixpoint itself and both fixpoint bodies should be same under the assumption that the recursive calls are the same.

## Normalization

We claim that the new reduction does not change the decidability of the conversion. Any two convertible terms with named fixpoint are convertible iff the same terms where the named fixpoints are simulated from anonymous fixpoints are convertible. Any term with infinite weak-head reduction would be also infinitely reducible after replacement of the named fixpoints by anonymous fixpoints.

# Further possible directions

- We may provide refined variants of the existing reduction rules for fixpoints: e.g. indicate that `add` will unfold only when applied to 2 arguments. Such kind of refinement would not make the kernel more complicated.

- Add support for recursive definitions also in the local context (`let rec`).

  Their reduction rules would be the same as for global fixpoints, i.e. such a `let rec` would never reduce (unless maybe if there is no more references to it in its scope, but even that is not really necessary). Its only effect would be on the reduction of expressions referring to it in its scope.

  Since `rel_context` are quite pervasively used in the implementation, that would be much more changes to do in the code though.

# Compatibilities

The most unclear issue is the one of the compatibility. Strictly speaking, this is a new feature and it could be stepwise tested on a definition-per-definition basis (with e.g. an attribute).

If it happens that it mostly preserves compatibility this could become the default behavior.
