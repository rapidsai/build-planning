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
