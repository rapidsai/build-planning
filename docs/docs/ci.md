# CI and Deployment

All RAPIDS packages are built using a standard shared CI infrastructure.
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
As described in [the conda section above](packaging.md#conda) the available packages differed between these versions.
Therefore, in RAPIDS conda CI images, for CUDA 11 images the CTK is installed on the system, while for CUDA 12 images it is not.
The installation of conda itself in our CI images is handled upstream in the miniforge-cuda images (see below).

The [ci-imgs repository is automatically rebuilt whenever new changes are pushed to any of the following:

- [dependency-file-generator](packaging.md#dependencies)
- miniforge-cuda (see below)
- [gha-tools](#gha-tools)

This ensures that the images always have the latest version of everything required by our CI build scripts.

#### miniforge-cuda

Our conda CI images are based on the [miniforge-cuda images](https://github.com/rapidsai/miniforge-cuda), which we also build.
These images are based off of [`nvidia/cuda`](https://hub.docker.com/r/nvidia/cuda) Docker images, like the wheel images.
They are a minimal installation of [`miniforge`](https://github.com/conda-forge/miniforge) on top of the CUDA images to enable conda in systems with the CTK installed.

## Workflows

### Runners

RAPIDS runs Github actions on [a mixture of public and self-hosted runners at NVIDIA](https://docs.gha-runners.nvidia.com/).

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
