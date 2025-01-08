- Title: Disentangling erasure and proof irrelevance

- Drivers: Abhishek Anand

----

# Summary

Currently, Prop serves the following 3 roles:
1. Impredicativity
2. Proof Irrelevance
3. Erasure

Sometimes, these roles conflict. For example, proof irrelevance prevents applications where specifications need to be defined by pattern matching on some "ghost data" that should be erased at runtime. An example is provided in the Motivation section.

This CEP proposes to create another universe, say `ESet`, to disentangle the role of erasure from `Prop`.
The typehood rules for `ESet` are partly like `Prop` (and partly like `Set`), but more permissive: elimination to computationally irrelevant types, such as universes, is allowed.
As a result, `ESet` does not support proof irrelevance.
ESet is tentatively predicative, because that is currently crucial for our meta-theoretic proofs, which translate away `ESet` to Set.

Extraction should be accordingly updated to guarantee that data of sort `EProp` or `ESet` is erased, at least to \box.
As a result, there should no longer be any need for the `Extraction Implicit` mechanism, whose success was not guaranteed.

# Motivation

Consider the following dependently-typed implementation of a denotational semantics:
```
Inductive ty : Set :=
| Nat : ty
| Prod : ty -> ty -> ty.

Inductive exp : ty -> Set :=
| NConst : nat -> exp Nat
| Plus : exp Nat -> exp Nat -> exp Nat
| Pair : forall t1 t2, exp t1 -> exp t2 -> exp (Prod t1 t2)
| Fst : forall t1 t2, exp (Prod t1 t2) -> exp t1.

Fixpoint typeDenote (t : ty) : Set :=
  match t with
    | Nat => nat
    | Prod t1 t2 => typeDenote t1 * typeDenote t2
  end.

Fixpoint expDenote t (e : exp t) : typeDenote t :=
  match e with
    | NConst n => n
    | Plus e1 e2 => expDenote e1 + expDenote e2
    | Pair e1 e2 => (expDenote e1, expDenote e2)
    | Fst e' => fst (expDenote e')
  end.
 ```

In this example, the elements of the type `ty` are only used for specifying properties (return type) of the function `expDenote`.
Many users wish that elements of type `ty` get erased during extraction to OCaml.
Note that types are computationally irrelevant in Coq: one cannot do a pattern match on something of type Type or Prop.
Thus, even though in `typeDenote`, a `ty` is eliminated to produce a type, the result is not computationally relevant.

One cannot change the type of `ty` from `Set` to `Prop`. That would make `typeDenote` ill-typed.
Also, there is no need for `ty` to be proof irrelevant.

Currently, one can use `Extraction Implicit ty` to request the extractor to erase terms of type `ty`. However, Coq's typesystem cannot prevent the user from unintentionally using members of `ty` in computationally relevant ways. In such cases, `Extraction Implicit ty` can fail.

This problem was pointed out to me by Greg Morrisett.

# Detailed design

In the new Coq, `ty` would have type `ESet`. Just like inductives in `Prop`, there are restrictions on when pattern matching
terms of inductive types in `ESet` is allowed. 
Here are the rules for deriving the allowed return types:
- if `P:Prop`, then `P` is allowed
- if `P:ISet`, then `P` is allowed
- if `P` is allowed, then so is `forall x:A,P`, for any A.
- if `P` is definitionally equal to a sort (`Type` or `Set` or `Prop`), then `P` is allowed

An intuition is that we are ensuring that as a result of extraction, the bodies of the match should each become `\box`.
Thus, the extraction can replace the whole match term with `\box`.

## Soundness

### Does the change preserve Coq's SN, type preservation, confluence etc.?
(I believe the existing Coq, at least without coinductives, has all those properties.)
It seems the above question can be easily answered affirmatively by writing a translation from the new Coq 
to the old Coq. The translation is nearly the identity function, except it translates `ISet` to `Set`.

### Does the change preserve the soundness of the extraction mechanism?
I believe that Pierre Letouzey's proof works almost as it is, except that now maximal subterms of sort `Prop` or `ISet` are erased.

# Drawbacks

Someone can argue that this change makes Coq's meta-theory more complex.

# Alternatives

This CEP is partially inspired partly by the following discussion I had with the F* development team:
https://github.com/FStarLang/FStar/issues/360
It seems F* now has different mechanisms (effect systems) for proofs and erasure.


# Unresolved questions

Formally write down the typing rules for `ESet`.

