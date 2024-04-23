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
Additionally, there is some manual modification done during wheel builds, [see below](#pyproject).
However, these issues should be addressed over time.

A couple of notes are in order:

- Since pip and conda are independent package managers, a project's dependencies may not be the same on pip and conda.
- Similarly, a project may have a different set of dependencies on pip than on conda.
- While pyproject outputs in `dependencies.yaml` are typically not suffixed, `requirements.txt` outputs must be. This is an artifact of the wheel building process relying on finding and replacing the unsuffixed names, and of the fact that [DFG did not originally support matrix entries for pyproject outputs](https://github.com/rapidsai/dependency-file-generator/pull/70). [This discrepancy will be addressed when RAPIDS switches to using the rapids-build-backend](#pyproject).

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

This difficulty is the reason why [the section on building Python packages recommended turning off build isolation](building.md#python-packages).
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
