- Title: Typeclasses

- Drivers: Cyril Cohen

- Status: Draft

----

# Wishes

- A cut tag between the (class) arguments of instances, with the same behavior as the prolog cut (as opposed to the Hint Cut). Syntax TBD
E.g. (using "cut" as a placeholder for the future syntax)
```coq
Instance A `{C x} "cut" `{C' x y} : C (f x y) := ...
```
If a solution for (C x) is found, then no backtrack is allowed if a search for (C' x y) fails.

- Control typeclass opacity by class and not globally. The opacity/transparency constraint should be stored in a hint database in order for it to parametrize typeclasses eauto with ...

