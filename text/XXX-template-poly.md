- Title: Template poly redesign using sort poly

- Drivers: Gaëtan Gilbert

----

# Summary

Template poly currently provides 2 features in 1: implicit universe
instances and adhoc sort polymorphism handling only Prop/Type.

We want to make it less adhoc and base it on more well understood
systems such as regular sort poly.

This should also allow to stop producing useless constraints (like u <= prod.u0)
when a template inductive is fully applied.

# Detailed design

## Declaration

An inductive may be declared to use implicit univ instances when:

- it is non mutual (nested is ok) (this rule is in master but maybe we will remove it)
- its univ declaration has only univ variables unbounded from below
- its univ declaration has at most 1 qvar variable.
  If it has one, it is the quality of the output sort.
- each parameter type either does not mention the bound univs (CLOSED),
  or is of the form `forall args, Type@{q|u}` (BINDING)
  where the `args` types do not mention the bound univs and qvar,
  `q` is either Type or the unique bound qvar,
  and `u` is either constant or one of the bound univ levels.
- the indices types and constructor types do not mention the bound univs and qvar.

NB: the qvar (if there is one) must be "above prop" so should not appear in relevance marks
(it's always `Relevant`) so eg
~~~
Inductive prod@{q|u v|} (A:Type@{q|u}) (B:Type@{q|v}) : Type@{q|max(u,v)} :=
  pair : A -> B -> prod A B.
~~~
is accepted and the arrow `A -> ...` has relevance mark `Relevant` not `RelevanceVar q`.
See also "Note on nested templates and above Prop" below.

NB': these rules mean the bound univs have `Irrelevant` cumulativity
variance (not to be confused with relevance).

## Instantiation

Given a default instance (with default quality `Type`),
a (possibly partial) application of the inductive or its constructors
to parameters is handled by:

For each BINDING parameter, we obtain the universe level (which may be algebraic)
and quality from the actual passed parameter if available, otherwise from the default instance.
The quality (if bound) must be "above Prop".

If the quality is bound (ie non constant), it is assigned to the max of the inferred qualities
(this is possible because of the "above Prop" assumption).
For each bound univ level, it is assigned to the max of the corresponding obtained levels.

This gives the implicit instance.
We then check any constraints on the variables and do regular typechecking.

## Subject reduction

### Inductive

Consider `(fun X:Type@{u} => I X) (P:Type@{v}) (Q:Type@{w})`
with `I@{i j | csts(i,j)} (p:Type@{i}) (q:Type@{j}) : Type@{f(i,j)}`
and default instance `{i0 j0}` (which must verify `|= csts(i0, j0)`).

For the beta redex to be welltyped, we must have
- `|= csts(u, j0)`
- `|= v <= u`
- `|= w <= j0`
and it has type `Type@{f(u,j0)}`.

The reduced value is `I P Q`. For subject reduction to hold we need
- `|= csts(v,w)`
- `Type@{f(v,w)} <= Type@{f(u,j0)}`

The "unbounded from below" rule means that the `csts(i,j)` are either
- not mentioning `i` and `j` -> trivially hold from `|= csts(i0,j0)`
  (or we extrude such constant constraints at declaration time)
- `i <= c` with `c` constant, in which case we have `|= u <= c` from `|= csts(u,j0)` and we have `|= v <= u`
  so `|= v <= u <= c`
- `j <= c` with `c` constant, in which case we have `|= j0 <= c` and from `|= csts(u,j0)` and we have `|= w <= j0`
  so `|= w <= j0 <= c`

Meanwhile `f` must be built using the `max` and `+1` operators so is monotonous.
Since `|= v <= u` and `|= w <= j0` we have `|= f(v,w) <= f(u,j0)`.

The fully general proof of subject reduction should work with the same style of reasoning.

#### PROBLEM

Consider `(fun X:Type@{u} => I X) (P:Type@{v})`
with `I@{i | csts(i)} (p:Type@{i}) (q:Type@{i}) : Type@{f(i)}`
and default instance `{i0}` (which must verify `|= csts(i0)`).

It has type `forall Type@{max(u,i0)}, Type@{f(u,i0)}` assuming
- `|= csts(u, j0)`
- `|= v <= u`

The reduced value `I P` has type `forall Type@{max(v,i0)}, Type@{f(v,i0)}`
which is NOT comparable to `forall Type@{max(u,i0)}, Type@{f(u,i0)}`!!

### Constructor

Consider `(fun X:Type@{u} => C X) (P:Type@{v}) (Q:Type@{w})`
with `I@{i j | csts(i,j)} (p:Type@{i}) (q:Type@{j}) : Type@{f(i,j)}`
and `C : forall p q, I p q`
and default instance `{i0 j0}` (which must verify `|= csts(i0, j0)`).

For the beta redex to be welltyped, we must have
- `|= csts(u, j0)`
- `|= v <= u`
- `|= w <= j0`
and it has type `I P Q`.

The reduced value is `C P Q`.For subject reduction to hold we need
- `|= csts(v,w)` (same reasoning as above)
- `I P Q <= I P Q` (trivial)

#### PROBLEM

Consider `(fun X:Type@{u} => C X) (P:Type@{v})`
with `I@{i | csts(i)} (p:Type@{i}) (q:Type@{i}) : Type@{f(i)}`
and `C : forall p q, I p q`
and default instance `{i0}` (which must verify `|= csts(i0)`).

It has type `forall Q:Type@{max(u,i0)}, I P Q` assuming
- `|= csts(u, j0)`
- `|= v <= u`

The reduced value `C P` has type `forall Q:Type@{max(v,i0)}, I P Q`
which is NOT comparable to `forall Q:Type@{max(u,i0)}, I P Q`!!

## Note on nested templates and above Prop

Consider
~~~
Inductive double@{q|u|} (A:Type@{q|u}) : Type@{q|u}
  := Double : prod A A -> double A.
~~~
When checking `prod A A` we must ensure that `q` is above Prop,
and if we had `prod A B` with A and B at different quality variables
we would have to unify them (in elaboration) or error (in kernel).

This means the kernel must handle "above prop" bound qvars.

## Note on squashing

In master template inductives are never squashed, but this does not
seem to be actually necessary (cf "subject reduction" above).

In master we typecheck against the default instance, so a squashed template would be useless,
but when we don't a type like `Inductive Squash (A:Type) : Prop := squash (_:A).`
could be usefully template (in master it behaves exactly the same as if mnomorphic).

## Some examples that should work

~~~coq
(* bound qvar without univ poly enabled -> must be template poly
   q above Prop
   non-output qvars not allowed *)
Inductive prod@{q|a b|} (A:Type@{q|a}) (B:Type@{q|b}) : Type@{q|max(a,b)} := ...

(* if the user wrote *)
Inductive prod@{a b} (A:Type@{a}) (B:Type@{b}) : Type@{max(a,b)} := ...
(* it should produce the same as above
   (this is the current way to write prod with explicit univs in master,
   don't want to break backwards compat (or do we?)) *)

(* must avoid inferring q:=Prop
   the rule is if all arguments are Prop then output is Prop otherwise it's Type *)
Check prod True nat.

(* no qvar allowed here (output is always Type) *)
Inductive option (A:Type@{a}) : Type@{max(Set,a)} := ...

(* equivalently(?): *)
Inductive option (A:Type@{a}) : Type@{a} := ...
~~~

# Drawbacks

The conditions to be (new) template polymorphic may be more
restrictive than the current ones, but probably not in a way that
anyone relies on (we hope).

We may consider ways to relax it later.

# Alternatives

Keep special handling of "floating" (not telated to Set) univ variables?

# Unresolved questions

??
