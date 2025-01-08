# Proof Modes

This CEP is about the infrastructure needed in order to have multiple 
proof modes coexisting in a modular way (i.e. without syntactic or semantic conflicts)

## Status quo (as per Coq 8.6)

1. Tactic names are kernel names, i.e. unique, and can be accessed this way

    ```
    foo.tac arg1.
    bar.tac arg1.
    ```
    
1. Linking (not even requiring) a plugin loads into the *lexer* the tokens 
   specific to the grammar implemented by the plugin
   
1. Linking (not even requiring) a plugin loads into the *parser* grammar the rules
   specific to the plugin 
   
1. Macros in `tacextend.mlp` like `TACTIC EXTEND` plug the new grammar rule into the
   *ltac* grammar `tactic` (non-terminal) symbol.  Note that tacticals are described by
   the `tactic_expr` non-terminal, hence the use of `GEXTEND` in some plugins.
   
   - Today *ltac* is a plugin, and `TACTIC EXTEND` is tied to this particular plugin.
     Tactic arguments as in `ARGUMENT EXTEND` are also part of this plugin.
   
   - `decl_mode`, for example, cannot use the `* EXTEND` mechanism because it needs to
     plug in its own non-terminal (to hide ltac).

1. So called "proof mode" is implemented by making the main "tactic" non-terminal
   mutable and having specific commands assign such non-terminal to the desired one.
   The current proof mode is not part of the `Summary` and is both stored in `Proof_global`
   and `Stm` that do their best to ensure parsing happens in a state where such non-terminal
   holds the expected value.

## Scenario not supported by the status quo

1. using `TACTIC EXTEND` for a non-terminal that is not ltac specific.

2. loading two proof languages with syntactic conflicts at the *grammar* level
   and using both.
   
    In `mode1` we have `Tactic Notation "foo" "bar" constr(x) := idtac x.`
    In `mode2` we have a conflicting grammar `Tactic Notation "foo" constr(x) := idtac x.`
    If we load both
    
    ```    
    Goal nat -> True.
    intro bar.
    foo bar bar.  (* works *)
    foo bar. (* does not parse *)
    ```
    
    Of course the grammar is ambiguous, but if the two modes were not blindly mixed
    one could imagine one of the following to work:
    
    `mode2:: rewrite ->H => [].`
    `mode1.(foo => bar).`
    
    ```
    begin mode2.
      foo bar.
    end.
    ```
    
    Where `mode2::` and `begin mode2` are understood by the parsing engine.

2. loading two proof languages with syntactic conflicts at the *lexer* level
   and using both.

   In `mode1` we have `Tactic Notation "foo" "=>" ">" := idtac 1.`
   and in `mode2` we have `Tactic Notation "bar" "=>>" := idtac 2.`
   If only `mode1` is loaded the sentence `foo =>>` parses fine,
   while it stops working when the grammar rule in `mode2` declares
   `=>>` as a single token.
   
   Another similar problem used to arise between IDENT if one declares a `Notation "'foo'"` for a
   term, fixed in 982460743a54ecfab1d601ba930d61c04972d17a 

1. Decouple the proof language from the `.vo` file.  Since the `.vo` contains the `Libobject`
   action corresponding to `Declare ML Module` one gets both (even by transitivity).  It should be
   possible to access a theorem without caring about the proof mode used to prove it.


## Proposal

Each proof mode installs all its grammar in a dedicated non-terminal `mode-root`.
Such non-terminal is installed as the main entry (today the one for `tactic-expr`)
in one of the following ways:
- `Set Default Proof Mode "mode1".` (TODO: disable inside proof?)
- `begin mode1. .... end.` within a proof.
- `mode1:: ... .` within a proof, changes mode for 1 statement only
- `Proof .... mode mode1 ... using ... .` (Useful?)

Also fix the token gluing problem (proposal https://coq.inria.fr/bugs/show_bug.cgi?id=4867 )
to make it simpler to reuse non-terminals of other modes inside the grammar of a
mode if one likes to.

## Bugs

In this case we don't get why the second rule does not mask the first one.

```
Notation "! 'bar' x " := 1 (at level 2).
Notation "! bar x" := x (at level 2).

Goal ! bar 2 = 3.
```

Also (this time it prints 'bar' is a keyword).

```
Notation "'bar' ! x " := 1 (at level 2).
Notation "bar ! x" := x (at level 2).

Goal bar ! 2 = 3.
```
