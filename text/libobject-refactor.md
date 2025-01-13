- Title: Libobject metadata and state redesign

- Driver: Emilio J. Gallego Arias

- Status: Draft, (partial) PR available, PR tested

---

# Summary and Objectives

Coq's module system is (mainly) implemented by an abstract notion of
"object" (`Libobject.t`), which corresponds to an existential type `t`
which has to be (implicitly) `Marshall`able, plus a set of operations
`Libobject.object_declaration` which specify the effects the object
will perform when affected by several operations such as `Require`,
`Import`, etc... in a fully imperative (that is to say `unit -> unit`)
way.

Examples of such objects are declarations, implicit arguments, modules
themselves, implicit arguments and quantifiers, classes, notations, ...

There are quite a few downsides to this approach, in particular:

- there is no mechanism for objects to declare common meta-data such
  as documentation, origin (user or auto-generated), location, etc...

- there is no mechanism for objects to declare common behavior such as
  registering with the nametab, declaring tables or maps, etc..

- there is no control of interaction of objects with the parser, kernel

- there is no possibility to systematically explore the list of
  objects currently in scope (other than hoping the ad-hoc internal
  structures are somehow exposed)

- in some cases (such as printing), we need to perform a costly (and
  messy) global imperative state update as to simulate that a module
  has been opened, due to the imperative nature of the module system
  implementation

These shortcoming are handled today by introducing ad-hoc effects in
the imperative handler, but clients such as UI interfaces or Search
have a hard time working in a clean way, the API state is far from
satisfactory.

This CEP proposes a remodelling of the `Libobject` API based on our
experience with alternative Coq toplevels and interaction systems. In
particular, we aim to:

- split concerns in the module system implementation: serialization,
  scope handling, object life-time and effects

- provide a common API for indexing a retrieving object metadata

- provide a common API for declaration of side-effects, such as
  creating tables, references, registering with the kernel, the
  parser, or the nametab

- set the path for a functional implementation of some basic
  operations of the module system, pushing the imperative state to the
  upper layers

- replace `.glob` files by a query to the object database, provide
  rich documentation, jump to definition features

## Coq DB Workshop

This CEP has its origins on the "Coq DB" workshop that was held
Thursday, March 31st, 14h00 (Paris time)

https://github.com/coq/coq/wiki/jsCoq--and-CoqDB-Working-Day

There are videos and a small summary that will be uploaded ASAP.

# Lifecycle of a `Libobject.t` object:

Lifecycle of `Libobject.t` values can be understood from the creation
point of view, which happens when new objects are registered, and its
dual, when existing objects are imported from `.vo` files.

The lifecycle is roughly as follows:

## Creation

- when clients want to "persist" (or declare) a new object, they will
  register it with `Lib.add_leaf`. `Lib`'s job is mainly to sit below
  all clients that can register objects, and track the (ephemeral)
  list of objects that have been declared while a module / library is
  built.

- when a module or section is closed, `Declaremods` will call `Lib` to
  collect the corresponding objects, and in some cases they will be
  re-declared if they survive the module / section.

- at the end of a library compilation, `Library` will call
  `Declaremods.end_library` to grab the list of objects to persist in
  the `.vo` file.

## Loading

- when a new library or module is `Required`, `Declaremods` will
  sequentially "load" the objects, basically performing the
  side-effects declared in their interface

- the objects are then stored in a internal map, as their interface
  may need to be called when importing for example

- users of `Declaremod` have to be careful about the state

# Logical and meta-logical objects

A common discussion topic is about the role that objects registered
with the kernel play. Indeed, some objects present in a `.vo` can be
understood as "logical" objects, as they will be registered with the
kernel.

A large set of objects are meta-logical: they are not registered with
the kernel, but still essential to Coq operation. Such objects are
notations, instance declarations, hints, etc...

Indeed it is a common source of discussion where the data for logical
objects should be stored. Some data will be stored invariant in the
kernel, but some other data, like documentation or location, has to
live outside it.

# Part I: common meta-data

For this part, we propose to decorate the `Libobject.t` type with a
`Data.t` structure. This approach is modelled after the approach used
in `Declare` for constant information, which has proven successful in
reducing code duplication, uniformizing the API, and making it more
extensible (see for example per-definition typing flags).

In this preliminary proposal, `Data.t`, will contain:

- `loc`: Location of the object, if it is coming from a source file
- `doc`: String containing the documentation of the object, to be extended
- `name`: Name of the object, as an `Id.t`, if not anonymous
- `category`: Already exiting in `Libobject`, used for import, could be also used by `Search`.

A draft implementation or Part I is available in PR
https://github.com/coq/coq/pull/16484

This changes affect the lifetime in the following way:

## Creation

Clients must provide the metadata when registering the objects.

## Loading

`Declaremods` already stores the objects in a map indexed by their
prefix, we propose to:

- create a reverse map or maps to allow clients to query the `Data.t`
  efficiently. In particular we could have reverse maps for
  categories, names, locations. For now we implement a reverse map for
  names, but more options could be desirable after more experimentation.

- expose the regular map for anonymous objects so clients can explore.

## Extra

It would be desirable to allow clients to register as to allow
incremental indexing of objects, of for example, to maintain a
location-based table.

Another interesting topic is versioning of objects. While preliminary,
we could include versioning information in the objects as for example
to allow to have hybrid Coq 8/9 objects, or several versions of the
same `Definition` for example.

# Part II: behavior specification

The second part of this CEP, not yet implemented, is to phase out the
`declare_object` API in favor a set of `Behavior.t` traits that an
object can implement, in a functional way.

The main goal here is to allow the upper layer to control the state,
and have more flexibility in updating the state. This has many
applications.

In particular, we can distinguish a few common behaviors:

- `Kernel` : objects with this trait provide a function to register with a `safe_env`

```ocaml
module type KernelTrait : sig
    type t
    val register : env -> t -> env
end
```
  currently the objects that register with the kernel are mainly
  inductives and definitions. Also, note that this allows to define a
  notion of soundness for document interpretation, namely, the
  document manager has to map all objects with kernel trait to the kernel.

- `Named` : objects with this trait can register with the `Nametab`, in a uniform way
```ocaml
module type NamedTrait : sig

    type path
    type elt

    module NameTable : NameTab.S with elt := elt and path := path

end
```

- `Parser` : objects with this trait can update the parser
- `Summary` : objects with this trait will register a map or value with the summary

In all cases, the registration of the trait exposes the state
manipulation in a functional way, so the upper layer [`DeclareMods`]
is responsible of threading the state in a proper way.

This should allow to reduce quite a bit of ad-hoc code duplication in
the current codebase.

We can think of a possible implementation strategy in two-phases:

1. declare the traits to be imperative
2. have to declaremods own the state, make the traits functional

I think this makes a lot sense at least for the objects exposed here.

# Open questions / risk

Main issue for the implementation of behaviors is what to do with
functors / module include. The state of this is not clear yet, we will
soon provide a first behavior implementation PR so we can evaluate the
impact on code.

# Extra notes and related work

- See also https://github.com/herbelin/ceps/blob/state-passing/text/state-passing.md
- Can we remove `Lib` at all when this is done and have all the logic in `DeclareMods`?
