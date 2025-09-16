# Release

This section covers miscellaneous tasks to do with the release.

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
