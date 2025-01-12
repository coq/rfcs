- Title: Namespaces

- Drivers: Cyril Cohen, Guillaume Melquiond, Hugo Herbelin, Enrico Tassi

- Status: Draft

----

# Summary

The main objective of namespaces is to de-correlate the name of Coq
objects from the file/module in which they are defined.

Modules have a logical status in Coq (e.g. an interface) and also
act as containers for extra-logical "stuff", like notations, type class
instances, hints, etc. Sometimes modules are just used to organize
Coq objects behind structured names, e.g. `Nat.to_Z`, but they
are at the same time too complex and not enough flexible for that purpose
(see the examples).
# Motivation by examples

Who would not like the following, raise his hand!
```coq
Check nat.
Check nat.plus.
Check nat.induction.
Print nat.__docstring__. (* eh eh, this is a joke... maybe *)
```
Note that `plus` is typically defined much later than the induction principle, maybe in another file.

One can simulate that today by juggling with modules, even if what he wants to do has little to do with modules. One would just like to write
```coq
Fixpoint nat.plus := ...
```
or
```coq
Namespae nat.
Fixpoint plus := ...
End nat.
```
to avoid repeating `nat.` in front of many definitions.

As of today, this is the drill:
```coq
Module more_nat.
 Module nat. (* the new "namespace" nat. *)
  Include nat. (* the previous namespace nat. *)
  Fixpoint plus := ...
 End nat.
End more_nat.
Import more_nat.
```
Note that, if by any chance `nat` already contains `plus` and one wants to
shadow it, then the trick is to use an auxiliary module and import it,
as we did with `more_nat`, since a module cannot contain two objects with
the same name.

Finally, remark that modules are in the *kernel*, while user facing names
are external to the kernel. The nametab component is used to give
a meaning to a user provided name `foo` in term of the more recently imported
module containing a `foo`.
What the last example struggles to do, is to use the nametab
component, but in order to do so one has to talk module's slang and work around
its design.

# Detailed design

Rules of thumb:
- each object born in one name space, which is orthogonal to the file. For
  backward compatibility, we can default to what we have today, that is
  `project.directory.file.module.name` where `project` is the root
  logpath (the one here `From project Require...`),
  `project.directory.file.module`
  is the name space and `name` is, well, the name.
- each namespace is rooted inside a project, eg `Namespace foo` means
  at least `Namespace project.foo`. This rule prevents unexpected
  collisions from different libraries.
- "opening" a name space does have no side effects, e.g. it only contains names,
  not metadata such as coercions, notations, ...

Implementation details:
- libobject is no more flat, the current "Module container" is explicit
- it may make sense to add another one for name spaces in charge of carrying all
  nametab objects
- nametab is almost what we need, but we may miss some functionalities,
  depending on how flexible "opening" a NS has to be (eg Haskells lets one
  rename on the fly, today we can't really, first of all because libstack
  objects are totally opaque, so you can't "play" them after fiddling with them)
- it makes sense to use name spaces not only for constants/inductive but also
  for other stuff that today we qualify, like modules or tactics (or even
  hind DBs could be in a namespace). So the
  "pointed to" may need to stay generic and/or be extensible by plugins.
- it makes sense to have a notion of namespace private to the file or to
  the library, which makes its contents inaccessible after its scope ends.
  In this way one can implement, for real, `Local Definition` or `#[private]`.

Syntax mockup:
- `Namespace N. ..... End N.` to open `N` and add stuff to `N`.
- `Open N` makes `N.x` available as `x` and implements shadowing (if there is
  already an `x` in scope)
- `Alias N := M` makes `N` an alias of `M`, and they are indistinguishable.
  For pretty printing purposes, `N` takes precedence on `M`
- `Alias Name N.x := M.x` is more fine grained than `Alias`

Backward compatibility (compile some module commands to smaller commands)
- Today's `Module X. ... End X.` will mean
  ```coq
  Module X.
  Namespace X.
  ...
  End X. (* the namespace *)
  End X.
  ```
- Today's `Import X.` means `Import X. Open X.`
- It may make sense to provide an anchor, to avoid this behavior.
  ```coq
  Module X.
  Namespace #Y. (* anchor Y to the project *)
  Definition a.
  ```
  which is compiled to
  ```coq
  Module X.
  Namespace X.
  Namespace #Y.
  Definition a.
  ```
  and see `a` be named `project.Y.a` instead of
  `project.directory.file.X.Y.a` which would be the result without `#`.

# Drawbacks

Today `About` almost tells you where the file in which the object is defined
can be found on the file system. If we de-correlate these concepts, then
we need a similar facility, also for UIs.
# Alternatives

These namespaces are similar to the ones of C++.
# Unresolved questions

Does this design solve the original problems of Jason Gross?
(It solves other issues, but we hijacked his CEP)
## Changes
* 2020-12-08, second draft
* 2016-07-10, first draft by Jason Gross
