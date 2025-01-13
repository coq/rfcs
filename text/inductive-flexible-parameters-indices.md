- Title: Flexible combination of parameters and indices in inductive types

- Drivers: Hugo

----

# Summary

Inductive types have three kinds of arguments:
- uniform parameters
- recursively non-uniform parameters
- indices (also known as real arguments)

These arguments have to be given by blocks in this order. The CEP proposes to unify the syntax of parameters and indices and to let Coq compute automatically the status of an argument, accepting combining arguments of different status in any order.

This has the following advantages:
- the `:` in the type of inductive type behaves like the `:` in declarations, that is, `(x : A) :` is the same as `: forall x :
A,`
- no more constraint on giving parameters before the indices and the uniform parameters before the recursively non-uniform ones
- different implicit status can be given at definition time to parameters depending on whether it is the parameter of a constructor or of the inductive type
- no more difference of status between an argument given as a parameter and the same kind of argument given as a index
- not all inductive types of a block of mutual inductive types need to have exactly the same parameters

# Basic proposal

The basic proposed changes are:
- the context of parameters currently given next to the inductive type can optionally be replicated in the type of constructors
- if the context is replicated in all constructors, it can be dropped for the inductive type itself
- any argument declared after the `:` in the type of the inductive type can equivalently be moved before the `:` and the parameter/index status is instead computed automatically

# Examples

The following syntaxes all define the same `list A` where `A` is a parameter:
```
Inductive list (A : Type) : Type :=
| nil : list A
| cons : A -> list A -> list A.

Inductive list : forall (A : Type), Type :=
| nil : list A
| cons : A -> list A -> list A.

Inductive list : forall (A : Type), Type :=
| nil : forall A, list A
| cons : forall A, A -> list A -> list A.

Inductive list : forall (A : Type), Type :=
| nil : list A
| cons : forall A, A -> list A -> list A.

Inductive list : forall (A : Type), Type :=
| nil : forall A, list A
| cons : A -> list A -> list A.

Inductive list (A : Type) : Type :=
| nil : list A
| cons : A -> list A -> list A.

Inductive list (A : Type) : Type :=
| nil : forall A, list A
| cons : forall A, A -> list A -> list A.

Inductive list (A : Type) : Type :=
| nil : list A
| cons : forall A, A -> list A -> list A.

Inductive list (A : Type) : Type :=
| nil : forall A, list A
| cons : A -> list A -> list A.

Inductive list :=
| nil : forall A, list A
| cons : forall A, A -> list A -> list A.
```

The following definitions define the same `eq A x y` where `A` and `x` are parameters:
```
Inductive eq (A : Type) (x : A) (y : A) : Type := refl : eq A x x.
Inductive eq (A : Type) (x : A) (y : A) : Type := refl : forall A x, eq A x x.
Inductive eq (A : Type) (x : A) : forall (y : A), Type := refl : eq A x x.
Inductive eq (A : Type) (x : A) : forall (y : A), Type := refl : forall A x, eq A x x.
Inductive eq : forall (A : Type) (x : A) (y : A), Type := refl : eq A x x.
Inductive eq : forall (A : Type) (x : A) (y : A), Type := refl : forall A x, eq A x x.
Inductive eq := refl : forall A (x : A), eq A x x.
```

We expect for simplificity that the parameters given explicitly in the type of a constructor are given in the same order as the order these arguments are applied in the type. E.g., the following are accepted:
```
Inductive I := C : forall x y : nat, I x y.
Inductive I (m:nat) (n:nat) := C : forall x y : nat, I x y.
```
but the following is rejected:
```
Inductive I := C : forall x y : nat, I y x.
```

For simplicity/clarity, we also require that when parameters are replicated in the type of a constructor, exactly all of them are replicated. E.g., the following would be rejected:
```
Inductive I (x:nat) (y:nat) := C : forall x : nat, I x y.
```

For clarity, we also expect that when the global context is not ommitted, it is not partial. E.g., the following would be rejected:
```
Inductive I (x:nat) := C : forall x y : nat, I x y.
```

For clarity of scoping, one may also continue to impose that a variable occurring after the `:` cannot be used in the type of constructors.

# Computing the status of arguments of inductive type

The counterpart of the freedom of syntax is that the classification between parameters, non-uniform parameters and indices needs to be computed.

Let's call inductive context the context occurring on the right of the inductive name. The status of an argument is computed as follows:

- if the argument of an inductive type is a variable in the conclusion of the type of all constructors of the inductive type, it is considered to be a parameter, independently of the question of whether it is quantified in the global context or quantified locally to the constructor (the variable has also to be fresh among the previous arguments already set as parameters)
- a parameter which occurs as the same variable at the position of this parameter in all recursive occurrences of the inductive type, it is considered a uniform parameter for this type and otherwise a recursively non-uniform parameter

The constructors depends by construction of the parameters. In theory, the parameters could be given in any order in the type of constructors, but it is probably simpler, at least for compatibility of the implementation, that they are given in the same order as in the inductive type, and that they come as prefix of the type of the constructor.

# Internal representation

The information about the number of parameters and number of uniform parameters is replaced by a list of status. It is now local to each type of a block of mutual types. The type of constructors are unchanged except that their parameters are not necessarily the same in each block of mutual inductive types.

The parameters to apply to a constructor are not anymore a prefix of the arguments of their type but a selection of them that is computed with the help of the list of status.

# Related issues

- [inconsistency](https://stackoverflow.com/questions/56143587/why-is-my-definition-not-allowed-because-of-strict-positivity) between indices behaving as parameters and parameters 
