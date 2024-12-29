- Title: Giving access in `match` to the expansion of the term being matched via an alias in order to support more fixpoints

- Drivers: Hugo Herbelin

----

# Summary

We propose to extend the context of each branch of `match` in the Calculus of Inductive Constructions with an alias referring to the constructor of the branch, so that more fixpoints are available in a "natural" way.

# Motivation

When writing fixpoints in inductive types with indices, there is a standard conflict between referring to the expansion of a variable being matched (so that its type corresponds to the type in the branch) or referring to the variable so that it is compatible with the guard when the fixpoint is later used in another fixpoint. A typical example (even without indices) is `Nat.sub`:
```coq
Fixpoint sub (n m : nat) {struct n} : nat :=
  match n, m with
  | S k, S l => sub k l
  | _, _ => n
  end
```
where it is important that `n` is not expanded.

The proposal is to resolve this conflict by giving an explicit name to the expansion of the term being matched in a branch so that the guard condition knows that it is a decreasing argument and not a constructor disconnected from the term being matched. For instance, `Nat.sub` would be written:
```coq
fix sub (n m : nat) {struct n} : nat :=
  match n, m with
  | S k, S l => sub k l
  | _ as n', _ => n'
  end
```

In the case of an inductive type with no indices, this does not provide much, but the situation changes for type families. For instance, imagine we want to apply parametricity to `Nat.sub`. That would give:
```coq
Fixpoint is_sub (n : nat) (Pn : is_nat n) (m : nat) (Pm : is_nat m) : is_nat (Nat.sub n m) :=
  match Pn in (is_nat n) return is_nat (Nat.sub n m) with
  | is_O => is_O
  | is_S x P_ =>
      match
        Pm in (is_nat m0)
        return is_nat (match m0 with 0 => S x | S l => Nat.sub x l end)
      with
      | is_O => is_S x P_
      | is_S l Pl => is_sub x P_ l Pl
      end
  end.
```
where, for typing, we have to expand `Pn` into a constructor in the right-hand side.

With the proposed extension, we would write:
```coq
Fixpoint is_sub (n : nat) (Pn : is_nat n) (m : nat) (Pm : is_nat m) : is_nat (Nat.sub n m) :=
  match Pn in (is_nat n) return is_nat (Nat.sub n m) with
  | is_O as p => p
  | is_S x P_ as p =>
      match
        Pm in (is_nat m0)
        return is_nat (match m0 with 0 => S x | S l => Nat.sub x l end)
      with
      | is_O => p
      | is_S l Pl => is_sub x P_ l Pl
      end
  end.
```
with `p` of the same type as the constructor but recognized as a subterm for the guard condition.

Note: for the record, the way `sub` is defined allows the guard to define `gcd` with a recursive call for `S a` on `sub a b` for some `b`. The way `is_sub` is defined (without the `as`) prevents the guard to define the parametricity for `gcd` with the same criterion. To define `gcd` using a recursor, a commutative cut would be needed in the theory, or `gcd` passed as a continuation to `sub` (see LPCIC/coq-elpi#493 for details).

# Details about the design

For an inductive type `[... Ii : Δi -> si := {... Cij : Ωij -> Ii δij ...} ...]`, the proposed new typing rule is:
```
  Γ ⊢ t : Ii δ     Γ, y : Δi, x : Ii y ⊢ P : s    ... Γ, z : Ωij, w : Ii δij  ⊢ uij : P[y,x:=δij,Cij z] ...
  ————————————————————————————————————————————————————————————————————————————————————————————————————
                Γ ⊢ (match t as x in Ii y return P with ... Cij z as w => uij ... end) : P δ
```
and new reduction rule is:
```
match (Cij u) as x in Ii y return P with ... Cij z as w => uij ... end
 →
uij[z,w:=u,Cij u]
```
while, for the guard condition, the new variable `w` is considered of the size of `t`.

# Alternative solution for the guard condition

Instead of adding an `as` clause, the guard condition could also detect subterms of the right-hand side that are syntactically equal (or even convertible) to the constructor and consider them automatically of the same size as the term being matched (proposal of G.M.). Then, the user would not have to care about whether it expands or not the right-hand side.

# Generality of the design

The proposal is useful for the guard condition to better scale to recursion on inductive families, but it makes sense also outside the context of the guard.

Consider a recursor such as:
```
nat_rect : forall P : nat -> Type, P 0 -> (forall n : nat, P n -> P (S n)) -> forall n : nat, P n
```
It is easy to define a function which e.g. turns 0 into 1 and otherwise is the identity:
```
Definition f n := nat_rect (fun _ => nat) 1 (fun _ _ => n) n.
```

Consider now the recursor for the property of being a `nat`:
```
Inductive is_nat : nat -> Type :=
    is_O : is_nat 0
  | is_S : forall H : nat, is_nat H -> is_nat (S H).
(*
is_nat_rect : forall P : forall s1 : nat, is_nat s1 -> Type,
       P 0 is_O ->
       (forall (n : nat) (P_ : is_nat n),
        P n P_ -> P (S n) (is_S n P_)) ->
       forall (s1 : nat) (i : is_nat s1), P s1 i
*)
```
We can't do the same with the ordinary recursor:
```
Fail Definition is_f n (is_n : is_nat n) :=
  is_nat_rect (fun n _ => is_nat (f n))
    (is_S 0 is_O)
    (fun _ _ _ => is_n) is_n.
```
To do it, one would need a recursor with an `as` clause, that is a recursor of this type:
```
is_nat_rect_with_alias : forall P : forall s1 : nat, is_nat s1 -> Type,
       (P 0 is_O -> P 0 is_O) ->
       (forall (n : nat) (P_ : is_nat n),
        P n P_ -> P (S n) (is_S n P_) -> P (S n) (is_S n P_)) ->
       forall (s1 : nat) (i : is_nat s1), P s1 i
```
so that we can write:
```
Definition is_f n (is_n : is_nat n) :=
  is_nat_rect_with_alias (fun n _ => is_nat (f n))
    (fun _ => is_S 0 is_O)
    (fun _ _ _ is_n => is_n) n is_n.
```
