- Title: Incremental `Include`

- Drivers: Gordon Stewart (gordon@bedrocksystems.com) and Gregory Malecha (gregory@bedrocksystems.com)

----

# Summary

This CEP proposes an extension to the Coq module system to support incremental `Include`s of
terms (constants, inductives) defined in `Module Type`s, with concrete syntax `Use x`,
`Use x From MODULE_TYPE`, and `UseAll From MODULE_TYPE`.

It additionally proposes the `Provide` command, which eagerly checks -- at
definition time rather than at module close time -- that definitions implementing
parameters/axioms of a module type interface have the correct types.

It additionally proposes a `Prove` mechanism to avoid re-stating the types
of axioms when implementing an interface.

Each of the proposed features -- `Use`, `Provide`, and `Prove` -- is independently
useful and could be independently adopted.

Concretely:
```
Module Type INTERFACE.
  Axiom N : nat.
  Definition n := N + 1.

  Axiom S : nat.
  Definition s := S + 1.

  Axiom N_S : n = S + 1.

  Definition x := N + s.
  Definition y := S + x.
End INTERFACE.

Module Implementation : INTERFACE.
  Provide Definition N := 0. (*Eagerly check that `N`'s type matches the `INTERFACE`.*)
  Use n. (*Include `Definition n := N + 1`, with `N` replaced by the `N` defined one line above.*)

  Provide Definition S := 0.
  Use s.

  Prove N_S. (*Open `Lemma N_S : n = S + 1`.*)
  Proof. reflexivity. Qed.

  Use x.
  Use y from INTERFACE. (*Specify the Module Type to disambiguate, if necessary.*)
End Implementation.
```

The following `ImplementationFail` illustrates some failure scenarios:
```
Module ImplementationFail : INTERFACE.
  (*1. `Provide` eagerly checks that types match.*)
  Fail Provide Definition N := false.

  Provide Definition N := 0.

  (*2. You can't `Provide` a name not in the interface.*)
  Fail Provide Definition Q := 3.

  (*3a. You can't use a definition until its dependencies (here `N` and `s`) have been resolved.*)
  Fail Use x.

  Provide Definition S := 0.

  Prove N_S.
  Proof. reflexivity. Qed.

  (*3b. You can't use a definition until its dependencies have been resolved (*`x` has not yet been defined).*)
  Fail Use y.

  Use x.

  (*4. You can't `Use` a definition that's not defined in `INTERFACE`.*)
  Fail Use z.

  Use y from INTERFACE.
End ImplementationFail.
```

# Motivation

The point of `Use` and `Prove` is to avoid code duplication.

Consider this same interface again:
```
Module Type INTERFACE.
  Axiom N : nat.
  Definition n := N + 1.

  Axiom S : nat.
  Definition s := S + 1.

  Axiom N_S : n = S + 1.

  Definition x := N + s.
  Definition y := S + x.
End INTERFACE.
```
To implement it in current Coq, one must (1) copy definitions (`n`,
 `s`, `x`, `y`) or (2) factor the interface into multiple module types
and use `Include`.
Factoring can be very heavy weight, especially when axioms and definitions
are naturally written in an order that intermingles them.
In both cases, the type of lemma `N_S : n = S + 1` must be
declared twice: in the module type and at the proof site.

### Copy Definitions

Copy--pasting definitions works, but isn't ideal for code maintenance:
```
Module Implementation0 : INTERFACE0.
  Definition N := 1.
  Definition n := N + 1.
  Definition S := 1.
  Definition s := S + 1.
  Lemma N_S : n = S + 1.
  Proof. reflexivity. Qed.
  Definition x := N + s.
  Definition y := S + x.
End Implementation0.
```
[NOTE: The definitions `x`, `y`, etc., above are artificially short. The copy--paste
overhead is typically much larger, because in real code there are either many more definitions
in the module type, or the definitions are more complicated, or both.]

### Factor Multiple Module Types

Alternatively, one can factor the interface into multiple module types:
```
Module Type N_TY.
  Axiom N : nat.
End N_TY.

Module Type N_Derived (Import N : N_TY).
  Definition n := N + 1.
End N_Derived.

Module Type S_TY.
  Axiom S : nat.
End S_TY.

Module Type NS_Derived (Import N : N_TY)(Import NS : N_Derived N) (Import S : S_TY).
  Definition s := S + 1.
  Definition x := N + s.
  Definition y := S + x.
End NS_Derived.
```
The implementation then alternates definitions of axioms with module type `Include`s:
```
Module FactoredImplementation : INTERFACE.
  Definition N := 1.
  Include N_Derived.

  Definition S := 1.
  Include NS_Derived.

  Lemma N_S : n = S + 1.
  Proof. reflexivity. Qed.
End FactoredImplementation.
```
The extra bureaucratic overhead of the separate module types is less than ideal.
Moreover, you need a new module type every time you alternate between concrete
definitions and parameters/axioms.

## `Use`ing

As we advertised in the **Summary**, `Use` lets us do the module type factoring implicitly, by writing the
implementation as:
```
Module Implementation : INTERFACE.
  Provide Definition N := 0.
  Use n.

  Provide Definition S := 0.
  Use s.

  Prove N_S. (*Opens `Lemma N_S : n = S + 1`.*)
  Proof. reflexivity. Qed.

  Use x y.
End Implementation.
```
This implementation avoids code duplication and the extra bureaucratic overhead of the
module type refactoring.

Using the proposed `UseAll From ...` syntax, the implementation could be further refined to:
```
Module Implementation : INTERFACE.
  Provide Definition N := 0.
  Provide Definition S := 0.
  UseAll From INTERFACE.
  Prove N_S.
  Proof. reflexivity. Qed.
End Implementation.
```

# Detailed design

We propose three new vernacular commands:

1. `Use` (id)+

In the context of a module implementation `ModuleName : ModuleTypeName`, include in lexical order the
definitions of `(id)+` from `ModuleTypeName`. The definitions may be open wrt. parameters
that appeared lexically before `id` in `ModuleTypeName`; substitute for these holes
the definitions/parameters with those names from the module currently being defined (`ModuleName`).
Fail if definitions are missing.

In the case of multiple module type ascriptions:
```
Module M <: M1 <: M2.
  Fail Use x.
End M.
```
there may be multiple syntactically distinct definitions of `x` in scope. In this case, `Use x` fails.
Users can disambiguate using the `Use (id)+ From ModuleTypeName` syntax.

&nbsp;&nbsp;&nbsp;1a. `Use` (id)+ `From` ModuleTypeName

Like `Use` (id)+, but include the definitions from the explicit module type `ModuleTypeName`.

&nbsp;&nbsp;&nbsp;1b. `UseAll` `From` ModuleTypeName

`Use` all the definitions (inductives/records and terms with bodies but not axioms) from `ModuleTypeName`.

&nbsp;&nbsp;&nbsp;1c. `UseAll` `Available` `From` ModuleTypeName

Like `UseAll`, but include only those definitions that are fully resolvable at this point in the module definition.

2. `Provide` Definition x : ty := body.

In the context of a module implementation `ModuleName : ModuleTypeName`,
define `x : ty := body` but additionally check that `ty` unifies with the type assigned to `x`
by `ModuleTypeName`, raising an error if not.

3. `Prove` id.

In the context of a module implementation `ModuleName : ModuleTypeName`,
open `Lemma id : type-of-id`, where `type-of-id` is the type assigned to axiom/parameter `id`
in `ModuleTypeName`.

&nbsp;&nbsp;&nbsp;3a. `Prove` `Instance` id.

Like `Prove`, but open an instance proof instead of a lemma.

# Related discussions

In this related issue: https://github.com/coq/coq/issues/8317, Jason Gross asks about the behavior
of a `Use`-like feature in the context of multiple module type ascriptions:
```
Definition mynat := nat.
Module Type MOD1.
  Definition t := nat.
End MOD1.

Module Type MOD2.
  Definition t := mynat.
End MOD2.

Module Mod1 <: MOD1 <: MOD2.
  Use t.
End Mod1.
Print Mod1.t. (* nat, or mynat? *)
```
(We've updated this example slightly to use the `Use` syntax proposed in this CEP.)

The `Use t` declaration is ambiguous. We propose that it should fail above, such that users
are instead required to disambiguate:

```
Module Mod1 <: MOD1 <: MOD2.
  Use t From MOD1.
End Mod1.
Print Mod1.t. (* nat *)
```

from

```
Module Mod1 <: MOD1 <: MOD2.
  Use t From MOD2.
End Mod1.
Print Mod1.t. (* mynat *)
```

* Related Coq-Club discussions:
  * https://sympa.inria.fr/sympa/arc/coq-club/2018-08/msg00035.html
  * https://sympa.inria.fr/sympa/arc/coq-club/2017-10/msg00036.html

* Other discussion:
  * https://github.com/mit-plv/bedrock2/commit/20d5fac26f#r29468057
