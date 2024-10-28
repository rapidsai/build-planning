# Building libraries

RAPIDS packages are typically composed of C++ and Python components.

## CUDA/C++

The CUDA and C++ libraries in RAPIDS are built using [CMake](https://cmake.org/).
The main common dependencies of all of RAPIDS are the CUDA toolkit itself, the [Google testing framework](http://google.github.io/googletest/) for tests, and both the [Google benchmark](https://github.com/google/benchmark) and [NVBench](https://github.com/NVIDIA/nvbench) tools for benchmarking.
All other dependencies vary from library to library.

Build-time dependencies excluding the CUDA Toolkit are managed using [`CPM.cmake`](https://github.com/cpm-cmake/CPM.cmake), a package manager for CMake that enables downloading dependencies and building them from source if they cannot be found.
This feature allows RAPIDS libraries to be built on systems where dependencies are not already installed.
RAPIDS wraps `CPM` calls using [`rapids-cmake`](https://github.com/rapidsai/rapids-cmake), a set of CMake modules written for RAPIDS to encapsulate common CMake logic, to also provide proper dependency tracking and propagation to consumers.
In addition, `rapids-cmake` also provides a number of additional common features needed across RAPIDS.

RAPIDS C++ libraries can be built like any other CMake project using the standard commands:
```
cmake -S <src_dir> -B build -GNinja
cmake --build build
```
Every library's CMake handles downloading `rapids-cmake` (usually by a `rapids_config.cmake` file at the repository root.
Note that consumers are still generally responsible for having all other dependencies installed.
Dependency installation is covered later in [the dependency section](packaging.md#dependencies).

## Python packages

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

```shell
python -m pip install /path/to/project
```

where `/path/to/project` is the path to a directory containing a `pyproject.toml` file.

By default, this will create an isolated build environment, and `rapids-build-backend` will ensure that the necessary build-time dependencies are installed in that environment.

## build.sh scripts

Eevry RAPIDS repository contains a `build.sh` script at the root of the repo.
This script provides a convenient interface for building all of the components of the project through a single command `./build.sh ...`.
This script should be kept up-to-date when new (C++ or Python) components are added to a library, when CMake options are changed, etc.
`build.sh` is seldom more than a trivial wrapper around the `cmake` and `pip` commands listed above.
However, it does set some convenient environment variables and/or CMake flags to ensure that the user gets a compatible set of installations on their local machine.
This may include setting the CUDA architectures to build for, or ensuring that the Python package build is provided the path to the C++ package build.
