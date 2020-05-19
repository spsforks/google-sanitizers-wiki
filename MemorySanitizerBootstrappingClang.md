# Introduction

[MemorySanitizer](MemorySanitizer) code itself can not be tested with [MemorySanitizer](MemorySanitizer) for obvious reasons, but the rest of Clang/LLVM can. Instructions on this page show how to do it.

This guide presumes you have a recent-enough Clang binary installed on your system to successfully compile LLVM and tools. If you don't, you can download a pre-built toolchain from [the LLVM website](https://releases.llvm.org/download.html) or compile one yourself using [this guide](https://clang.llvm.org/get_started.html).

If you're going to compile Clang yourself, make sure you also build LLD and compiler_rt, as they'll be necessary here.

*Note: The following instructions were adapted from the [buildbot](http://lab.llvm.org:8011/builders/sanitizer-x86_64-linux-bootstrap-msan/) bootstrap process.*

## Building `libcxx` and `libcxxabi`

MSan requires that all code in the process is instrumented (see [handling-external-code](http://clang.llvm.org/docs/MemorySanitizer.html#handling-external-code)). Luckily, the only external dependency of Clang is the C++ standard library (and, of course, libc, but MSan _almost_ takes care of it).

First, clone the LLVM source from github and enter a new build directory:

```bash
git clone --depth=1 https://github.com/llvm/llvm-project
cd llvm-project

LIBCXX_DIR="$PWD/build_libcxx"
mkdir $LIBCXX_DIR
cd $LIBCXX_DIR
```

Second, configure the CMake project:

```bash
cmake -GNinja ../llvm \
	-DCMAKE_BUILD_TYPE=Release \
	-DLLVM_ENABLE_PROJECTS="libcxx;libcxxabi" \
	-DCMAKE_C_COMPILER=clang \
	-DCMAKE_CXX_COMPILER=clang++ \
	-DLLVM_USE_SANITIZER=MemoryWithOrigins
```

This will generate a [Ninja](https://ninja-build.org/) build configuration in the current build directory. If you don't want to use Ninja, omit `-GNinja` above and it will generate a Makefile instead.

Lastly, build the libraries:

```bash
cmake --build cxx cxxabi
```

This will place the relevant build artifacts in `./lib` and `./include`.

## Building `clang`

Go back up to the root `llvm-project` directory, and create one more build directory:

```bash
CLANG_DIR="$PWD/build_clang"
mkdir $CLANG_DIR
cd $CLANG_DIR
```

Then, configure CMake again. This time, we're going to specify the `libc++` we just built in the previous section, using the build directory `$LIBCXX_DIR` we used previously.

```bash
cmake -GNinja ../llvm \
	-DCMAKE_BUILD_TYPE=Release \
	-DLLVM_ENABLE_PROJECTS="clang;lld" \
	-DCMAKE_C_COMPILER=clang \
	-DCMAKE_CXX_COMPILER=clang++ \
	-DLLVM_USE_SANITIZER=MemoryWithOrigins \
	-DLLVM_ENABLE_LIBCXX=ON \
	-DCMAKE_CXX_FLAGS=" -nostdinc++ -isystem $LIBCXX_DIR/include -isystem $LIBCXX_DIR/include/c++/v1" \
	-DCMAKE_EXE_LINKER_FLAGS=" -stdlib=libc++ -Wl,--rpath=$LIBCXX_DIR/lib -L$LIBCXX_DIR/lib"
```

Then, build the binaries:

```bash
cmake --build clang lld
```

This will place the built binaries in `./bin`.

*Note: building all targets in the instrumented tree will attempt to link newly built MSan runtime with MSan runtime from the previous stage, which is not a good idea.*