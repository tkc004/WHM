# Troubleshooting

## CAMB imports the wrong package

Check the imported path:

```bash
.venv/bin/python -c "import camb; print(camb.__file__)"
```

It must point to `WHM-CAMB/camb`. If it points to site-packages from a normal
PyPI installation, reinstall the local package:

```bash
.venv/bin/pip uninstall camb
.venv/bin/pip install -e ./WHM-CAMB --no-build-isolation
```

## `pkg_resources` or setuptools build failure

The local CAMB build expects `pkg_resources`:

```bash
.venv/bin/pip install 'setuptools<81' wheel
```

Then rebuild with `--no-build-isolation` so the pinned setuptools is used.

## `ld: library 'System' not found` on macOS

The compiler may be using a stale SDK path. Rebuild with:

```bash
SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk \
MACOSX_DEPLOYMENT_TARGET=15.0 \
.venv/bin/pip install -e ./WHM-CAMB --no-build-isolation
```

Confirm the SDK exists with:

```bash
xcrun --show-sdk-path
```

## EuclidEmulator2 cannot load GSL or OpenBLAS

Install the native libraries:

```bash
brew install gsl openblas
```

Run Python with the fallback library path:

```bash
DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/opt/openblas/lib:/opt/homebrew/opt/gsl/lib \
.venv/bin/python -c "import euclidemu2; print('ok')"
```

Do not use `DYLD_LIBRARY_PATH` globally with NumPy on macOS. It can cause
Accelerate/OpenBLAS symbol conflicts such as
`_cblas_caxpy$NEWLAPACK$ILP64`.

## `baccoemu` cannot download emulator data

The package may import successfully while `Matter_powerspectrum()` fails on
its first construction. Run that test with network access:

```bash
.venv/bin/python -c \
  "import baccoemu; baccoemu.Matter_powerspectrum(); print('bacco ok')"
```

After the data are downloaded, subsequent runs normally use the local cache.

## `classy` build fails with missing C++ headers or `libc++`

Use Apple's system compilers rather than Homebrew LLVM:

```bash
CC=/usr/bin/clang CXX=/usr/bin/clang++ \
SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk \
MACOSX_DEPLOYMENT_TARGET=15.0 \
.venv/bin/pip install /tmp/class_public --no-build-isolation
```

The CLASS C library can still be built with `make`; only the Python wrapper
needs the compiler override.

## Jupyter reports `Operation not permitted` when starting a kernel

The Jupyter kernel opens local TCP sockets. Execute `nbconvert` in an
environment that permits local kernel networking. If the execution environment
has a sandbox, run the command with the required local-network permission.

## Notebook cannot find data files

Run from the notebook directory:

```bash
cd WHM-notebooks
```

Do not execute `WHMcode.ipynb` from the repository root unless the notebook's
relative paths have been changed.

## LaTeX errors while saving figures

The notebooks use Matplotlib's `usetex=True`. Install a LaTeX distribution and
check that the executable is visible:

```bash
command -v latex
command -v pdflatex
```

## Smoke test failure summary

Run the complete test and address the first missing dependency reported:

```bash
MPLCONFIGDIR=/tmp/whm-cache \
XDG_CACHE_HOME=/tmp/whm-cache \
DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/opt/openblas/lib:/opt/homebrew/opt/gsl/lib \
.venv/bin/python WHM-notebooks/env/import_smoketest.py
```

Use `WHM_REQUIRE_CLASSY=1` when validating the full notebook environment.
