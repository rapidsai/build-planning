# About this documenting

TODO: Some of the sequence is ending up quite far off.

This document

Goals:

- To provide a one-stop shop for all information relating to RAPIDS build, packaging, and deployment. If you have a question, it should be possible to find the answer by starting at this document (including by being linked from this document to other, more specific docs elsewhere).


Non-goals:

- This document is meant to be comprehensive, but not self-contained or exhaustive. There is a lot of great documentation in other forums that this document links to freely. If there is a more appropriate home for a particular piece of information, it should go there.
- This is not a "getting started" guide. For that, see the documentation on [devcontainers](link), which are our preferred entrypoint for starting RAPIDS development, or the various CONTRIBUTING guides in each repository.



# Building RAPIDS

## General Case

RAPIDS packages are typically composed of C++ and Python components.

## CUDA/C++

The CUDA and C++ libraries in RAPIDS are built using [CMake](link).
Generally speaking, every RAPIDS library has some shared dependencies.
Everything depends on some part of the CTK (usually at least the CUDA runtime).
Tests and benchmarks are placed in subdirectories with their own CMake lists files.
Tests usually use the [Google testing framework](link), while benchmarks use a mix of [Google benchmark](link) and [NVBench](link).

Dependencies in CMake are managed using [CPM.cmake](link), a package manager for CMake that enables downloading dependencies and building them from source if they cannot be found.
This feature allows RAPIDS libraries to be built on systems where dependencies are not already installed.
We do not use CPM directly, but rather via wrappers present in [rapids-cmake](link).
rapids-cmake is a set of CMake modules written for RAPIDS to encapsulate common CMake logic.
Among the most important features of rapids-cmake is standardizing dependency handling via CMake and standardizing tracking of these dependencies in a way that is propagated to consumers.
That means that when building a project that relies on a RAPIDS library, say cudf, a `find_package(cudf)` call will automatically ensure that all dependencies of cudf are found as well.
Without that functionality from rapids-cmake, that configuration becomes the responsibility of each project like cudf to write its own config file accordingly, or worse, of consumers of cudf to find all of its dependencies before finding cudf.

Since they are standard CMake libraries, RAPIDS C++ libraries can be built like any other CMake project using the standard commands:
```
cmake -S . -B build -GNinja
cmake --build build
```
Every library's CMake handles downloading CMake (usually by a `fetch_rapids.cmake` or `rapids_config.cmake` file at the repository root.
Note that consumers are still generally responsible for having all other dependencies installed.
Dependency installation is covered later in [the dependency section](link).

## Pure Python packages

RAPIDS C++ libraries are generally exposed via Python wrappers.
These wrapper libraries usually contain Python [extension modules](link), Python modules written in a compiled language that are exposed to Python using the [CPython C API](link).
The standard approach used in RAPIDS is to write these modules using [Cython](link).
Cython is a superset of Python that enables writing more efficient code using strongly typed variables and removing the various overheads associated with Python's flexible interpreter.
Cython also offers a way to write foreign function interfaces (FFI) in Python, allowing you to call C or C++ (or FORTRAN) functions from Python via Cython wrappers.
This is the primary use case for Cython in RAPIDS, wherein C++ library functions and classes are exposed to Python via Cython bindings.

Cython code must be compiled, just like C code.
The Cython compiler generates C code using the Python C API from Cython code, and then compiles that code down to a native library.
Traditionally, Python packages are built using [setuptools](link), and Cython hooks directly into this process [with its own setuptools plugins](link).
With the modern, more complex builds that have become common and are present throughout RAPIDS, we prefer to instead handle the full compilation process with CMake.
setuptools is not a proper build system, and compiling complex extensions with setuptools easily becomes intractable when complex dependency chains must be managed or compiler flags must be carefully controlled.

To integrate CMake builds with Cython, we use [scikit-build-core](link).
This package is a [PEP 517 builder](link), which means that the build follows a standards-compliant API that we can hook into if necessary (more on that later).
At the level of the build, what this means is that we write CMake code to compile our Python packages.
One of the first steps in the CMake is to find the corresponding C++ library, e.g. the cuDF Python build calls `find_package(cudf)`, to find the C++ library.
Compilation of the Cython modules is managed using some CMake modules in rapids-cmake.

Some RAPIDS packages are pure Python and simply depend on other packages for compiled functionality.
Those packages do not contain any Cython or CMake, and instead use the more traditional [setuptools](link) build backend.

To build and install Python packages, you follow the same approach used to build any normal Python package:
```
python -m pip install --no-build-isolation /path/to/project
```
where `/path/to/project` is the path to a directory containing a `pyproject.toml` file.
Note the specification of `--no-build-isolation` above; this assumes that you have already installed all the Cython and C++ build dependencies of your project into your environment.
Currently this is a requirement of RAPIDS builds, which are not able to correctly specify the Python package requirements [for more information, see the dependencies section](link), but this [will be fixed in the future](link to rbb).
This command also assumes that the corresponding RAPIDS C++ library has already been installed to a system library location.

## build.sh scripts

Eevry RAPIDS repository contains a `build.sh` script at the root of the repo.
This script provides a convenient interface for building all of the components of the project through a single command `./build.sh ...`.
This script should be kept up-to-date when new (C++ or Python) components are added to a library, when CMake options are changed, etc.
`build.sh` is seldom more than a trivial wrapper around the `cmake` and `pip` commands listed above.
However, it does set some convenient environment variables and/or CMake flags to ensure that the user gets a compatible set of installations on their local machine.
This may include setting the CUDA architectures to build for, or ensuring that the Python package build is provided the path to the C++ package build.

# Packaging files

RAPIDS is generally delivered to users either via packages or in containers.
The various container options for getting RAPIDS are out of scope for this document, but the package configurations are assuredly not.
The two primary kinds of package distribution built in RAPIDS are conda and pip packages.

# Dependencies

RAPIDS projects need to specify dependencies in multiple ways.
The two most important ones are those mentioned in [the packaging section](link), conda recipes and pyproject.toml files.
These control the metadata embedded in our build packages that are distributed to users.
However, we also need ways to generate lists of requirements for local installation for both pip and conda users; these take the form of requirements.txt and environment.yaml files, respectively.
More file formats may be added in the future if RAPIDS attempts to support other packaging formats.
In some of these use cases we always want to persist these changes out to disk to files that are checked into source control, whereas in others a more transient output is sufficient (such as in CI, more on that later).

The [rapids-dependency-file-generator](link) tool (dfg for short) is our answer to these problems.
dfg defines a specification for a single file, dependencies.yaml, that lives at the root of each RAPIDS repository.
This file is intended as a single source of truth for all the dependencies of a project.
As of this writing, this is not quite true for a few reasons:

- [dfg does not yet support meta.yaml files](https://github.com/rapidsai/dependency-file-generator/issues/7)
- As mentioned in the [pyproject.toml section](link), Python package dependencies may need to be adjusted for the CUDA version. This logic is currently handled in [the scripts we use to build wheels in CI](link).
- Dependencies that are also C++ build dependencies are specified via [rapids-cmake's versions.json file](link) and by corresponding information embedded in CMake function calls.

## conda

[conda](link) is a cross-platform, language-agnostic package manager.
Although it was originally developed for Python use cases, it is not specific to Python and can host packages built in any language.
From a Python packaging perspective, a major strength of conda is its ability to specify native library dependencies of Python packages with ease.
That means that it is possible to acquire essentially the entire dependency tree of most RAPIDS projects using conda.

conda packages are specified using [recipes, in the form of meta.yaml files](link).
By convention, RAPIDS stores recipes in the conda/recipes/${PACKAGE\_NAME} directory.
TODO: When is CBC used?
The actual build logic for a package is controlled using the build.sh script in the recipe directory.
Typically, recipe build.sh scripts simply invoke the top-level build.sh script with a specific set of arguments.
This allows both Python and C++ conda packages to follow the same process and simply request that the top-level `build.sh` script build the right package.

TODO: Talk about the CUDA conda packages here
TODO: Talk about how we handle CUDA 11 vs CUDA 12+, especially anything that's specific enough to RAPIDS to not fit in the more general conda-forge document we're writing

## pyproject

Python packages are configured via [pyproject.toml files](https://packaging.python.org/en/latest/specifications/pyproject-toml/).
A typical RAPIDS repository places Python packages at python/${PACKAGE\_NAME}, so that directory will contain the pyproject.toml file.
In addition to the project metadata indicated in the linked specification (analogous to the metadata in the conda meta.yaml recipe), pyproject.toml is also designed for use as a general configuration file for arbitrary Python tools related to a project.
The most common use case of this is for the build backend; scikit-build-core is configured in the `tool.scikit-build` table.
Note that since pip and conda are independent package managers, a project's dependencies may not be the same on pip and conda.
Therefore, the dependency lists in pyproject.toml may not match those in conda's meta.yaml exactly.
In general, these lists should not be modified directly.
[For more information, see the dependencies section](link)

When building our Python packages for distribution via pip, we need to make some additional modifications.
Python packages distributed for pip are called [wheels](link).
Since pip is only a Python package manager, not a general package manager, it has no knowledge of native dependencies nor CUDA.
These limitations mean that we must create a separate wheel for every CUDA version we want a package to support, and that package's dependencies must also be appropriately updated to match that CUDA version.
To give a concrete example, we publish cudf-cu11 (CUDA 11) and cudf-cu12 (CUDA 12) wheels, and these depend on rmm-cu11 and rmm-cu12, respectively.

This difficulty is the reason why [the section on building Python packages recommended turning off build isolation](link).
The default dependency list specified in pyproject.toml for our packages does not contain the necessary suffix, and will therefore be incorrect.
However, the correct dependency list may be generated using dfg, and that requirements.txt file can be installed into the environment by the developer.
TODO: Should discuss that we try to keep environment.yaml files handy for developers in the repo.

When we build wheels for distribution, we modify the metadata in pyproject.toml directly before building the package to address the above concerns.
These modifications can be seen in [our CI scripts](link).
If developers wish to build and/or install a specific RAPIDS wheel locally, they will have to make similar modifications.
In the future, this will be fixed with the [`rapids-build-backend`](link) (rbb).
rbb is a PEP 517 build backend wrapper that essentially provides the glue between dfg and the underlying build backend (typically scikit-build-core or setuptools), ensuring that when a Python package is built the appropriate suffixes are added.

## Nightlies

RAPIDS packages are distributed both as nightlies and as final releases.
The term "nightly" is somewhat misleading; new RAPIDS packages are in fact built and published every time a PR is merged.
conda packages go to the [rapidsai-nightly conda channel](link), while pip wheels go to the [nightly simple index that we host via anaconda](link).

The distribution and usage of nightlies causes additional difficulties for managing dependencies of wheels.
Due to the frequency of changes in RAPIDS, RAPIDS packages depend on nightly versions of other RAPIDS packages.
With conda, dependency constraints of the form `X.Y.*` will permit the installation of nightly packages as well as releases.
pip is stricter and will not generally install nightlies unless the `--pre` argument is specified.
However, pip can be instructed to install nightlies of specific packages by [using any specifier containing a pre-release or development version](https://pip.pypa.io/en/stable/cli/pip_install/#pre-release-versions).
Therefore, during wheel builds we also do this [in our CI scripts](link).


# CI

All RAPIDS packages are built using the standard infrastructure leveraged by CI.
Our CI infrastructure is a complex network of different pieces, built around [Github Actions](link) workflows running on custom containers.

## CI scripts

The logic for running RAPIDS CI jobs is encoded in scripts that are placed in the ci/ directory of every repository.
These include scripts for building packages, running test, linting code, and other more specialized tasks.
While each repository is free to define whatever scripts it requires, most RAPIDS repositories share at least some common scripts:
TODO: Probably want to rename the conda scripts to have conda in them.

- `check_style.sh`: This script runs style checks. In general, RAPIDS projects use [pre-commit](link) for style checks, so the style checking script is usually little more than a pre-commit invocation. TODO: Should there be a section on pre-commit in this document?
- `build_cpp.sh`: Builds the conda C++ library.
- `build_python.sh`: Builds all of the conda Python packages.
- `build_wheel_${lib}.sh`: Build the wheel ${lib}. May be multiple per repo.
- `test_cpp.sh`: Runs test on the conda C++ library.
- `test_python.sh`: Runs test on all the conda Python packages.
- `test_wheel_${lib}.sh`: Test the wheel ${lib}.

Some repositories will include helper scripts in this directory to share logic between the above scripts.
This practice is especially common if there are multiple wheels to build since the logic for building various wheels is usually similar.

These scripts also contain multiple commands prefixed with `rapids-*`, such as `rapids-print-env`.
These scripts are standard tools used in RAPIDS CI jobs.

## gha-tools

The [gha-tools repo](link) defines a number of bash scripts for tasks common to various RAPIDS builds.
These tools perform tasks like uploading and downloading built artifacts, or configuring [sccache](link). TODO: Do we need to talk about sccache?
These tools are used extensively in the CI scripts, where they are assumed to be available.
As a result of this (and other assumptions made in these scripts), the CI scripts are only safe to run inside the Docker images built for RAPIDS CI.
These images come with the gha-tools installed.

## ci-imgs

The RAPIDS CI Docker images are defined in the [ci-imgs repository](link).
The base Dockerfiles for the images install core dependencies required for all RAPIDS builds.

For wheels, the Dockerfile installs the necessary version of Python using [pyenv](link), while all the main build dependencies (compilers, CMake, system libraries, etc) are installed using the OS's package manager.
RAPIDS attempts to build wheels that are as portable as possible (in compliance with the [manylinux standard](link)), which means that the binaries must be compiled on the oldest supported toolchain and bundle all external library dependencies so as to not throw inscrutable errors when installed onto a user system that is missing those libraries or has ABI-incompatible versions of those libraries installed.
Note that we do not use the upstream [PyPA manylinux images](link).
Those do not contain CUDA, and attempting to maintain a fork of those images with CUDA proved quite challenging.
Instead, we select an OS that we know has the glibc version associated with the manylinux standard that we wish to comply with, then choose [NVIDIA Docker images](link) for that OS as the basis of our CI images.
An appropriate OS by manylinux version may be found using [this handy website](https://github.com/mayeut/pep600_compliance).

For wheels, separate images are created for build and testing.
While the build images contain all libraries required for building RAPIDS, the test image is slimmed down and is missing some of these libraries.
This allows us to test (at least to some extent) our manylinux compliance by verifying that our packages run even without those extra libraries installed.
In an ideal world the testing image would not even have the CTK installed, but that is not currently possible because RAPIDS has some dependencies (numba and cupy) that are not as self-contained and still rely on a system CUDA installation.
Until that changes, we cannot remove the CTK from wheel testing images.

For conda, the system is different.
CUDA installations from conda prior to CUDA 12 were incomplete.
Users were required to install some parts of the CTK (most notably, compilers) on the system because they were not available from conda.
With CUDA 12, the entire CTK became fully available via conda-forge (TODO: Link to the new CUDA cf docs once they're up).
Therefore, in RAPIDS conda CI images, for CUDA 11 images the CTK is installed on the system, while for CUDA 12 images it is not.
The installation of conda itself in our CI images is handled upstream in the miniforge-cuda images.

The ci-imgs repository is automatically rebuild whenever new changes are pushed to any of the following:

- dfg
- miniforge-cuda
- gha-tools

This ensures that the images always have the latest version of everything required by our CI build scripts.

### miniforge-cuda

Our conda CI images are based on the [miniforge-cuda images](link), which we also build.
These images are based off of [nvidia/cuda](link) Docker images, like the wheel images.
They are a minimal installation of miniforge on top of the CUDA images to enable conda in systems with the CTK installed.

## Workflows

### Runners

RAPIDS runs Github actions on [a set of self-hosted runners at NVIDIA](https://docs.gha-runners.nvidia.com/).

### Shared workflows

Our CI pipelines are built around a collection of [shared workflows](link) stored in the [shared-workflows repository](https://github.com/rapidsai/shared-workflows/).
This repository defines a number of common workflows used throughout RAPIDS for building and testing C++ and Python packages.
The primary logic encapsulated in these shared workflows is around behaviors that we want standardized across RAPIDS: the set of CUDA/Python versions and architectures that we want tested, the Docker images that we want used for builds, etc.
The workflows do some standardized setup for these purposes, then delegate back to the scripts in each repository's ci/ directory.
This allows each repository to largely have complete control over what a CI run looks like, while centralizing core information around things like the RAPIDS support matrix that need to be maintained uniformly.

TODO: Should we document the custom-job? Is there anything specific worth including?

#### Matrix selection

An important bit of logic in the shared workflows is the matrix on which we build and test packages.
In general, the build matrix is identical across all workflows since we always want to produce the artifacts, but the test matrix typically only includes a subset.
The reason is that test jobs run on runners with GPUs, while build jobs can run on runners with only CPUs, making build jobs far cheaper.
For PRs, we therefore run a substantially limited subset of test jobs.
To balance coverage of our matrix with resource usage, the selection is typically based on trying to maximize coverage across the different axes of interest: CUDA version, Python version, OS, CUDA driver version, and architecture.

### pr.yaml

Each repository contains a pr.yaml workflow that defines what happens when a pull request is created or has code pushed to it.
This is the typical entrypoint for most developers into our CI.
Jobs in pr.yaml should set their `build_type` to `pull-request`, and should otherwise generally be deferring to the shared workflows repo.
They may specify a path to a script to run; each shared workflow has a default, but repositories with multiple packages may need separate scripts for each package, for example.

### build.yaml, test.yaml, and the nightly pipeline

Each repository also contains two additional workflow files, build.yaml and test.yaml.
These files are used in less familiar scenarios.
To understand their purpose, we must first consider the RAPIDS nightly pipeline.
Every night, we run a nightly pipeline stored in [this repository](https://github.com/rapidsai/workflows/).
This workflow builds and tests every RAPIDS project.

The nightly pipeline is built around the two workflow files described above.
First each repository's build.yaml is invoked (via [workflow dispatch](link)) to build all of that repository's packages.
Then, the test.yaml is invoked to run the tests on those packages.
Typically, the nightly pipeline runs a much more complete test suite than the pr.yaml file, running across the full matrix of supported versions rather than a representative subset.
In both of these cases, the `build_type` is set to `nightly`.

In addition to the nightly pipeline, the build.yaml script serves another purpose.
This workflow runs whenever any code is merged to a RAPIDS release branch `branch-*`.
In this case, the `build_type` is initially empty, and every job in this workflow should specify `build_type: ${{ inputs.build_type || 'branch' }}` so that it defaults to a `branch` build.
The purpose of a branch build is to build and publish a package each time a PR is merged.
That way, we always have artifacts available of the latest versions of our packages.

## devcontainer builds

In addition to building the conda and pip packages, RAPIDS CI also builds the packages in [devcontainers](link).
devcontainers provide a standardized development environment for users.
[The RAPIDS devcontainers](link) leverage the standard to produce containers that make it easy to develop RAPIDS projects.
These containers leverage [sccache](link) (TODO: probably needs to be discussed separately from everything else) to reduce compilation time.
To populate the cache, package builds are also done in CI.

RAPIDS has both pip and conda devcontainers, allowing users to develop in whichever environment they prefer.
Note that, in order to utilize sccache effectively, pip devcontainers turn off build isolation and generate dependencies using dfg.
Therefore, a project's dependencies.yaml must contain entries for requirements.txt file generation that include the appropriate suffixes. TODO: This probably should just be discussed in the dependencies section.


# Release

This section covers miscellaneous tasks to do with the release.

## ptxcompiler and cubinlinker

In addition to the standard RAPIDS packages, RAPIDS also published ptxcompiler and cubinlinker Python packages.
These are CUDA 11-specific tools to enable [MVC](link) for JIT-compiled code using numba.
[ptxcompiler is an open-source but deprecated package](https://github.com/rapidsai/ptxcompiler), scheduled for removal when RAPIDS drops CUDA 11 support.
cubinlinker, on the other hand, is a private repository maintained internally due to its use of CUDA internals.

At present, conda packages for ptxcompiler are built and published via [this conda-forge feedstock](https://github.com/conda-forge/ptxcompiler-feedstock).
conda packages for cubinlinker have generally been built manually and uploaded.
Wheels for both ptxcompiler and cubinlinker are built and uploaded using [this script](link) in the [pypi-wheel-scripts repository](link).

## pynvjitlink

The CUDA 12 replacement for ptxcompiler and cubinlinker is [pynvjitlink](link), which uses the public CUDA [nvjitlink](link) APIs for the purpose of enabling MVC with PTX code.
This package is built and managed in CI similarly to other RAPIDS packages.
However, it is not versioned like other RAPIDS packages.
TODO: Should we talk about versioning and the branching model? Probably just link to the maintainer docs for branching, but yes to versioning.
It follows a traditional Semantic Versioning model.
It may eventually be moved out of RAPIDS and to an owner that reflects its broader utility.


## dask

RAPIDS uses dask extensively.
However, dask tends to make breaking changes fairly frequently, so RAPIDS tries to track the latest version of dask at all times during development.
A dask version is pinned during the release.
This process is handled using the [rapids-dask-dependency package](link), which is depended upon by every other RAPIDS package using dask and manages the pinning.
In addition to functioning as a dask metapackage, this package also supports patching dask behaviors where it is necessary for RAPIDS copmatibility.
