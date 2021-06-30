CApp
====

# Overview
CApp is like a package manager, but it avoids a lot of package manager responsibilities
by building packages just to support one application, hence the name is a shortening of "C Application".
CApp depends only on CMake and Git, and is actually written in the CMake language for portability.
It assumes all packages have source code stored in Git repositories and are compiled with CMake.
CApp is built around the notion of an application repository, which is a Git repository
containing "package files": CMake language files which describe packages needed
to build the application, their exact version expressed as a Git commit,
their dependencies, and the CMake arguments needed to configure each package.
By knowing only about exactly one version of each package, CApp avoids any responsibilities
related to version compatibility resolution.
What CApp does do is automate the process of cloning, configuring, building, installing,
and otherwise developing the various packages needed to build an application.
It can be used for automated installation, continuous integration, and local
developer work, all in a unified system that is portable across operating systems.

# Usage

## Installing

CApp consists of a CMake script and two drivers: a Bash driver for Linux or Mac and
a Batch driver for Windows.
The drivers can be used from the cloned Git repository or the three files can be
copied somewhere else (`capp.cmake` needs to be in the same directory as the driver).

```bash
git clone git@github.com:SNLComputation/capp.git
cd capp
cp capp.cmake capp.sh capp.bat /where/you/want
```

On Windows, this will install `capp.bat` to the `bin` directory and
elsewhere it will install `capp`.

## Setting Up an Application

### init command

The first step to using CApp is to create a repository that describes the packages needed
for the application.
This can be done by using the `capp init` command.
As the name suggests, an application repository is also a Git repository, and `capp init`
will call `git init`.

```bash
mkdir my-application
cd my-application
capp.sh init my-application
vim app.cmake
git commit -a -m "initial application commit"
```

### app.cmake

An application repository needs to have an `app.cmake` file in the top-level directory.
Similar to how Git identifies a Git repository by the presence of a `.git` directory,
CApp identifies an application repository by the presence of an `app.cmake` file.
`capp init` creates this file with some default content for convenience.
An application's `app.cmake` file should at minimum contain a call to the `capp_app` command,
whose signature is as follows:

```cmake
capp_app(
  NAME app_name
  ROOT_PACKAGES package1 [package2 ... ]
  [BUILD_TYPE <type>])
```

Where `NAME` should be the name of your overall application.
`ROOT_PACKAGES` should be the list of packages that CApp
must always try to compile.
CApp will try to build all packages that `ROOT_PACKAGES`
depend on (see the documentation on `capp_package`)
in the right order.
Most applications will have an "application package", which is a package
that contains the source code for the application but not any of the
other supporting packages.
The typical use case will be to list the `ROOT_PACKAGES` as
just the application package.
By default, CApp will configure all packages with `-DCMAKE_BUILD_TYPE=RelWithDebInfo`.
The `BUILD_TYPE` argument can be used to change the build type to something else
(CMake supports `Release`, `Debug`, and `None`).

### clone command

The easiest way to add a new package to the application repository is via `capp clone`:

```bash
cd my-application
capp.sh clone git@github.com:Unidata/netcdf-c.git
vim package/netcdf-c/package.cmake 
git commit -a -m "added netcdf package"
```

### package.cmake

The primary package information in a CApp application repository
is stored in files:

```
package/<package_name>/package.cmake
```

The actual names of packages are the names of the sub-directories
in the `package` directory of the application repository.

A package's `package.cmake` file should at minimum call `capp_package`,
whose signature is as follows:

```cmake
capp_package(
  GIT_URL git_url
  COMMIT commit
  [OPTIONS option1 [option2 ...]]
  [DEPENDENCIES dep1 [dep2 ...]]
  [NO_CONFIGURE_CACHE]
  [IGNORE_UNCOMMITTED])
```

The `GIT_URL` option should be a Git URL suitable for `git clone`
and other commands.
An example Git URL would be `git@github.com:SNLComputation/omega_h.git`.
While CApp doesn't impose restrictions on the type of Git URL,
it has only been tested with SSH protocol URLs and not
with HTTPS protocol URLs.

The `COMMIT` option should be the SHA-1 hash for a certain Git
commit, meaning the version of the package repository we
want to checkout and build.

Note that by directly listing a SHA-1 commit in `package.cmake`
files, CApp implements a submodule-like system that ensures
that the SHA-1 commit hashes of the application repository
are tied to the SHA-1 commit hashes of the package repositories.
This means that it is enough to know the version of the
application repository that was built in order to know
the exact versions of all package repositories that were built.

The `OPTIONS` list should be a list of command line arguments
to CMake when configuring the package.
For example, an item `option1` might be `-DFOO_ENABLE_FAST=ON`.
Note that CApp already defines 
`CMAKE_BUILD_TYPE` and `CMAKE_INSTALL_PREFIX` when
configuring a package.

The `DEPENDENCIES` list should be a list of package names
on which the current package directly depends.
Recall that package names are the names of subdirectories
of the `package` directory of the application repository.

If the `NO_CONFIGURE_CACHE` option is present, then when
CApp tries to re-configure a package it will first
remove the `CMakeCache.txt` file.
This is useful when packages have buggy CMake files
that produce different results when incrementally reconfigured
than they do when configured without a cache,
even with the same arguments supplied.

The `capp commit` command discussed later will check whether there
are uncommitted changes to a package's source repository.
However, some packages modify their source directory during their
configure, build, and install process even though this is bad practice.
For those packages, one can add `IGNORE_UNCOMMITTED` to the
`capp_package` arguments to tell CApp to ignore uncommitted changes
to that package.

## Installing an Application

### build command

With a copy of the application repository available, CApp can build it with the `capp build` command.

```bash
cd my-application
capp.sh build
```

### Directory structure

CApp will create sub-directories in the application repository directory called
`source`, `build`, and `install`.
Each package will have its Git repository cloned into:
```
source/<package-name>
```
Each package will have its CMake binary directory (where CMake is configured
and all intermediate build files are stored) at:
```
build/<package-name>
```
Each package will be installed by setting `CMAKE_INSTALL_PREFIX` equal to:
```
install/<package-name>
```

The `capp init` command populates `.gitignore` to ignore these directories,
since their content is an artifact of the current build and not part of the
fundamental specification of an application repository.

## Developing an Application

Another key use case of CApp is actual application development.

### Rebuilding a Package

Users of CApp are encouraged to edit package source code in the
the `source/` subdirectory of the application repository and follow
the normal Git workflow for each package as desired.
At some point, to see the effects of the source code changes,
users may want to rebuild a package and all downstream dependencies.
This can be done with the `capp rebuild` command:

```bash
cd my-application
cd source/package1
vim package_function.c
capp.sh rebuild package1
```

If given no arguments, the `capp rebuild` command will rebuild all packages.

### Reconfiguring a Package

Sometimes a change to a package is substantial enough that it requires
CMake reconfiguration in order to properly update the build.
This can be done with the `capp reconfig` command, which works just
like `capp rebuild` but will force a fresh CMake configuration of
the package prior to rebuilding.
If given no arguments, the `capp reconfig` command will reconfigure and rebuild all packages.

### Accepting a New Package Version

After a developer is done modifying a package, the natural next step is getting the
application repository to accept and point to the new modified version of the package.
This is done by the `capp commit` command, which will update the git URL and git commit that
the application repository points to.

```bash
cd my-application
cd source/package1
git pull
capp.sh commit package1
git commit -a -m "updated version of package1"
```

`capp commit` will fail if there are uncommitted changes to the package's source
repository, thereby helping to ensure that the current state of the source code
really gets captured.
This behavior can be avoided by specifying `IGNORE_UNCOMMITTED` in `package.cmake` files.

If given no package arguments, `capp commit` will operate on all packages.

### Checking Out New Package Versions

An inverse of the prior function is to update the package source in the source directory
to be exactly the version pointed to by the application repository.
We do this with the `capp checkout` command.
This is especially helpful to run after other developers have made changes to package
versions.

```bash
cd my-application
git pull
capp.sh checkout
capp.sh build
```

Even easier, the `capp pull` command is equivalent to `git pull` followed by `capp checkout`:

```bash
cd my-application
capp.sh pull
capp.sh build
```
