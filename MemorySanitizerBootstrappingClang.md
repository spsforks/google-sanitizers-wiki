# Introduction

[MemorySanitizer](MemorySanitizer) code itself can not be tested with [MemorySanitizer](MemorySanitizer) for obvious reasons, but the rest of Clang/LLVM can. Instructions on this page show how to do it.

# Details

MSan requires that all code in the process is instrumented (see [handling-external-code](http://clang.llvm.org/docs/MemorySanitizer.html#handling-external-code)). Luckily, the only external dependency of Clang is the C++ standard library (and, of course, libc, but MSan _almost_ takes care of it).

Checkout the source:
```
svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm
cd llvm
R=$(svn info | grep Revision: | awk '{print $2}')
(cd tools && svn co -r $R http://llvm.org/svn/llvm-project/cfe/trunk clang)
(cd projects && svn co -r $R http://llvm.org/svn/llvm-project/compiler-rt/trunk compiler-rt)
(cd projects && svn co -r $R http://llvm.org/svn/llvm-project/libcxx/trunk libcxx)
(cd projects && svn co -r $R http://llvm.org/svn/llvm-project/libcxxabi/trunk libcxxabi)
```

## Build Clang

```
mkdir /code/build && cd /code/build && cmake -GNinja -DCMAKE_BUILD_TYPE=Release /code/llvm && ninja
```

## Build libc++ and libc++abi with MemorySanitizer

```
mkdir /code/build-libcxx-msan && cd /code/build-libcxx-msan
CC=/code/build/bin/clang \
CXX=/code/build-llvm/bin/clang++ \
cmake -GNinja \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_USE_SANITIZER=MemoryWithOrigins
      /code/llvm
ninja cxx cxxabi
```

## Build clang with MemorySanitizer, using the new libc++

```
MSAN_FLAGS=" \
  -nostdinc++ \
  -isystem /code/build-libcxx-msan/include \
  -isystem /code/build-libcxx-msan/include/c++/v1  \
  -lc++abi \
  -Wl,--rpath=/code/build-libcxx-msan/lib \
  -L/code/build-libcxx-msan/lib \
  -fsanitize=memory \
  -fsanitize-memory-track-origins \
  -w"

CC=/code/build/bin/clang \
CXX=/code/build/bin/clang++ \
CFLAGS=$MSAN_FLAGS \
CXXFLAGS=$MSAN_FLAGS \
cmake -GNinja \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_USE_SANITIZER=MemoryWithOrigins \
      -DLLVM_ENABLE_LIBCXX=ON \
      -DCMAKE_EXE_LINKER_FLAGS="-lc++abi -Wl,--rpath=/code/build-libcxx-msan/lib -L/code/build-libcxx-msan/lib" \
      /code/llvm

ninja clang
ninja check-clang check-llvm
```

Note that building all targets in the instrumented tree will attempt to link newly built MSan runtime with MSan runtime from the previous stage, which is not a good idea.