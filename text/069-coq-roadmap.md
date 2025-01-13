- Title: Coq roadmap

- Drivers: Nicolas Tabareau (@tabareau), Théo Zimmermann (@Zimmi48)

----

# Summary

This CEP aims to establish a short term roadmap for the Coq project (while
also stating the need for a medium and long term roadmap as well).

It highlights both:

- what are considered as priorities for the short term future,
- what priorities we commit to make progress on, based on the available
  resources.

Resources mean availability of persons to conduct the work. For technical axes,
requiring changes in the Coq system, it includes availability of persons to
review and merge the changes.

# Motivation

Producing a roadmap for the Coq project is important for several reasons:

- It helps the community to know what to expect from the Coq project in the
  future, in particular what changes can reasonably be expected to become
  part of next releases, and what changes are more likely to be delayed
  because of lack of resources.

- It helps Coq developers to choose priorities together, and to work more
  efficiently toward those by making sure that their work will be supported
  by other developers, including reviewers.

- It helps contributors to know in what areas their contribution is likely
  to receive the most support from the Coq developers.

- It helps highlighting other priorities where progress is not possible for
  lack of resources, and where the Coq project should aim to find
  more resources.

# Detailed design

The Coq roadmap is defined at three levels: short, medium and long term.
Each of these levels addresses different needs, but they should be checked
for consistency with each other.

- The short term roadmap is focused on priorities on which we are committed
  to progress in the coming months and that are thus the most likely to
  actually happen. We define a commitment from the Coq team as a commitment
  of several (at least two) contributors, including at least one relevant
  maintainer to review the work proposed by other contributors. The work
  can include implementation work when the design is already clear, work on
  the design from ideas that are well understood, or even exploratory work,
  but we should only list the parts that are likely to converge in the
  specified time-frame (i.e., if it is not clear if exploratory work will
  lead to a solution that can be implemented, we should only commit on the
  exploratory work).

- The medium term roadmap is focused on areas where we see important progress
  being needed, which could take several years to complete, and for which we
  do not always have the resources needed. It is used to guide the renewal
  of the short term roadmap, but also the search for new resources to make
  progress in these areas. The medium term roadmap should contain high level
  descriptions of the areas requiring progress, but can contain rough plans
  for how to proceed to make progress in these areas if the team already has a
  vision about the steps needed.

- The long term roadmap is focused on what our vision for the future of Coq is
  and not necessarily how we are going to achieve it. It is used to provide
  general ideas that we can check our medium and short term roadmaps against,
  and to give users more clarity on what the future of Coq could look like. It
  should include our vision of what kind of users Coq will target in the future.

As creating a roadmap and keeping it updated and aligned with the actual goals
of the team members requires a change of culture, we do not try to create all
of these levels at the same time. In this CEP, we aim to produce a complete
short term roadmap, and to start a sketch of the medium term roadmap. We will
use our experience building the short term roadmap and keeping it up-to-date
to improve the process and to start building more precisely the other levels.

## Detailed design for the short term roadmap

This roadmap is a consolidated view created by the Coq developers, based on
their shared understanding of the priorities of the project and of the
resources available to work on these priorities. It is completed by a
discussion with the community, as part of the CEP process.

The fact that an item is part of the roadmap does not mean that its design
is already fixed, as design can be part of the work to be done. Therefore,
disagreement can still arise on such items, and may require to be resolved
through PR reviews, or through the CEP process. However, if an item is
part of the roadmap, it means that the Coq developers are committed to
work on it, and that they have resources available to do so, including the
reviews, so the work should not stall because of lack of resources or lack
of interest.

The description of each item of the roadmap will include the available
resources, the expected duration of the work, and the expected outcome,
including any blocking issues that will need to be resolved to complete
the work, and any still unresolved questions to be answered.

Rules :

- An item cannot be part of the roadmap if it is not supported by at least
  two persons, including at least one Coq maintainer to review the changes.
- No one should commit to more work than they can realistically undertake
  at the same time (which will be a different amount depending on whether
  someone can work full time on Coq or not).

Each Coq Call will start with a roundtable about progress on items of the
roadmap.

Gaëtan Gilbert will be the technical coordinator of the roadmap, overseeing
progress being made.

Théo Zimmermann will be the editorial coordinator of the roadmap, proposing
to add and remove items, to reflect the evolution of the project.

Once adopted through the CEP process, the roadmap will be kept up-to-date
in a living document on the wiki (while the merged CEP will represent a
point in time). From time to time, we will go back to using the CEP process
to trigger a discussion with the community on the updated roadmap (and also
to address more precisely the medium and long term levels of the roadmap).

## Short term roadmap

As this short term roadmap is focused on priorities for which we have
committed resources, it should not be considered as an exhaustive list of
the important aspects on which we hope Coq to progress in the future (more
can be found in the medium term roadmap). Rather, it gives clarity of what
should be the upcoming changes in the next releases. Yet, some changes may
still find their way in the next releases without being listed first in
this short term roadmap (although for significant changes, it would be
better if they would).

### Change of Name: `Coq` -> `The Rocq Prover`

- Yves Bertot, Nicolas Tabareau
- 3 to 6 months

The Coq developer team has validated the renaming of Coq into the Rocq Prover. 
There are several steps to make this happen. 

#### Legal issues 

We are checking that we can actually use the new name from a legal point of view. 

#### Visual identity and website

The change of name is an opportunity to redesign the visual identity of the Rocq Prover
and to give a fresher look to the historical website of Coq. 

#### Technical issues 

On the technical issues, the renaming practically means a huge amount of search&replace 
in many parts of the code base, website links, etc. We may ask for user contributions 
for more peripherical changes, for instance on user contributions. 

### Kernel, theory

#### Template polymorphism

- Gaëtan Gilbert, Pierre Maria Pédrot
- in progress (maybe done by 8.21)
- notably https://github.com/coq/coq/pull/19228

Make template poly more usable and more robust, basing the metatheory on sort poly.

TODO:

- metatheory in metacoq (?)
- allow template univs which don't appear in the conclusion
  (eg `u` in `eq : forall A:Type@{u}, A -> A -> Prop`)
- remove above-Prop restriction in template poly (?)
- I feel like I forgot something but can't remember what

#### Algebraic universes

- Matthieu Sozeau, Pierre-Marie Pédrot. Joint work with Marc Bezem on the constraint
 inference/checking algorithms.
- CEP: todo
- PR: https://github.com/coq/coq/pull/16022
- 1 year

The goal is to support arbitrary universes (e.g. max(u+k, l)) in any position,
while generalizing the constraint resolution system to support more complex constraints
(e.g. v <= max(u + k, l)). The inference and checking algorithms are quite different from
the current one and will be less performant by at least an order of magnitude on the current 
developments that are mostly using global universes. It should however be competitive on 
universe-polymorphic code (with much less universes and constraints to consider by definition). 

This should enable a more user-friendly interface to universe
polymorphism, by reducing the number of "dummy" generated universes and constraints that 
are here only to deal with the current limitation. Also, we would get rid of the mandatory 
"refresh_universes" on the Coq API side, allowing to freely use inferered types and terms 
in tactics/tactic languages.

On the user side this mostly adds an enriched language to describe universes with +n's and max(). 
Small syntactic incompatibilities related to universe declarations (e.g `@{u v + | u <= v +}` change
to `@{u v ?| u <= v + 2 ?}`.

#### Rewrite rules

- Working Coq fork: https://github.com/Yann-Leray/coq (examples in
https://github.com/Yann-Leray/coq/blob/rewrite-rules/test-suite/success/rewrule.v).
- CEP: https://github.com/coq/ceps/pull/50
- Yann Leray, Théo Winterhalter
- DONE (8.20)

The goal is to add (unsafe) user-defined rewrite rules. This feature allows
users to add computation rules to axioms which can be useful for prototyping.
It also allows for different kinds of computation rules with respect to what Coq
currently permits: non-linearity, overlapping left-hand sides (*eg* one can write
an addition on natural numbers that reduces on both sides: `0 + n` and `n + 0`
both reducing to `n`).

It would be deactivated by default and be optionally on by *eg* setting
`Rewrite Rules On`. They should be supported by unification and the main
reduction machines (not `vm_compute` and `native_compute` for now, Coq
should give a warning when those are used with rewrite rules on). Rewrite
rules would propagate to any module that requires the module that defines them.

### Surface language

#### Make SSReflect available in Ltac2

The main missing tactics in Ltac2 are currently those of SSReflect. Exposing them in Ltac2
implies writing a good chunk of boilerplate binding code for the Ltac2-OCaml FFI
and defining the corresponding grammar rules for the Ltac2 language.

- Enrico Tassi and Pierre-Marie Pédrot

#### Notations

- Hugo Herbelin, Pierre Roux

##### General recursive notations

The objective is to grant wish [#7959](https://github.com/coq/coq/issues/7959)
(see there for details).

### Tools

#### Proved extraction

- Yannick Forster, Matthieu Sozeau
- 6 months to 1 year

Alternative to Coq's current extraction mechanism (implemented as an OCaml plugin) by a re-implementation from scratch based on MetaCoq (and thus implemented in Coq itself). This new extraction framework already supports *verified* type and proof erasure based on the intermediate language LambdaBox,  several verified intermediate compilation steps, and extraction to OCaml cmx files via [Malfunction](github.com/stedolan/malfunction). The goal is to support unverified extraction to the current back-ends OCaml, Haskell, Scheme, and JSON. To serve as a drop-in alternative to the current extraction mechanism several parts of it have to be re-implemented, including mli generation as well as Haskell, Scheme, and JSON file generation.

Other aspects that need re-examination because they are problematic already in the current extraction mechanism include
- Prop included in Type
- Module dependency analysis
- Hugo Herbelin + Yannick Forster

### Cleanup

#### Retiring the STM, step 1

- Enrico Tassi, Gaëtan Gilbert
- 1 month of work, 6 months timeframe

The objective of the first step is to have coqc and coqtop not depend on the STM, while keeping coqidetop, coqc-vio and coqc-vos depend on it.
Sub items:

- fix the kernel w.r.t. side effects and code paths in the stm. That is revive https://github.com/coq/coq/pull/16367
- `par:` implemented using [SEL](https://github.com/gares/sel) (already done in vscoqtop, to be ported to Coq). Currently `par:` does not use the STM, but uses its code to spawn workers, it depends on `-thread` etc.
- make coqc-vio and coqc-vos legacy binaries using the stm library

### Libraries and community

#### Boost stdlib development

- Cyril Cohen, Pierre Roux
- 6 months to 1 year
- [CEP #83](https://github.com/coq/ceps/pull/83)

As part of the promotion effort around the Coq Platform, we would like to
ensure that the stdlib does not enjoy special status and that Coq can be
used without it. We should consider stdlib components as libraries to
advertise for their own values as part of the Coq Platform packages, but
without their historical origin, or their development and release process
being tied to Coq, (mis)leading users to consider them as the only
officially recommended libraries.

In particular, we should:

- Identify consistent stdlib components that can be used independently
  from each other and that would be worth distributing as separate
  packages. Identify their maintainers and give them freedom to define
  the future of the components they maintain, in the limits set by the
  Coq Platform charter. Allow maintainers to extract stdlib components
  to maintain and evolve them outside the core Coq repository and to have
  their own release schedule and versioning scheme, in case they wish to
  do so.

- Extract the prelude + a minimum set of components that alternative
  general libraries like MathComp and coq-stdpp need as a basis.
  Make sure that this reduced core stdlib component can evolve to allow
  other libraries to use newer features of Coq (like universe polymorphism,
  SProp and primitive projections).
  Stop including any other stdlib components as part of the `coq` opam
  package and encourage maintainers of Coq packages in other distributions
  to do the same.

The technical details (prelude content, split into packages, logpath,
mono/multi repo, call for maintainers, documentation, test-suite,
CI,...) will be discussed in this CEP: https://github.com/coq/ceps/pull/83 to
ensure a reasonnable agreement is reached on those points before
implementing any actual change.

## Medium term roadmap

This part should be considered as a sketch, so not everything in there
is presented at the same level of detail yet.

### Kernel, theory

#### Streamline the use of `SProp`

Short-term work on sort polymorphism should make it easier to adapt
all tactics and libraries so that `SProp` becomes actually usable.

For tactics it's mostly a matter of waiting for bugs to be reported so we know what it broken.

#### Guard condition

There are several ways to make the guard condition evolve. Some
usability issues today could be lifted if they are properly
justified as not extending the theoretical set of functions that can
be defined. The guard condition could be made more flexible by
allowing to choose between a very safe mode, a normal / legacy mode,
an experimental / extended mode, and the disabled mode that can
already been set today.

Here are some of the items that could possibly be worked on in the
relative short-term depending on availability of reviewers and
consensus on how to handle them:

- Guard condition able to treat nested inductive types as mutual inductive types (recomputing the recursive structure dynamically), Hugo (PR coq/coq#17950), a few weeks for discussions and reviewing
- Guard condition able to detect uniform parameters of inner fixpoints, Hugo (PR coq/coq#17986), a few weeks for discussions and reviewing
- Expanded constructors of a branch in a `match` considered smaller for the guard condition (CEP #73), a few weeks depending on discussions
- Refinement of the guard condition through `match` constructs (PR coq/coq#14359), a few weeks for discussions and reviewing

#### Sections

- Design of a way to refer to the generalized version of a constant/inductive from within the inside of a section (depends on the time needed to reach a consensus on the design)

#### Observational type theory

#### Global fixpoints

- Hugo less convinced of the importance of global fixpoints vs modifying the fix/cofix rules of CIC so that they unfold for named fixpoints on the name rather than the body of the fix

#### Streamline the use of primitive projections

Some discussions still need to happen to create a precise plan on how to proceed.

One item that is often discussed is the following:

- Removal of compatibility layer

#### Sections

- Design of a way to refer to the generalized version of a constant/inductive from within the inside of a section

#### Modules

### Surface language

#### Ltac2

The overarching goal is to get to the point where we can recommend new
developments to only use Ltac2 (without having to load Ltac1). There are several
fronts on which to make progress.

One major point is to make available in Ltac2 all the basic tactics from Ltac1.
The main missing part is currently the ssreflect framework.

Another important thing for extensibility is the table feature. One should be
able to define global tables with several kind of indices through a vernacular
and extend them after the fact. With this feature, mutable definitions are just
a specific case of tables with a unit index.

`coqdoc` needs to understand Ltac2 (link to definitions, skip bodies).

#### Parsing

- Hugo Herbelin, Pierre Roux
- 2 years

We plan to remove the recovery mechanism of the camlp5/coqpp parsing
engine (see [#17876](https://github.com/coq/coq/pull/17876)). This
will make the parser simpler and easier to understand and will enable
to actually implement `no associativity`, which is currently just an
alias for `left associativity`. This should also pave the way to lift
the restriction that a same parsing level cannot be both left and
right associative, avoiding conflicts between libraries. See
[CEP 71](https://github.com/coq/ceps/pull/71) for more details.

A first step required in this direction is to enable declaring
notations at the same level than previous notations with a common
prefix, without knowing the said level. This will make it possible to
modify levels of notations without breaking backward compatibility
with their dependencies. Once this first step done, users will need
wait to require the Coq version implementing it, then use it to
finally enable changing levels of notations that currently use the
recovery mechanism. Fixing the levels of those notations will
eventually enable to remove the recovery mechanism.

We will finally precise the design of a more liberal handling of
associativity levels, based on arbitrary strings and ordering
constraints (alike universe constraints) rather than the current 0 to
100 integers. This should ease combination of various libraries by
limiting the current conflicts on notations.

#### Attaching comments to declarations

- Hugo Herbelin + ??
- time needed to converge on the design

PR [#18193](https://github.com/coq/coq/pull/18193) implements a table for binding comments to declarations, but it lacks surface syntax.

#### AST / interpretation

#### Bi-dimensional notations

#### Nametab / libobject

#### Removing clenv

#### Unifall

#### Unification heuristics

(improving evarconv)

#### Support for async tactics

#### Reactive elaboration

#### Functionalisation

### Tools

#### Remove CoqIDE

See [068-coqide-split.md](068-coqide-split.md).

### Libraries and community

#### Promote the Coq Platform

The Coq Platform is already the officially recommended installation method.
We would like to make it clear to users that they are encouraged to rely on
the packages that it provides, and that libraries in the Platform should be
generally considered as a collaborative extended standard library for Coq
(the historical stdlib being only a part of this).

As part of this effort we should do:

- Editorial work for curation of packages included in the Coq Platform
  (to bring guarantees on their level of quality).

- Consolidation of documentation, so that it is easy for users to find
  documentation about the included packages, ideally with a consistent
  presentation.

- Infrastructure / automation work to stabilize the release process of
  the Coq Platform and ensure that releases are more consistently
  delivered according to a predefined schedule.

#### Tooling for building libraries

(Coq universe)

# Drawbacks

Creating and updating a roadmap requires significant work in itself. Our
view is that this work should be nonetheless useful by making the rest of
the work around Coq more efficient.

# Alternatives

We could either do nothing, or try to coordinate work on selected topics
only, without creating a global overview and unified process. However,
experience leads us to believe that this global overview and unified
process is needed to create a shared understanding of the priorities of
the project and to make significant progress toward them.

# Unresolved questions

We leave many questions for future work, including the frequency of the
CEPs needed to update the roadmap, how the regular discussion of the work
on roadmap priorities will take place, and how to build the medium and
long term roadmap.
