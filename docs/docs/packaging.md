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
As of this writing, this is not quite true because [DFG does not yet support meta.yaml or versions.json files](https://github.com/rapidsai/dependency-file-generator/issues/7).
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

Starting with CUDA 12, [the entire CUDA toolkit is available from conda-forge](https://github.com/conda-forge/staged-recipes/issues/21382) (TODO: link to conda-forge CUDA docs when they are available).
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
These suffixes are part of the package name, but they don't change the import names of the package.
Each package will contain roughly the same contents, just built with different CUDA versions.
In other words, `cudf-cu11` and `cudf-cu12` should not be installed together in the same environment, because they will clobber each other and cause unpredictable results.

The default dependency list specified in `pyproject.toml` for our packages does not contain the necessary suffix, and will therefore be incorrect.
However, the correct dependency list is generated at wheel-building time using [`rapids-build-backend`](https://github.com/rapidsai/rapids-build-backend/), a PEP 517
build backend wrapper that provides the glue between `rapids-dependency-file-generator` and the build backend (typically `scikit-build-core` or `setuptools`).

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

## `dependencies.yaml` Conventions

Dependency lists (under the `dependencies` top-level key) shall have the
following conventions:

- Build-time dependencies intended for the `build-system.requires` table shall
  be under a dependency list named `py_build_<wheel_name>`. This should only be
  packages that are required for `rapids-build-backend` to run and generate
  wheel metadata. For example:
    - `rapids-build-backend` itself
    - whatever provides the build backend indicated in
      `tool.rapids-build-backend.build-backend` (e.g.`scikit-build-core` or
      `setuptools`)
    - if the project uses a `setup.py`, and libraries that are imported within
      that `setup.py`
- Build-time dependencies intended for the `tool.rapids-build-backend.requires`
  table shall be under a dependency list named `py_rapids_build_<wheel_name>`.
  This should be packages that are needed to create the files that end up in
  the wheel. For example:
    - build tools like `ninja` and `cmake`
    - `Cython`
    - packages providing headers / libraries used during compilation (e.g.
      `librmm`, `libcudf`)
- Runtime dependencies (under `project.dependencies`) shall be under a
  dependency list named `py_run_<wheel_name>`.
- Test dependencies (under `project.optional-dependencies.test`) shall be under
  a dependency list named `py_test_<wheel_name>`.
- CUDA-version-specific dependencies on other RAPIDS packages with a `-cu*`
  suffix shall have a `cuda_suffixed: "true"` parameter in their matrix entry.
  The fallback entry shall have the list of RAPIDS dependencies without their
  `-cu*` suffix to serve as documentation in the source-controlled
  `pyproject.toml`, and so that use cases that build the projects without the
  `-cu*` suffix (such as DLFW) get the unsuffixed dependencies.
- CUDA wheel dependencies (`nvidia-cublas-cu12`, etc.) shall be under a
  dependency list named `cuda_wheels`. The CUDA-version-specific dependencies
  (with the `-cu*` suffix) shall be under a `specific` matrix entry with
  `cuda_version` and `use_cuda_wheels: "true"` parameters. This shall be
  followed by a matrix entry with `use_cuda_wheels: "false"` and no packages,
  to ensure that use cases like DLFW and devcontainers don't get CUDA wheels.
  The fallback matrix entry shall contain all of the wheels without the `-cu*`
  suffix. This list will never be used in an actual build, but ensures that
  the default CUDA wheels that appear in `pyproject.toml` in source control
  represent a CUDA-version-neutral version of the list of needed CUDA wheels.

    Example:

    ```
    cuda_wheels:
      specific:
        - output_type: pyproject
          matrices:
            - matrix:
                cuda_version: "12.*"
                use_cuda_wheels: "true"
              packages:
                - nvidia-cublas-cu12
            # CUDA 11 does not provide wheels, so use the system libraries instead
            - matrix:
                cuda_version: "11.*"
                use_cuda_wheels: "true"
              packages:
            # if use_cuda_wheels=false is provided, do not add dependencies on any CUDA wheels
            # (e.g. for DLFW and pip devcontainers)
            - matrix:
                use_cuda_wheels: "false"
              packages:
            # if no matching matrix selectors passed, list the unsuffixed packages
            # (just as a source of documentation, as this populates pyproject.toml in source control)
            - matrix:
              packages:
                - nvidia-cublas
    ```
- `requirements` or `pyproject` dependencies that require indices other than pypi.org should include
  those indices in `dependencies.yaml`.

    Example:

    ```
    run_pylibcudf:
      common:
        - output_types: requirements
          packages:
            # pip recognizes the index as a global option for the requirements.txt file
            - --extra-index-url=https://pypi.nvidia.com
            - --extra-index-url=https://pypi.anaconda.org/rapidsai-wheels-nightly/simple
      specific:
        - output_types: [requirements, pyproject]
          matrices:
            - matrix: {cuda: "12.*"}
              packages:
                - cuda-python>=12.0,<13.0a0
            - matrix: {cuda: "11.*"}
              packages: &run_pylibcudf_packages_all_cu11
                - cuda-python>=11.7.1,<12.0a0
            - {matrix: null, packages: *run_pylibcudf_packages_all_cu11}
    ```
- Dependencies appearing in several lists  Dependencies with complex requirements such as extra index URLs or package names that vary by CUDA version (like `-cuXX` suffixes) or format (different conda and PyPI names) should be in their own standalone `depends_on_{project}` lists. Simpler dependencies should use YAML anchors to avoid duplication of the pinning specs.

    Example:

    ```
    files:
      all:
        output: conda
        matrix:
          cuda: ["11.8", "12.5"]
          arch: [x86_64]
        includes:
          # ...
          - depends_on_rmm
          - py_build
          - py_run
          # ...
      py_rapids_build_cuspatial:
        output: [pyproject]
        pyproject_dir: python/cuspatial
        extras:
          table: tool.rapids-build-backend
          key: requires
        includes:
          # ...
          - depends_on_rmm
          - py_build
          # ...
      py_run_cuspatial:
        output: [pyproject]
        pyproject_dir: python/cuspatial
        extras:
          table: project
        includes:
          # ...
          - depends_on_rmm
          - py_run
          # ...
    dependencies:
      depends_on_rmm:
        common:
          - output_types: conda
            packages:
              - &rmm_unsuffixed rmm==24.12.*,>=0.0.0a0
          - output_types: requirements
            packages:
              # pip recognizes the index as a global option for the requirements.txt file
              - --extra-index-url=https://pypi.nvidia.com
              - --extra-index-url=https://pypi.anaconda.org/rapidsai-wheels-nightly/simple
        specific:
          - output_types: [requirements, pyproject]
            matrices:
              - matrix:
                  cuda: "12.*"
                  cuda_suffixed: "true"
                packages:
                  - rmm-cu12==24.12.*,>=0.0.0a0
              - matrix:
                  cuda: "11.*"
                  cuda_suffixed: "true"
                packages:
                  - rmm-cu11==24.12.*,>=0.0.0a0
              - {matrix: null, packages: [*rmm_unsuffixed]}
      py_build:
        common:
          - output_types: [conda, pyproject, requirements]
            packages:
              - &numpy numpy>=1.23,<3.0.0a0
      py_run:
        common:
          - output_types: [conda, pyproject, requirements]
            packages:
              - *numpy
              - scipy>=1.14
    ```
