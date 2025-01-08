- Title: Extending and unifying the support for giving arguments by name or index

- Drivers: Hugo Herbelin

----

# Summary and motivation

Depending on situations, there are three ways to provide arguments to a function:
- the standard `term` form, as part of an ordered list of arguments
- by name as in `(ident := term)`, and sometimes `(ident binders := term)`
- by index as in `(int := term)`

The current situations where it is possible to give arguments by name or index are:
- to explicitly supply instances for implicit arguments, as in `id (A:=nat) 0`
- in tactics, with the `term with bindings` construction, where bindings are:
  - either a list of terms (`one_term`), when the number of arguments is exactly the expected number of missing dependent arguments
  - or by name of dependent argument as in `(ident := term)`, or index of non-dependent argument as in `(int := term)`

The objective is to extend/relax/smoothen/fix the different ways to provide argument by name or index.

# Detailed objectives

Ability to give arguments by name or index also for global references prefixed by `@`, as an alternative way to refer to an argument (similar to OCaml's labels or Ada's named arguments, but where using a label is not the exclusive way to give the argument). This means we have several ways to give an argument and thus requires to state rules:
- about when a named/indexed argument is taken precedence on the position:
  - taking first into account all named or indexed arguments then fill the hole with the other arguments in order?
  - or treat arguments in order and fails if a named or indexed argument comes too late in the sequence?
- about what to do when an intermediate argument is missing: re-abstracting over the missing arguments or failing?

Addressing some floppy aspects of the current implementation:
- it is possible to refer to an anonymous implicit argument with syntax `(arg_n:=term)` when `n` is the absolute index of the argument among all arguments; this is liable to collide with the legal argument name `arg_n`, so either `arg_n` names should be reserved or another syntax should be found
- the `with` clause of tactics applies to any term, and in particular adds implicit arguments if one writes a global reference alone; this is unintuitive and leads to using two ways to provide arguments, as in `foo (A:=bar) with (A':=bar')` (see e.g. issue coq/coq#6029) 
- the `with` clause of tactics infers the possible names and dependent arguments dynamically depending on the names used in the surrounding context; this is fragile; `with` should typically restricted to global references and used the names shown by `About`
- do we want expressions such as `(fun A => A) (A:=nat)` to be accepted (it currently fails with a confusing "Wrong argument name: A")?

Unifying the syntax:
- possibly generalizing the syntax to `(ident binder := term)`, as done e.g. in tactic `set`: it is convenient but with the risk of becoming heavy and not always readable (consider `(id := fun a => term)` vs `(id a := term)`)?
- the question of referring to non-dependent arguments (see e.g. PR coq/coq#11099)
  - decide whether the best criterion is by:
    - absolute index
    - or index among non-dependent parameter (as in the `with` clause)
    - or is after all a bad idea: we should force instead to give a name even to non-dependent arguments, and to use this name
  - if by index, find a less hackish syntax to refer to non-dependent arguments; one I have in mind is `(Nth binders := term)`, where `Nth` is either `1st`, `2nd`, `3rd`, etc.; another one could be `(#n binders := term)`; propositions need to be made
- possibly move the `with` construction in the syntax of term

More generally, does anyone know what other systems do?
