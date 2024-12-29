- Title: Rocq Package Metadata: Piggybacking on Findlib

- Drivers: Rodolphe Lepigre

----

# Summary

The current installation scheme for Rocq theories, which consists in mingling
files from various packages under a single `user-contrib` folder, poses many
modularity problems. To solve them, we propose to piggyback on existing OCaml
Findlib infrastructure, which would support a much better installation scheme,
targeting different directories for different packages (as is typically done
in other languages, for example OCaml). This proposal is in part motivated by
the desire to improve the Rocq support in the `dune` build system.

# Motivation

Rocq packages, typically in the form of Opam packages, currently adhere to the
following installation scheme (focusing on plugins and theories only). If
there are plugins, they are simply installed as standard OCaml libraries. This
builds on existing OCaml infrastructure, and works very well in practice since
the move to the `findlib` method for loading plugins. Rocq theories, however,
are all installed under `$PREFIX/lib/rocq/user-contrib/`, independently of the
package they originate from.

The `user-contrib` setup poses several modularity issues:
1. The connection between theories and the packages they originate from is
   lost. In particular, this makes it non-trivial to uninstall a package, and
   extra metadata needs to be kept in a package-specific way. For example, the
   `dune` build system relies on its `dune-package` files to remember exactly
   what files belong to a package.
2. Two packages can overlap in the directory paths they define, but the setup
   prevents them from defining theories with the same name (they would end up
   at exactly the same file path). For instance, this forbids one to provide
   alternative packages defining theories with the same interface, which can
   be useful for configuration options, or alternative implementations when
   there are various trade-offs to choose from.
3. It is not possible to infer the transitive dependencies of a given package
   from its installed theories. As a consequence, build systems are left with
   two alternatives:
   - Always give access to the whole of user-contrib, which is what happens
     with typical `rocq_makefile` setups.
   - Only give access to user-selected theories (as with the `dune` setup),
     but require users to make all transitive dependencies explicit, which
     clearly does not scale.
4. It is not possible to know what plugins a given package might want to
   load, which again poses problems for build systems like `dune`.

# Detailed design

The main idea is to piggyback on the OCaml Findlib infrastructure. It is used
for plugins already, and that should remain unchanged. In particular, plugins
should continue to be installed in the same way.

The proposal is to install Rocq theories similarly to OCaml packages. Here,
the notion of package is that of Findlib, not that of Opam, even though they
are related (an Opam package generally consists in a collection of Findlib
packages). Findlib packages are declared in `META` files, which typically
look like the following.
```
version = "dev"
description = "..."
requires = "str unix yojson"
...

package "subpackage1" (
  directory = "subpackage1"
  version = "dev"
  description = "..."
  requires = "str unix yojson"
  ...
)

package "subpackage2" (
  directory = "package2"

  package "subsubpackage1" (
    directory = "subsubpackage1"
    version = "dev"
    description = "..."
    requires = "str unix"
    ...
  )

  package "subsubpackage2" (
    ...
  )
)
```
Assuming that the above `META` file is placed in a folder `example`, then
it declares four packages named `example`, `example.subpackage1`,
`example.subpackage2.subsubpackage1`, and
`example.subpackage2.subsubpackage2`.
Each package has associated metadata, including a list of the other packages
it depends on (`requires` field), and a directory (`directory` field). The
directory for package `example` is the `example/` directory holding the `META`
file. The directory for `example.subpackage1` is `example/subpackage1/`. In
effect, the nesting of packages reflects a nesting of directories, and it is
expected for packages to place their files (which, or OCaml would be `.cma`,
`.cmxa`, and `.cmxs` files, for example) in their respective directories.

For Rocq theories, the same structure can be preserved: the package name can
be used to require the corresponding theory in build systems, and the package
directory can be used to store the theory's files (`.v`, `.vo`, `.glob`). The
`requires` field can also be used to specify dependencies on both plugins, and
other theories uniformly. The only missing piece is the expected directory
path for the theory. For that, we can simply rely on a new `rocqpath` field,
whose presence also indicates what findlib package is a Rocq theory.

## Package structure

A Rocq theory Findlib package contains the following information:
- The package name built in the usual Findlib way. Note that the package name
  may (and often will) differ from the directory path of the theory. Indeed,
  the contrary would be way too constraining (and would also prevent fixing
  modularity issue 2). For example, the findlib package for the Rocq stdlib
  would likely be named `rocq-stdlib`, while its directory path is `Stdlib`.
- Field `rocqpath` specifying the directory path associated to the package's
  theory. The presence of this field also distinguishes Rocq theory packages
  from standard OCaml package (including Rocq plugins).
- Standard field `directory`, indicating the directory under which the files
  for the package are expected to be found. Unlike OCaml packages, theories do
  not generally have a flat directory structure. To guard against collisions
  between package names and Rocq path members, theory files should be stored
  under an additional `rocq.d` directory. Note that since `rocq.d` contains a
  dot, it does not constitute a valid package name, which should prevent name
  clashes in practice. (Generally, (sub-)package names coincide with their
  directory field, but that is not a requirement of the META format. If that
  convention was to be adopted, clashes would become impossible.)
- Standard field `requires`, giving a list of dependencies, in the form of
  package names. These include both plugins and other theories, as they can
  be handled uniformly. Note that a package can always be identified as a
  theory by looking for the `rocqpath` field.
- Other standard metadata fields like `version` or `description`.

## Requiring a package

When a package `my_package` is required via a build system, the metadata can
be used to compute a set of transitive dependencies for it. Using `ocamlfind`,
this can be done as follows.
```
ocamlfind query p -r -format "%p"
```

The obtained packages can then be sorted into two lists: one containing OCaml
libraries (including Rocq plugins), and one containing Rocq theories. One may
run the following command to determine whether a package is a theory or not.
```
ocamlfind query some_dep -format "%(rocqpath)"
```
The result is empty if `some_dep` is a standard OCaml package (possibly a Rocq
plugin), and it gives a directory path for Rocq theories.

The mapping for each theory package `some_theory` can then be computed using
the following command.
```
ocamlfind query some_theory -format "-Q %d/rocq.d %(rocqpath)"
```

## Example: stdpp

The packages for [stdpp](https://gitlab.mpi-sws.org/iris/stdpp) could end up
being installed under `<PREFIX>/lib/rocq-stdpp/`, which would contains the
following `META` file.
```
version = "dev"
description = "An extended \"Standard Library\" for Rocq."
rocqpath = "stdpp"
requires = "rocq-stdlib"

package "bitvector" (
  directory = "bitvector"
  version = "dev"
  description = "A library for bitvectors based on std++."
  rocqpath = "stdpp.bitvector"
  requires = "rocq-stdpp"
)

package "unstable" (
  directory = "unstable"
  version = "dev"
  description = "Unfinished std++ libraries."
  rocqpath = "stdpp.unstable"
  requires = "rocq-stdpp"
)
```

Differences with the current `stdpp` setup:
- Rocq-ified package names.
- Use of sub-packages form `stdpp.bitvector` and `stdpp.unstable`, instead of
  different Opam packages.

## Example: coq-elpi

```
rocqpath = "elpi"
requires = "rocq-stdlib coq-elpi.plugin"
...

package "plugin" (
  directory = "plugin"
  requires = "rocq-core.plugins.ltac rocq-core.vernac elpi"
  ...
)

package "apps" (
  directory "apps"

  package "derive" (
    directory "derive"
    rocqpath = "elpi.apps.derive"
    requires = "rocq-elpi rocq-elpi.apps.derive.elpi"
    ...

    package "elpi" (
      directory "elpi"
      rocqpath = "elpi.apps.derive.elpi"
      requires = "rocq-elpi"
      ...
    )
  )

  package "tc" (
    directory "tc"
    rocqpath = "elpi.apps.tc"
    requires = "rocq-elpi rocq-elpi.apps.tc.elpi rocq-elpi.apps.tc.plugin"
    ...

    package "plugin" (
      directory = "plugin"
      requires = "rocq-core.plugins.ltac rocq-core.vernac rocq-elpi.plugin"
      ...
    )

    package "elpi" (
      directory "elpi"
      rocqpath = "elpi.apps.tc.elpi"
      requires = "rocq-elpi"
      ...
    )
  )

  ...
)
```

# Drawbacks

None, except the eventual need to port all projects to use a build system that
respects the new installation scheme. However, as a first step, it is possible
to keep using the current installation scheme at the same time.

# Alternatives

All reasonable systems have a modular installation scheme. An alternative
implementation would be to start from scratch and not reuse the Findlib file
format, but given the close integration of Rocq in the OCaml ecosystem that
probably would not make sense.

# Unresolved questions

Can we adapt `rocq_makefile` so that it expects dependencies to be installed
following the considered installation scheme, and itself installs libraries
appropriately?

We need to confirm that this is good enough for good `dune` support.
