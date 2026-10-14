# Software Consistency Controls
* **Status:** In-Review
* **Author(s):** douglasjacobsen
* **Last Updated:** 2026-10-14

---

## Overview
Ramble currently supports several directives related to the underlying software
stack. These include `define_compiler` and `software_spec`. The goal of this
document is to clean up the semantics around defining compilers (and software
packages) that applications / experiments require.


### Current Issue / Current State / Motivation
When Ramble was first created, we wrote software specs and compiler definitions
into application definitions (this is still true as of writing this document)
to provide an easy default for end-users to set. Ramble provides a front-end
command ramble workspace concretize which is intended to extract these
definitions, and write them into the workspace’s ramble.yaml file. The main
issue comes in when users want to customize these through some other object
that defines similar software packages.

An example of this can be illustrated by imagining how a system class could
define one or more compilers that should be used within that system’s
environment. Currently the system object would define a compiler package, but
that package can’t be used as it’s not tied directly into the application
definition. There isn’t a nice semantic (beyond just providing consistent
default names for standard packages) that allows an external object to define
the compiler (or MPI library) that is used within an experiment. As a result,
users are left having to manually configure their experiments if they want to
use a system suggested default compiler.

As an illustration, imagine we have the following.

System class:

```
with when("experiment_compiler=gcc14"):
  define_compiler("gcc", spec="gcc@14.2.0")

with when("experiment_compiler=gcc15"):
  define_compiler("gcc", spec="gcc@15.1.0")
```

Application class:
```
define_compiler("gcc", spec="gcc@9.3.0")
software_spec("gromacs", spec="gromacs ... ", compiler="gcc")
```

The system class implies that the user should be able to change the value of
the `experiment_compiler` variant, and change the underlying compiler. However
the way the application class is written makes this impossible.


## Requirements

* Package dependencies can be easily configured by end users during concretization:
    * i.e. a user can change the compiler, or MPI used within their experiments
* Application definitions (and other objects?) can define the package types they require (compiler, MPI, anything else?)
    * It’s probably worth exploring how we want to convey this. This is a semantic Spack already has, and us adding an explicit `package_dependency` might just complicate the state of things.
* Object definitions can register a package / spec to satisfy a dependency (i.e. gcc can be registered as a compiler)
* Application definitions can define software specs that rely on some external definition
    * i.e. a system class defines the compiler that should be used for my application
* All changes need to be backwards compatible

### Goals

Ultimately the goal is to make the user experience for selecting system level
software more consistent across application definitions, and to remove the need
for applications to define system software as required (though, they will still
be able to do this).

### Non-Goals

This design, and implemented, do not aim to tightly control the mechanisms
developers have for defining software specifications.

## Implementation Plan / Detailed Design

Overall this proposal includes three primary changes (at least for the objects in the ramble repository).

### Language Features

To aid applications (which require a compiler / MPI / library to function
properly) in connecting to software packages that should come from a system
object, language features will be added to build these connections.

#### Software Registration

Objects will be allowed to register software packages as standard types, or arbitrary strings.

Example system class:
```
from ramble.definitions import STD_SPECS

class Foo(SystemBase):
  define_compiler("system-compiler", pkg_spec="gcc@8.5.0")
  register_spec(provides=STD_SPECS.ExperimentCompiler, pkg_spec="gcc@14.2.0 +binutils target=x86_64", compiler_spec="gcc@14.2.0", compiler="system-compiler")
  register_spec(provides=STD_SPECS.ExperimentMPI, pkg_spec="intel-oneapi-mpi@2021.18.0 +external-libfabric")
  register_spec(provides="libfabric", pkg_spec="libfabric@2.4.0 fabrics=rxm,verbs,psm3,tcp,sockets")
```

And other objects will be able to create specs from these registered specs:

Example application class:
```
from ramble.definitions import STD_SPECS

class Gromacs(ApplicationBase):
  define_compiler("gromacs-compiler", from_def=STD_SPECS.ExperimentCompiler)
  software_spec("gromacs-mpi", from_def=STD_SPECS.ExperimentMPI)
  software_spec("libfabric", from_def="libfabric")
  software_spec("gromacs", pkg_spec="gromacs@2025.3", compiler="gromacs-compiler")
```

When defining specs from registered definitions, additional arguments can be used to override values in the original definition.

e.g.
```
from ramble.definitions import STD_SPECS

class Wrf(ApplicationBase):
  software_spec("wrf-mpi", from_def=STD_SPECS.ExperimentMPI, pkg_spec="intel-oneapi-mpi@2021.17.1")
```

Would use the base definition from `STD_SPECS.ExperimentMPI`, but then
overwrite the `spec` portion of the definition. In this case, there isn't a
good reason to do this, but you could use this to override the compiler used on
a package if you needed to. Essentially, explicitly defined arguments on the
`software_spec` and `define_compiler` directives take precedence.

##### Semantics

* The `register_spec` directive should use lazy deffered resolution. When
  registered specs are extracted (in `software_spec` or `define_compiler`), the
  reference for the referenced spec should be extracted from the registered
  definitions.
* At least one of either the `from_def`, or the `spec` attributes (on
  `software_spec` and `define_compiler`) are required.
* If any additional arguments are provided to `software_spec` or
  `define_compiler` in addition to `from_def`, they will override the
  corresponding definion in the registered definition.
* The precedence of registered specs defined within objects will map the
  existing object precedence order, e.g. Application > Modifier > Platform >
  System > Base
* Objects that refer to a registered spec (i.e. an appliaction) that is not
  registered in a diffrent object should cause an error that is easily understandable.

#### System Class Changes

Existing system classes in Ramble's builtin repo (including the user-managed
system) should register the standard specs that are required. For the purposes
of Ramble's builtin repo, that would include the following:

```
class UserManaged(SystemBase):
  register_spec(provides=STD_SPECS.ExperimentCompiler, spec="gcc@9.3.0 +binutils target=x86_64", compiler_spec="gcc@9.3.0")
  register_spec(provides=STD_SPECS.ExperimentMPI, spec="intel-oneapi-mpi@2021.17.1")
```

#### Application Class Changes

All application definitions in the builtin repo should have their
`define_compiler` and `software_spec` directives updated to connect to these
standard spec names where possible.

## Alternatives Considered 

An alternative was considered for simply organizing around standardized package
names (such as `experiment-compiler` and `system-compiler`). However this is
brittle, as there is no enforcement of this, and it would be very easy for the
system to become inconsistent and messy. This also makes it more difficult for
one center to share their configuration files / object definitions with another
center, as the names are inconsistent across each of them.

## Future Work

Eventually, it would be beneficial to have required specs within specific
classes. This is not added into this design, primarily to allow backwards
compatibility. However, the system classes were only recently added into
Ramble, and have not really been adopted much by external collaborators. As a
result, it's possible that we should simply add the requirement into this
design and perform both in one shot.

The required specs could be within any object's base class, and for the most
part we would only add required packages to the base system class.


This would also require adding a method to the `object-mixin` class to make sure
any required specs are registered by that object.
