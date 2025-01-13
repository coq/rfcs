- Title: A cleaner Reals library

- Drivers: Pierre Rousselin/Villetaneuse

----

# Summary

Our goal is to clean up the Reals library. In particular:
- Since the main source of documentation are the `.v` files themselves (with
  eventually some `coqdoc` formatting), having 82 files is very inconvenient for
  the user. Moreover the titles themselves can be misleading. We wish to regroup
  the content in a very reduced amount of files (e.g. `Rfunctions.v` could be
  one file).
- The proofs in Reals are very outdated, do not respect what is now considered
  good practice (strict focusing, not using automatically generated names), and
  use auto pervasively with a not very thought off hint database.  They tend to
  use a lot of unneeded automation because there are many missing lemmas. This
  in turn make Reals quite slow to compile. We believe the proofs could be
  reworked to be both prettier, shorter, quicker to compile and closer to usual
  mathematical practice.
- Some definitions are ill-chosen. We could offer alternatives with translation
  lemmas for compatibility. The unfortunate `sum_f_R0` (which lacks a notion of
  empty sum) is a good example.
- Some identifiers are ill-chosen in an international setting, for instance,
  using `pos` for "positif" (in french) when the intended meaning is
  "non-negative" or even worse, using `neg` (for "negative") instead of `opp`
  (for "opposite"). Others are not informative at all (for instance `tech_23`).
  This should, at least partially, be addressed. Although some concessions will
  have to be made in order to preserve, to a large extend, backward
  compatibility.
- The logic of Reals (as they were first formalized) is now better understood
  (see `Reals/Rlogic.v`). One can derive consequences of the weak excluded
  middle and the limited principal of omniscience and use them to simplify
  proofs and statements. This was done in the Coquelicot library, to define, for
  instance, a total "limit" function on real sequences. It can be used here as
  well.

# Motivation

We encountered Reals when writing a small course for undergraduate students.
While we were seduced by its simplicity, some aspects were very frustrating.
This lead to a first PR (coq/coq#17036) with a better `RIneq.v`.
After that, we tried to clean up other files but got carried away. And indeed,
we believe that a substantial change is off the scale of small pull requests.

We think that a better Reals library could be very useful for teaching
mathematics, for instance in first year of university (or even in high school).
The fact that it does not use a lot of abstraction can be an asset in this
regard. Our course went all the way to convergence of sequences. It would
however be difficult to cover more content without a better designed library.

While we are aware that the standard library may lose its "standard" status in
the future, this may take some time. And even after demotion, if it does indeed
happen, having a cleaner library would make it easier to find maintainers.

# Detailed design

The detailed design is to be discussed with Reals maintainers (namely Hugo
Herbelin, Laurent Théry and Guillaume Melquiond) and/or other Coq developers.

Here is a first rough and incomplete plan:
1. Yet another PR about `RIneq` completing results about division and
   subtraction and adding common one-step reasoning lemmas, for instance
   "changing the side of a term" in a equation or an inequality.
   These are needed to obtain smaller proofs afterwards and correspond to usual
   mathematical practice for, e.g. high school students.
2. Incorporate all parts of `Rfunctions` inside `Rfunctions` itself in, say, the
   same order as they are required. This could be split in separate PR (for
   instance one for `Rsqr`, one for `Rmin` and `Rmax`, etc). The continuous
   integration would tell if the small files need to be kept with a deprecation
   period or can be thrown away directly.
3. The same method can then be applied to the other main parts of the library,
   namely `SeqSeries` `Rtrigo` `Ranalysis` and `RiemannInt`.

The following [branch](https://github.com/Villetaneuse/coq/tree/Reals_prototype)
contains an incomplete prototype with `Rfunctions` and `SeqSeries` as single
files. While it still needs more polishing and requires advice, it shows that it
is possible to considerably reduce the size of proofs, without any automation
except that of the `easy` tactic.

We do not, at this stage, wish to change the underlying logic of the library. We
can work with this middle ground between intuitionistic and classical logic.
While we aim at making important changes, we also wish to preserve, to some
extend, backward compatibility.

# Drawbacks

- The major drawback is probably that it may be time consuming for the
  maintainers. It could be deemed not worth it.
- Giving alternative (better) definitions could lead to increase unreasonably
  the number of lemmas, this will have to be carefully balanced.
- Having less files *could* lead, in a heavily multithreaded environment to
  increased compilation time (but we deem it unlikely). We will have
  coq/coq#17877 in mind and try to break dependencies when we can.

# Alternatives

- It may be possible to be less ambitious regarding the changes, only cleaning
  up some proofs and deprecating some lemmas, without touching the general
  structure of the library.
- Another possible design was described by Vincent Semeria: have a first layer
  of constructive mathematics and a second layer specialized using stronger
  axioms. It would be beautiful but it does not solve our issues.
- The library `mathcomp-analysis` is a very advanced analysis library built on
  top of Mathcomp, ssreflect and Hierarchy Builder.
  The definition of real numbers is never used. Instead, at this time, the
  lemmas do not rely on a specific construction of the real numbers, but are
  parameterized by instances of type `realType`. Its logic is stronger than that
  of Reals, assuming not only functional extensionality but also propositional
  extentionality and constructive indefinite description (which in turn imply
  the excluded middle in its strongest form). Most basic lemmas are instead very
  general lemmas about, say, abelian groups or ordered fields of which real
  numbers are just another instance. While it certainly is a very carefully
  written library, it is very involved and, in our opinion, demanding for the
  average Coq user. We would not use it with first year undergraduate students
  for instance. Furthermore, unless Flocq and Coquelicot are in a large part
  rewritten, they will still depend on Reals. So I think mathcomp-analysis and
  Reals should coexist, as they target very different users and have very
  different philosophies.
- The library C-Corn is another analysis library with constructions of real
  numbers. It focuses mostly on constructive analysis. The C-corn library
  is itself based on Math-classes, a library of abstract interfaces for
  mathematical structures, using Coq's type classes. It certainly is a very
  interesting library, but is probably less practical for a classical
  introductory mathematics course.

# Unresolved questions

- The status of functional extensionality in Reals has not, to our knowledge,
  been discussed. It is used to define the real numbers but never after.
  Maybe it is to keep Reals independent of this particular construction.
- The same question holds for Setoids. They are used in the definition of real
  numbers. This allows to use `setoid_rewrite` during the development, which can
  be very handy (we can rewrite under `forall` and `exists`). With functional
  extensionality and Setoids, we could rewrite under `fun` too.
