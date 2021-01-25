- Title: Conditional Compilation

- Drivers: Enrico Tassi

----

# Summary

Most programming languages provide out of the box support for some form
of conditional compilation, either via a preprocessor or by integrating
in the syntax conditionally compiled blocks.

This CEP proposes a similar feature for Coq documents, but constraining it
in order to make it easy to integrate it in existing tools and not mess up
documents too much.

# Motivation

I see 3 main use cases:
- backward compatibility
- code generation
- code sharing

### Backward compatibility

One common use case for conditional compilation is to allow the same source
file to work in two incompatible versions of the language

```coq
#[if(COQ <= "8.13")] Old Command.
#[if("8.13" < COQ)] New Command.
```

This would make it always possible to write backward compatible overlays for vernac files. One could also imagine a warning the user would enable when she decides to drop compatibility with a Coq version. If any condition is true (assuming a Coq version to drop) then the warning is raised and the user removed the line or tightens the bound.

### Code generation

One problem about code generation this CEP tackles is the need to have the
generator around in order to check the document (or having the generator take
over the entire file).

```coq
#[if(derive)] Require Derive.

#generated(derive).
  (* does not need Derive in order to execute *)
  Module bar_utils.
    Definition foo := ....
    Inductive baz := ....
  End bar_utils.
#with.
  (* implemented by plugin Derive, does not even parse without Derive *)
  Derive foo, baz For bar.
#end.
```

If `derive` is not defined, the generated code is used,
otherwise the generator code is run (and its output is recorded and 
eventually replaces the old code, not in this CEP, which only provides a
place to store the output and conditionally run the generator or the
generated code).

This helps when `Derive` is a project with many dependencies one does not
want to impose to users. One cannot use `#[if(derive)] Derive...` since that
sentence would not even parse without the generator around.

Another example inspired by tactician:

```coq
Lemma foo : statement.
Proof.
#generated(fancy).
  simple_tactic_using hint1 hint2.
#with.
  fancy_tac. (* takes time, require ML stuff around *)
#end.
Qed.
```

UIs hide/fold the generated code, which should then display as:

```coq
#[if(fancy)] Require FancyTactic.

Lemma foo : statement.
Proof.
▷ fancy_tac.
Qed.
```

and unfold to the former when `▷` is clicked. 

### Code sharing

Some users raised the need for running the same fine against a different
set of imports, e.g. the architecture for which one wants to verify a file.

```coq
#[if(ARCH = "32")] From Compcert32 Require Import X.
#[if(ARCH = "64")] From Compcert64 Require Import X.
```

# Detailed design

The design makes the vernacular command interpreter aware of these attributes
and execute as no-op code which is not selected. In order to make this task
reasonably easy we impose the following limitations.

A common piece of code interpreting the conditional expressions is placed in
`lib/` and shared by all components.

## Conditions

The grammar

```
COND ::= VERSION_TEST | VARIABLE_TEST
VERSION_TEST ::= PRODUCT UBOUND | LBOUND PRODUCT | LBOUND PRODUCT UBOUND
PRODUCT ::= COQ | <ident>
UBOUND ::= < "VERSION" | <= "VERSION"
LBOUND ::= "VERSION" < | "VERSION" <=
VERSION ::= <num> | <num> . <num> | <num> . <num> . <num>
VARIABLE_TEST ::= <ident> | <ident> = <string>
```

Examples
```
"8.13" < COQ < "8.14.2" (* from 8.13+alpha to 1.14.1 *)
"8.13.0" <= COQ (* to rule out +something *)
"1.4" <= EQUATIONS (* 1.4 ... 1.4+8.13 ... 1.4+8.14 *)
FOO
FOO = "yes"
FOO = "99"
```

Version comparisons as per OPAM, but with a very simplified grammar,
only 1 to 3 digits in bounds, `+something` is epsilon.

## No nesting

`#generated.` ... `#with.` `#end.` blocks cannot nest.

This limitation is not imposed by an implementation difficulty, but rather to keep files understandable.

## No `Require` in generated code

The `Require` statement cannot occur in a generated block, in order to
simplify the job of `coqdep` (see below).

## Condition variables are immutable

Tools `coqc`, `coq*top` and `coqdep` accept `-D VAR=val` and there is no other
way to set variables used in the conditionals. There is no `Set Variable Foo.`
no `#define FOO.` and no `getenv("FOO")`.

`coq_makefile` and Dune's `coq-lang` can use their own ways to pass options to
the tools.

`coqdep` evaluates the conditional expression in from of `Require` statements
and spits the dependency if needed. This is enough to cover `coq_makefile` and
`dune`. 

## No unbalanced sections

`Section`, `Module` and `End` do not accept the `#[if(..)]` attribute and
`#generated` blocks are not allowed to leave blocks open or close too many.

This limitation is not imposed by an implementation difficulty, but rather to keep files understandable.

# Drawbacks

Conditional compilation is well known to be a tool which is easy to misuse,
leading to code hard to understand. This CEP constrains it by limiting its
application to full sentences and making the condition's value not vary across
compilation (eg no `#define FOO`).

## Long tem

UIs based on LSP or similar delegate to Coq the parsing, no problem there.
They still could provide code folding as in the example.

## Short term

UIs like PG or CoqIDE which are still "parsing" the document sentence by
sentence:
- can continue to do so, since `#generated.`, `#with.` and `#end.` look like
  sentences. We pick `#` to distinguish them from regular sentences since they
  are "directives" (like attributes).
- the vernac parser/interpreter can easily parse/execute no-ops instructions
  enclosed in false branches.

# Alternatives

Completely delegating conditional compilation to the build system is not
really an option here, since it would limit the `#generated` use case quite a
lot. Build systems can select a file, and a file is too coarse-grained in Coq.

One could see `#generated.`, `#with.` and `#end.` as floating attributes
(attributes which are not attached to a sentence, OCaml has them and they are
used, among other things, for this use case).
In that case a more adequate syntax inspired by Rust could be
`#![generated(...)].` (I would keep the `.` to help UIs, see above).

# Unresolved questions

This CEP does not cover how to fill-in `#generated.` blocks, e.g. tactician
writes a message the user has to copy-paste. This process can be simplified
if the build system supports promotion of updated source files.
