# Introduction

[MemorySanitizer](MemorySanitizer) code itself can not be tested with [MemorySanitizer](MemorySanitizer) for obvious reasons, but the rest of Clang/LLVM can. Instructions on this page show how to do it.

# Details

MSan requires that all code in the process is instrumented (see [handling-external-code](http://clang.llvm.org/docs/MemorySanitizer.html#handling-external-code)). Luckily, the only external dependency of Clang is the C++ standard library (and, of course, libc, but MSan _almost_ takes care of it).

Checkout the source:
```
(svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm && cd llvm &&
(R=$(svn info | grep Revision: | awk '{print $2}') &&
(cd tools && svn co -r $R http://llvm.org/svn/llvm-project/cfe/trunk clang)  &&
(cd projects && svn co -r $R http://llvm.org/svn/llvm-project/compiler-rt/trunk compiler-rt)  &&
(cd projects && svn co -r $R http://llvm.org/svn/llvm-project/libcxx/trunk libcxx)  &&
(cd projects && svn co -r $R http://llvm.org/svn/llvm-project/libcxxabi/trunk libcxxabi)))
```

## Build Clang

```
(mkdir -p build && cd build && cmake -GNinja -DCMAKE_BUILD_TYPE=Release ../llvm && ninja)
```

## Build libc++ and libc++abi with MemorySanitizer

```
(mkdir -p build-libcxx-msan && cd build-libcxx-msan &&
  (cmake -GNinja -DCMAKE_BUILD_TYPE=Release \
    -DLLVM_USE_SANITIZER=MemoryWithOrigins \
    -DCMAKE_C_COMPILER=$PWD/../build/bin/clang \
    -DCMAKE_CXX_COMPILER=$PWD/../build/bin/clang++ \
    ../llvm) &&
  ninja cxx cxxabi)
```

## Build clang with MemorySanitizer, using the new libc++

```
(mkdir -p build-clang-msan && cd build-clang-msan &&
  (CLANG_BUILD=$PWD/../build;
    LIBCXX_BUILD=$PWD/../build-libcxx-msan;
    MSAN_LINK_FLAGS="-lc++abi -Wl,--rpath=$LIBCXX_BUILD/lib -L$LIBCXX_BUILD/lib";
    MSAN_FLAGS="$MSAN_LINK_FLAGS -nostdinc++ \
      -isystem $LIBCXX_BUILD/include \
      -isystem $LIBCXX_BUILD/include/c++/v1  \
      -fsanitize=memory \
      -fsanitize-memory-track-origins \
      -w";
    cmake -GNinja \
      -DCMAKE_C_COMPILER=$CLANG_BUILD/bin/clang \
      -DCMAKE_CXX_COMPILER=$CLANG_BUILD/bin/clang++ \
      -DCMAKE_C_FLAGS="$MSAN_FLAGS" \
      -DCMAKE_CXX_FLAGS="$MSAN_FLAGS" \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_USE_SANITIZER=MemoryWithOrigins \
      -DLLVM_ENABLE_LIBCXX=ON \
      -DCMAKE_EXE_LINKER_FLAGS=$MSAN_LINK_FLAGS \
      ../llvm) &&
  ninja clang check-clang check-llvm)
```

Note that building all targets in the instrumented tree will attempt to link newly built MSan runtime with MSan runtime from the previous stage, which is not a good idea.