# Building libraries

RAPIDS packages are typically composed of C++ and Python components.

## CUDA/C++

The CUDA and C++ libraries in RAPIDS are built using [CMake](https://cmake.org/).
The main common dependencies of all of RAPIDS are the CUDA toolkit itself, the [Google testing framework](http://google.github.io/googletest/) for tests, and both the [Google benchmark](https://github.com/google/benchmark) and [NVBench](https://github.com/NVIDIA/nvbench) tools for benchmarking.
All other dependencies vary from library to library.

Build-time dependencies are managed using [`CPM.cmake`](https://github.com/cpm-cmake/CPM.cmake), a package manager for CMake that enables downloading dependencies and building them from source if they cannot be found.
This feature allows RAPIDS libraries to be built on systems where dependencies are not already installed.
RAPIDS wraps `CPM` calls using [`rapids-cmake`](https://github.com/rapidsai/rapids-cmake), a set of CMake modules written for RAPIDS to encapsulate common CMake logic, to also provide proper dependency tracking and propagation to consumers.
In addition, `rapids-cmake` also provides a number of additional common features needed across RAPIDS.

RAPIDS C++ libraries can be built like any other CMake project using the standard commands:
```
cmake -S . -B build -GNinja
cmake --build build
```
Every library's CMake handles downloading `rapids-cmake` (usually by a `fetch_rapids.cmake` or `rapids_config.cmake` file at the repository root.
Note that consumers are still generally responsible for having all other dependencies installed.
Dependency installation is covered later in [the dependency section](link).

## Pure Python packages

RAPIDS C++ libraries are generally exposed via Python wrappers.
These wrapper libraries contain Python [extension modules](https://docs.python.org/3/extending/extending.html), Python modules written in a compiled language that are exposed to Python using the [CPython C API](https://docs.python.org/3/c-api/index.html).
The standard approach used in RAPIDS is to write these modules using [Cython](https://docs.cython.org/en/latest/).
Cython is a compiler that supports a superset of Python, allowing far more efficient code generation via static typing and removing Python overheads.
Cython also offers a way to write foreign function interfaces (FFI) in Python, allowing you to call C or C++ (or FORTRAN) functions from Python via Cython wrappers.

The Cython compiler generates C code using the Python C API from Cython code, and then compiles that code down to a native library.
Traditionally, Python packages are built using [`setuptools`](https://setuptools.pypa.io/en/latest/), and Cython hooks directly into this process [with its own `setuptools` plugins](https://cython.readthedocs.io/en/latest/src/quickstart/build.html#building-a-cython-module-using-setuptools).
With the modern, more complex builds that have become common and are present throughout RAPIDS, we prefer to instead handle the full compilation process with CMake.
`setuptools` is not a proper build system, and compiling complex extensions with `setuptools` easily becomes intractable when complex dependency chains must be managed or compiler flags must be carefully controlled.

To integrate CMake builds with Cython, we use [`scikit-build-core`](https://scikit-build-core.readthedocs.io), a standards-compliant [PEP 517 builder](https://peps.python.org/pep-0517/) that integrates Python builds with CMake.
This allows us to build our Python projects using CMake to seamlessly integrate the Python extension modules with a build of the C++ library.
We define numerous helpful modules for Cython compilation in rapids-cmake.

Some RAPIDS packages are pure Python and simply depend on other packages for compiled functionality.
Those packages do not contain any Cython or CMake, and instead using `setuptools`.

To build and install Python packages, you follow the same approach used to build any normal Python package:
```
python -m pip install --no-build-isolation /path/to/project
```
where `/path/to/project` is the path to a directory containing a `pyproject.toml` file.
Note the specification of `--no-build-isolation` above; this assumes that you have already installed all the Cython and C++ build dependencies of your project into your environment.
Currently this is a requirement of RAPIDS builds, which are not able to correctly specify the Python package requirements, [but this will be fixed in the future with the rapids-build-backend](link to section).

## build.sh scripts

Eevry RAPIDS repository contains a `build.sh` script at the root of the repo.
This script provides a convenient interface for building all of the components of the project through a single command `./build.sh ...`.
This script should be kept up-to-date when new (C++ or Python) components are added to a library, when CMake options are changed, etc.
`build.sh` is seldom more than a trivial wrapper around the `cmake` and `pip` commands listed above.
However, it does set some convenient environment variables and/or CMake flags to ensure that the user gets a compatible set of installations on their local machine.
This may include setting the CUDA architectures to build for, or ensuring that the Python package build is provided the path to the C++ package build.

# Creating packages

RAPIDS is generally delivered to users either via packages or in containers.
The various container options for getting RAPIDS are out of scope for this document.
Here we focus on the two kinds of package distributions built in RAPIDS: conda and pip packages.

## Dependencies

RAPIDS projects need to specify dependencies in multiple ways:

- conda package files (`meta.yaml`)
- pip package files (`pyproject.toml`)
- conda environments for end users (`environment.yaml`)
- pip environments for end users (`requirements.txt`)
- CMake versions during the build (`versions.json`)

In some cases we may want these files persisted to disk, while in others the outputs may be transient and fed directly to a package manager (e.g. via `conda env create`).
Maintaining package versions in all of these separate locations is tedious and error-prone.
Moreover, as support for RAPIDS expands to encompass more use cases, more such formats may be added.

RAPIDS automates this process using the [rapids-dependency-file-generator](https://github.com/rapidsai/dependency-file-generator/) (DFG) tool.
DFG defines a specification for a single file, `dependencies.yaml`, that lives at the root of each RAPIDS repository.
This file is intended as a single source of truth for all the dependencies of a project.
As of this writing, this is not quite because [DFG does not yet support meta.yaml or versions.json files](https://github.com/rapidsai/dependency-file-generator/issues/7).
Additionally, there is some manual modification done during wheel builds, [see below](link).
However, these issues should be addressed over time.

A couple of notes are in order:

- Since pip and conda are independent package managers, a project's dependencies may not be the same on pip and conda.
- Similarly, a project may have a different set of dependencies on pip than on conda.
- While pyproject outputs in `dependencies.yaml` are typically not suffixed, `requirements.txt` outputs must be. This is an artifact of the wheel building process relying on finding and replacing the unsuffixed names, and of the fact that [DFG did not originally support matrix entries for pyproject outputs](https://github.com/rapidsai/dependency-file-generator/pull/70). [This discrepancy will be addressed when RAPIDS switches to using the rapids-build-backend](link to section).

## conda

[conda](https://docs.conda.io/en/latest/) is a cross-platform, language-agnostic (although originally developed for Python) package manager.
A major strength of conda is its ability to specify native library dependencies of Python packages, making it possible to acquire almost the entire dependency tree of RAPIDS projects.
conda packages are specified using [recipes, in the form of `meta.yaml` files](https://docs.conda.io/projects/conda-build/en/latest/resources/define-metadata.html).
By convention, RAPIDS stores recipes in the `conda/recipes/${PACKAGE_NAME}` directory.
Some package versions are directly specified in the recipe, while others are specified in the [`conda_build_config.yaml`](https://docs.conda.io/projects/conda-build/en/stable/resources/variants.html) (CBC) file.
All variables defined in that file are directly available in the recipe.
The actual build logic for a package is controlled using the `build.sh` script in the recipe directory.
Typically, recipe `build.sh` scripts simply invoke the repository's top-level build.sh script with a specific set of arguments.
This allows both Python and C++ conda packages to follow the same process and simply request that the top-level `build.sh` script build the right package.

Starting with CUDA 12, [the entire CUDA toolkit is available from conda-forge](https://github.com/conda-forge/staged-recipes/issues/21382) (Note: link to conda-forge CUDA docs when they are available).
Therefore, RAPIDS packages rely entirely on those packages, with the main exception being that the host system must have the CUDA driver library installed.
Prior to CUDA 12, the CTK experience on conda was fragmented, with the [`cudatoolkit`](https://anaconda.org/conda-forge/cudatoolkit) conda-forge package being the most commonly used in the ecosystem while the more officially supported packages on the nvidia channel had a different structure, and neither provided compilers.
RAPIDS recipes have conditional logic littered throughout to support both CUDA 11 and CUDA 12 builds.
This logic will be dramatically simplified when we drop CUDA 11 support.

## pyproject

Python packages are configured via [`pyproject.toml` files](https://packaging.python.org/en/latest/specifications/pyproject-toml/).
A typical RAPIDS repository places Python packages at `python/${PACKAGE_NAME}`, so that directory will contain the `pyproject.toml` file.
In addition to the project metadata indicated in the linked specification (analogous to the metadata in the conda `meta.yaml` recipe), `pyproject.toml` is also designed for use as a general configuration file for arbitrary Python tools related to a project.
The most common use case of this is for the build backend; `scikit-build-core` is configured in the `tool.scikit-build` table.

When building our Python packages for distribution via pip, we need to make some additional modifications.
Python packages distributed for pip are called [wheels](https://packaging.python.org/en/latest/guides/distributing-packages-using-setuptools/#wheels).
Since pip is only a Python package manager, not a general package manager, it has no knowledge of native dependencies nor CUDA (for a more detailed discussion of the various limitations in the wheels space, see [the pypackaging-native website](https://pypackaging-native.github.io/)).
These limitations mean that we must create a separate wheel for every CUDA version we want a package to support, and that package's dependencies must also be appropriately updated to match that CUDA version.
To give a concrete example, we publish `cudf-cu11` (CUDA 11) and `cudf-cu12` (CUDA 12) wheels, and these depend on `rmm-cu11` and `rmm-cu12`, respectively.

This difficulty is the reason why [the section on building Python packages recommended turning off build isolation](link).
The default dependency list specified in `pyproject.toml` for our packages does not contain the necessary suffix, and will therefore be incorrect.
However, the correct dependency list may be generated using DFG, and that `requirements.txt` file can be installed into the environment by the developer.
When we build wheels for distribution, we modify the metadata in `pyproject.toml` directly before building the package to address the above concerns.
These modifications can be seen in our CI scripts: for instance, [you can see that cudf changes `rmm` to `rmm-cu11` or `rmm-cu12` with sed in its wheel building scripts](https://github.com/rapidsai/cudf/blob/branch-24.06/ci/build_wheel.sh#L42).
If developers wish to build and/or install a specific RAPIDS wheel locally, they will have to make similar modifications.
In the future, this will be fixed with the [`rapids-build-backend`](https://github.com/rapidsai/rapids-build-backend/) a PEP 517 build backend wrapper that essentially provides the glue between DFG and the underlying build backend (typically `scikit-build-core` or `setuptools`), ensuring that when a Python package is built the appropriate suffixes are added.

## Nightlies

RAPIDS packages are distributed both as nightlies and as final releases.
The term "nightly" is somewhat misleading; new RAPIDS packages are in fact built and published every time a PR is merged.
conda packages go to the [rapidsai-nightly conda channel](https://anaconda.org/rapidsai-nightly/), while pip wheels go to the [nightly simple index that we host via anaconda](https://anaconda.org/rapidsai-wheels-nightly/repo?type=pypi&label=main).

The distribution and usage of nightlies causes additional difficulties for managing dependencies of wheels.
Due to the frequency of changes in RAPIDS, RAPIDS packages depend on nightly versions of other RAPIDS packages.
With conda, dependency constraints of the form `X.Y.*` will permit the installation of nightly packages as well as releases.
pip is stricter and will not generally install nightlies unless the `--pre` argument is specified.
However, pip can be instructed to install nightlies of specific packages by [using any specifier containing a pre-release or development version](https://pip.pypa.io/en/stable/cli/pip_install/#pre-release-versions).
Therefore, during wheel builds we also do this [in our CI scripts by adding a `>=0.0.0a0` specifier to all the relevant constraints](https://github.com/rapidsai/cudf/blob/branch-24.06/ci/build_wheel.sh#L34).


# CI and Deployment

All RAPIDS packages are built using the standard infrastructure leveraged by CI.
Our CI infrastructure is a complex network of different pieces, built around [Github Actions](https://docs.github.com/en/actions) workflows running on custom containers.

## CI scripts

The logic for running RAPIDS CI jobs is encoded in scripts that are placed in the `ci/` directory of every repository.
These include scripts for building packages, running test, linting code, and other more specialized tasks.
While each repository is free to define whatever scripts it requires, most RAPIDS repositories share at least some common scripts:

- `check_style.sh`: This script runs style checks. In general, RAPIDS projects use [`pre-commit`](https://pre-commit.com/) for style checks, so the style checking script is usually little more than a pre-commit invocation.
- `build_cpp.sh`: Builds the conda C++ library.
- `build_python.sh`: Builds all of the conda Python packages.
- `build_wheel_${lib}.sh`: Build the wheel `${lib}`. May be multiple per repo.
- `test_cpp.sh`: Runs test on the conda C++ library.
- `test_python.sh`: Runs test on all the conda Python packages.
- `test_wheel_${lib}.sh`: Test the wheel `${lib}`.

Some repositories will include helper scripts in this directory to share logic between the above scripts.
This practice is especially common if there are multiple wheels to build since the logic for building various wheels is usually similar.

### Versioning

RAPIDS libraries are versioned using [CalVer](https://calver.org/).
For nightlies, an additional alpha segment is added indicating the number of commits since the last release, e.g. `24.04.00a240` is 240 commits past the `24.02.00` release.
To manage this, every repository contains a single plain-text `VERSION` file at the root of the repository.
This file is symlinked into every directory that might need it for packaging purposes.
[Although other places in RAPIDS currently encode the version, we are currently in the process of making this file the sole source of truth](https://github.com/rapidsai/build-planning/issues/15).
When packages are built in CI, this file is overwritten to reflect the appropriate number of commits since the last release, and then that file is automatically read during package builds.

## gha-tools

The CI scripts also contain multiple commands prefixed with `rapids-*`, such as `rapids-print-env`.
These scripts are standard tools used in RAPIDS CI jobs that are defined in the [gha-tools repo](https://github.com/rapidsai/gha-tools/).
These tools perform tasks like uploading and downloading built artifacts, or configuring [sccache](https://github.com/mozilla/sccache).
The assumption that these tools are available is one reason that the CI scripts are currently only safe to run inside the Docker images built for RAPIDS CI.
These images come with the `gha-tools` installed.

## ci-imgs

The RAPIDS CI Docker images are defined in the [ci-imgs repository](https://github.com/rapidsai/ci-imgs/).
The base Dockerfiles for the images install core dependencies required for all RAPIDS builds.

### wheel images

For wheels, the Dockerfile installs the necessary version of Python using [`pyenv`](https://github.com/pyenv/pyenv), while all the main build dependencies (compilers, CMake, system libraries, etc) are installed using the OS's package manager.
RAPIDS attempts to build wheels that are as portable as possible (in compliance with the [manylinux standard](https://peps.python.org/pep-0513/)), which means that the binaries must be compiled on the oldest supported toolchain and bundle all external library dependencies so as to not throw inscrutable errors when installed onto a user system that is missing those libraries or has ABI-incompatible versions of those libraries installed.
Note that we do not use the upstream [PyPA manylinux images](https://github.com/pypa/manylinux/).
Those do not contain CUDA, and attempting to maintain a fork of those images with CUDA proved quite challenging.
Instead, we select an OS that we know has the glibc version associated with the manylinux standard that we wish to comply with, then choose [NVIDIA Docker images](https://hub.docker.com/r/nvidia/cuda) for that OS as the basis of our CI images.
An appropriate OS by manylinux version may be found using [this handy website](https://github.com/mayeut/pep600_compliance).

For wheels, separate images are created for build and testing.
While the build images contain all libraries required for building RAPIDS, the test image is slimmed down and is missing some of these libraries.
This allows us to test (at least to some extent) our manylinux compliance by verifying that our packages run even without those extra libraries installed.
In an ideal world the testing image would not even have the CTK installed, but that is not currently possible because RAPIDS has some dependencies (numba and cupy) that are not as self-contained and still rely on a system CUDA installation.
Until that changes, we cannot remove the CTK from wheel testing images.

### conda images

For conda, the images are different depending on whether they are for CUDA 11 or for CUDA 12+.
As described in [the conda section above](link) the available packages differed between these versions.
Therefore, in RAPIDS conda CI images, for CUDA 11 images the CTK is installed on the system, while for CUDA 12 images it is not.
The installation of conda itself in our CI images is handled upstream in the miniforge-cuda images (see below).

The ci-imgs repository is automatically rebuild whenever new changes are pushed to any of the following:

- dependency-file-generator
- miniforge-cuda
- gha-tools

This ensures that the images always have the latest version of everything required by our CI build scripts.

#### miniforge-cuda

Our conda CI images are based on the [miniforge-cuda images](https://github.com/rapidsai/miniforge-cuda), which we also build.
These images are based off of [`nvidia/cuda`](https://hub.docker.com/r/nvidia/cuda) Docker images, like the wheel images.
They are a minimal installation of [`miniforge`](https://github.com/conda-forge/miniforge) on top of the CUDA images to enable conda in systems with the CTK installed.

## Workflows

### Runners

RAPIDS runs Github actions on [a set of self-hosted runners at NVIDIA](https://docs.gha-runners.nvidia.com/).

### Shared workflows

Our CI pipelines are built around a collection of [shared workflows](https://docs.github.com/en/actions/creating-actions/sharing-actions-and-workflows-with-your-organization) stored in the [shared-workflows repository](https://github.com/rapidsai/shared-workflows/).
This repository defines a number of common workflows used throughout RAPIDS for building and testing C++ and Python packages.
The primary logic encapsulated in these shared workflows is around behaviors that we want standardized across RAPIDS: the set of CUDA/Python versions and architectures that we want tested, the Docker images that we want used for builds, etc.
The workflows do some standardized setup for these purposes, then delegate back to the scripts in each repository's `ci/` directory.
This allows each repository to largely have complete control over what a CI run looks like, while centralizing core information around things like the RAPIDS support matrix that need to be maintained uniformly.

#### Matrix selection

An important bit of logic in the shared workflows is the matrix on which we build and test packages.
In general, the build matrix is identical across all workflows since we always want to produce the artifacts, but the test matrix typically only includes a subset.
The reason is that test jobs run on runners with GPUs, while build jobs can run on runners with only CPUs, making build jobs far cheaper.
For PRs, we therefore run a substantially limited subset of test jobs.
To balance coverage of our matrix with resource usage, the selection is typically based on trying to maximize coverage across the different axes of interest: CUDA version, Python version, OS, CUDA driver version, and architecture.

### pr.yaml

Each repository contains a `pr.yaml` workflow that defines what happens when a pull request is created or has code pushed to it.
This is the typical entrypoint for most developers into our CI.
Jobs in `pr.yaml` should set their `build_type` to `pull-request`, and should otherwise generally be deferring to the shared workflows repo.
They may specify a path to a script to run; each shared workflow has a default, but repositories with multiple packages may need separate scripts for each package, for example.

### build.yaml, test.yaml, and the nightly pipeline

Each repository also contains two additional workflow files, `build.yaml` and `test.yaml`.
These files are used in less familiar scenarios.
To understand their purpose, we must first consider the RAPIDS nightly pipeline.
Every night, we run a nightly pipeline stored in [this repository](https://github.com/rapidsai/workflows/).
This workflow builds and tests every RAPIDS project.

The nightly pipeline is built around the two workflow files described above.
First each repository's `build.yaml` is invoked (via [workflow dispatch](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)) to build all of that repository's packages.
Then, the `test.yaml` is invoked to run the tests on those packages.
Typically, the nightly pipeline runs a much more complete test suite than the `pr.yaml` file, running across the full matrix of supported versions rather than a representative subset.
In both of these cases, the `build_type` is set to `nightly`.

In addition to the nightly pipeline, the `build.yaml` script serves another purpose.
This workflow runs whenever any code is merged to a RAPIDS release branch `branch-*`.
In this case, the `build_type` is initially empty, and every job in this workflow should specify `build_type: ${{ inputs.build_type || 'branch' }}` so that it defaults to a `branch` build.
The purpose of a branch build is to build and publish a package each time a PR is merged.
That way, we always have artifacts available of the latest versions of our packages.

## devcontainer builds

In addition to building the conda and pip packages, RAPIDS CI also builds the packages in [devcontainers](https://containers.dev/).
devcontainers provide a standardized development environment for users.
[The RAPIDS devcontainers](https://github.com/rapidsai/devcontainers/) leverage the standard to produce containers that make it easy to develop RAPIDS projects.
These containers leverage sccache to reduce compilation time.
To populate the cache, package builds are also done in CI so that local development has a high chance of cache hits during compilation.
RAPIDS has both pip and conda devcontainers, allowing users to develop in whichever environment they prefer.
Note that, in order to utilize sccache effectively, pip devcontainers turn off build isolation and generate dependencies using DFG.


# Release

This section covers miscellaneous tasks to do with the release.

## ptxcompiler and cubinlinker

In addition to the standard RAPIDS packages, RAPIDS also published `ptxcompiler` and `cubinlinker` Python packages.
These are CUDA 11-specific tools to enable [MVC](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#cuda-compatibility-developer-s-guide) for JIT-compiled code using numba.
[ptxcompiler is an open-source but deprecated package](https://github.com/rapidsai/ptxcompiler), scheduled for removal when RAPIDS drops CUDA 11 support.
`cubinlinker`, on the other hand, is a private repository maintained internally due to its use of CUDA internals.

At present, conda packages for `ptxcompiler` are built and published via [this conda-forge feedstock](https://github.com/conda-forge/ptxcompiler-feedstock).
conda packages for `cubinlinker` have generally been built manually and uploaded.
Wheels for both `ptxcompiler` and `cubinlinker` are built and uploaded using [this script](https://github.com/rapidsai/pypi-wheel-scripts/blob/main/cibuildwheel-local-scripts/cibuildwheel_ptxcompiler_cubinlinker.sh) in the `pypi-wheel-scripts` repository.

## pynvjitlink

The CUDA 12 replacement for `ptxcompiler` and `cubinlinker` is [pynvjitlink](https://github.com/rapidsai/pynvjitlink/), which uses the public CUDA [nvjitlink](https://docs.nvidia.com/cuda/nvjitlink/index.html) APIs for the purpose of enabling MVC with PTX code.
This package is built and managed in CI similarly to other RAPIDS packages.
However, it is not versioned like other RAPIDS packages.
It follows a traditional Semantic Versioning model.
It may eventually be moved out of RAPIDS and to an owner that reflects its broader utility.

## dask

RAPIDS uses Dask extensively.
Dask tends to make breaking changes fairly frequently, so RAPIDS tries to track the latest version of Dask at all times during development.
A Dask version is pinned during the release.
This process is handled using the [`rapids-dask-dependency` package](https://github.com/rapidsai/rapids-dask-dependency), which is depended upon by every other RAPIDS package using Dask and manages the pinning.
In addition to functioning as a metapackage, this package also supports patching Dask behaviors where it is necessary for RAPIDS compatibility.

## Release scripts

The official release process is largely handled by the ops team.
There is a some degree of manual work involved.
Some of the relevant scripts involved in this process may be found [in this repository](https://github.com/rapidsai/release-scripts).
