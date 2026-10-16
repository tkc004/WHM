# Installation and dependencies

## Supported setup

The tested workflow uses Python 3.11 or 3.12, a Fortran compiler, and a
native Apple Silicon environment on macOS. Linux should work with the normal
CAMB compiler toolchain, but the macOS library notes below are not applicable.

All commands in this document are run from the repository root unless stated
otherwise:

```bash
cd /Users/tsangkeungchan/Programs/WHM/WHM
```

## Local virtual environment

The simplest isolated setup is a repository-local virtual environment. The
`.venv/` and `*.venv/` patterns are ignored by Git.

```bash
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip 'setuptools<81' wheel
.venv/bin/pip install numpy scipy matplotlib jupyter cython pyfftw
```

`setuptools<81` is intentional: the CAMB build imports `pkg_resources` while
compiling the extension.

Build the modified local CAMB:

```bash
.venv/bin/pip install -e ./WHM-CAMB --no-build-isolation
```

Never replace this with `pip install camb` when testing WHM. The published
package does not contain the repository's Brieden/WHM modes.

## Full notebook dependencies

```bash
.venv/bin/pip install euclidemu2 baccoemu \
  'git+https://github.com/sfschen/velocileptors' \
  'git+https://github.com/FLAMINGOSIM/FlamingoBaryonResponseEmulator'
```

The first construction of `baccoemu.Matter_powerspectrum()` downloads several
emulator files. A network connection is therefore required even after the
Python packages are installed.

## Conda alternative

The pinned environment specification is:

```bash
conda env create -f WHM-notebooks/env/environment.yml
conda activate whm
pip install -e ./WHM-CAMB --no-build-isolation
```

The helper script combines environment creation, package installation, local
CAMB compilation, and a smoke test:

```bash
bash WHM-notebooks/env/setup_whm_env.sh
```

The helper defaults to native `osx-arm64`. Its documented fallback is:

```bash
WHM_ARCH=osx-64 bash WHM-notebooks/env/setup_whm_env.sh
```

## CLASS and `classy`

`classy` is required by the main paper notebook but is deliberately kept as a
separate external build. The documented version is CLASS 3.3.4:

```bash
git clone --depth 1 --branch v3.3.4 \
  https://github.com/lesgourg/class_public /tmp/class_public
make -C /tmp/class_public -j4 class
make -C /tmp/class_public libclass.a
```

Install its Python wrapper into the same environment:

```bash
CC=/usr/bin/clang CXX=/usr/bin/clang++ \
SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk \
MACOSX_DEPLOYMENT_TARGET=15.0 \
.venv/bin/pip install /tmp/class_public --no-build-isolation
```

The explicit Apple compiler selection avoids Homebrew LLVM linker issues on
some macOS installations.

## macOS native libraries

EuclidEmulator2 may require GSL and OpenBLAS:

```bash
brew install gsl openblas
```

OpenBLAS is normally keg-only. Use the fallback path when running Python:

```bash
export DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/opt/openblas/lib:/opt/homebrew/opt/gsl/lib
```

Prefer this over `DYLD_LIBRARY_PATH`: forcing OpenBLAS globally can override
NumPy's Accelerate-linked BLAS symbols and make NumPy fail to import.

If CAMB reports `ld: library 'System' not found`, rebuild it with the active
SDK:

```bash
SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk \
MACOSX_DEPLOYMENT_TARGET=15.0 \
.venv/bin/pip install -e ./WHM-CAMB --no-build-isolation
```

## Verification

Run the smoke test:

```bash
mkdir -p /tmp/whm-cache
MPLCONFIGDIR=/tmp/whm-cache \
XDG_CACHE_HOME=/tmp/whm-cache \
DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/opt/openblas/lib:/opt/homebrew/opt/gsl/lib \
.venv/bin/python WHM-notebooks/env/import_smoketest.py
```

To make missing CLASS a hard failure:

```bash
WHM_REQUIRE_CLASSY=1 \
MPLCONFIGDIR=/tmp/whm-cache \
XDG_CACHE_HOME=/tmp/whm-cache \
DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/opt/openblas/lib:/opt/homebrew/opt/gsl/lib \
.venv/bin/python WHM-notebooks/env/import_smoketest.py
```

The successful result must end with `SMOKETEST PASSED`.
