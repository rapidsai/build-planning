# Release

This section covers miscellaneous tasks to do with the release.

## JIT and Minor Version Compatibility

A common problem that RAPIDS has to deal with is CUDA version compatibility.
Since CUDA 11, [the CTK promises binary compatibility across all minor release](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#cuda-toolkit-versioning).
However, [PTX code does not have the same guarantees](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#using-ptx).
The linked guides contain complete information on this, and a more brief summary may be found on the [`pypackaging-natives` website](https://pypackaging-native.github.io/key-issues/gpus/).
The main consequence of the difference is that while CUDA code compiled with a CTK X.Y at build time will work on a system with the minimum required driver corresponding to X.0 installed, PTX code gnerated by CTK X.Y may require the driver corresponding to CTK X.Y.
This is problematic for RAPIDS for two reasons:

1. We use numba to JIT kernels. If the user has upgraded their CTK but not their driver, their runtime CTK could produce PTX code that the driver cannot handle.
2. Some parts of RAPIDS actually ship compiled PTX code for specific features. If that PTX code is run on a user's system with an older driver (satisfying the minimum requirement for CUDA code, but not for the PTX), they will see errors at runtime.

To resolve this we have two solutions.
One is the modern CUDA 12 solution, while the other is our legacy CUDA 11 solution.

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

## Archived Projects

### pynvjitlink

One of the new libraries added to the CTK in CUDA 12 was [nvjitlink](https://docs.nvidia.com/cuda/nvjitlink/index.html), which exposed APIs for the purpose of enabling minor version compatibility with PTX code.
RAPIDS created a set of Python bindings for this in [pynvjitlink](https://github.com/rapidsai/pynvjitlink/), which were needed for using JIT-compiled code from numba.

That same functionality was eventually exposed via `cuda-bindings`, `cuda-core`, and `numba-cuda`, making `pynvjitlink` unncessary for CUDA 12 and beyond.

RAPIDS stopped publishing `pynvjitlink` around RAPIDS 25.10, and never added CUDA 13 support to it.

For more history on that, see:

* https://github.com/rapidsai/pynvjitlink/issues/130
* https://github.com/rapidsai/build-planning/issues/210

### ptxcompiler and cubinlinker

Prior to CUDA 12, enabling minor version compatibility for PTX was far more challenging because the necessary public APIs did not exist.
To bridge this gap, we built the `ptxcompiler` and `cubinlinker` packages, the latter of which leveraged internal CUDA APIs to enable this functionality.
These were CUDA 11-specific tools and are not used in CUDA 12 environments.
Development on both stopped when RAPIDS dropped CUDA 11 support (https://github.com/rapidsai/build-planning/issues/184).
