- Title: Fine-tuning the language to control reduction / uniformization of reduction behaviors
- Driver: Hugo
- Status: Draft

## Summary and motivation

- There are a few discrepancies between reduction strategies
- While doing proofs, one often needs ability for more fine-tuning of the reduction

This proposal is about erasing some discrepancies and about developing a richer language of commands to control reduction.

## Examples of discrepancies in reduction

```
Eval cbn delta in let x := 0 in x. (* x unfolded *)
Eval cbv delta in let x := 0 in x. (* x not unfolded *)
Eval lazy delta in let x := 0 in x. (* x not unfolded *)
```

## Proposal

### Extensions of the language for fine-tuning reduction

- To add a variant of ζ restricted to a list of name, so that e.g. `eval zeta [a] in let b:=1 in let a:=0 in a+b` contracts only `b`

- To add a variant of δ for local definitions restricted to a list of name, so that e.g. `eval delta [b] in let b:=1 in let a:=0 in a+b` unfolds only `b` (and keeps the `let`)

- To have this variant support occurrences, e.g. `eval delta [a@1] in let b:=1 in (let a:=0 in a+b, let a:=1 in a+b)` unfolds only the first `a` (and keeps the let-in)

- To delimit a subexpression to evaluate by a context, as in `eval lazy subterm X of pattern [?X + ?] in ((2 - 1) + 3) * 4`

- To restrict a reduction to weak-head reduction, as in `eval whd lazy subterm X of pattern [?X + ?] in ((2 + (x + (2 * 2))) + 3) * 4`

- To restrict a reduction to a number of steps of recursion as in `eval whd lazy fix [3] subterm X of pattern [?X + ?] in ((4 + (x + (2 * 2))) + 3) * 4` which would give `((1 + S (S (S (x + (2 * 2))))) + 3) * 4`

Note that the concrete syntax is pretty speculative yet.

### Proposal for δ on local definitions

To always unfold local definitions whenever δ is expected.

## Related issues

To be eventually able to unfold `fix` and `cofix` into their named variant when such a name exists.

