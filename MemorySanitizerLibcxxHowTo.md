## Introduction

If you want [MemorySanitizer](MemorySanitizer) to work properly and not produce any false positives, you must ensure that all the code in your program and in libraries it uses is instrumented (i.e. built with `-fsanitize=memory`). In particular, you would need to link against MSan-instrumented C++ standard library. We recommend to use [libc++](http://libcxx.llvm.org/)
for that purpose.

In this manual, we will try to build and run simple program that uses [googletest](https://code.google.com/p/googletest/) framework.

## Test program
Here's our simple test program:
```
$ cat test.cc 
#include "gtest/gtest.h"
#include <string>

TEST(FooTest, Foo) {
  std::string foo("foo");
  EXPECT_STREQ("foo", foo.c_str());
}

int main(int argc, char **argv) {
  ::testing::InitGoogleTest(&argc, argv);
  return RUN_ALL_TESTS();
}
```

[MemorySanitizer](MemorySanitizer) will not work out-of-the-box, and will instead report
false positives coming from uninstrumented code:

```
$ clang++ -fsanitize=memory -I googletest/include/ test.cc gtest-build/libgtest.a -lpthread
$ ./a.out
Uninitialized bytes in __interceptor_strlen at offset 7 inside [0x60400000eff8, 8)
==36428== WARNING: MemorySanitizer: use-of-uninitialized-value
    #0 0x7f50fc26c891 in testing::internal::String::String(char const*) (a.out+0xa3891)
    #1 0x7f50fc264346 in testing::internal::UnitTestImpl::GetTestCase(char const*, char const*, void (*)(), void (*)()) (a.out+0x9b346)
...
```

We need to re-build both C++ standard library and googletest with [MemorySanitizer](MemorySanitizer).

## Instrumented libc++

You need to build both [libc++](http://libcxx.llvm.org/) and
[libc++abi](http://libcxxabi.llvm.org/) with MSan:

First, clone the LLVM source from github and enter a new build directory:

```bash
# clone LLVM
git clone --depth=1 https://github.com/llvm/llvm-project
cd llvm-project
mkdir build; cd build
# configure cmake
cmake -GNinja ../llvm \
	-DCMAKE_BUILD_TYPE=Release \
	-DLLVM_ENABLE_PROJECTS="libcxx;libcxxabi" \
	-DCMAKE_C_COMPILER=clang \
	-DCMAKE_CXX_COMPILER=clang++ \
	-DLLVM_USE_SANITIZER=MemoryWithOrigins
# build the libraries
cmake --build cxx cxxabi
```

This will place the relevant build artifacts in `./lib` and `./include`.

## Instrumented gtest

Now you need to use MSan-ified libc++ to build googletest:

```
$ svn co -r613 http://googletest.googlecode.com/svn/trunk googletest
$ mkdir gtest-msan && cd gtest-msan
$ MSAN_CFLAGS="-fsanitize=memory -stdlib=libc++ -L/path/to/libcxx_msan/lib -lc++abi -I/path/to/libcxx_msan/include -I/path/to/libcxx_msan/include/c++/v1"
$ cmake ../googletest -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_C_FLAGS="$MSAN_CFLAGS" -DCMAKE_CXX_FLAGS="$MSAN_CFLAGS"
$ make -j12
```

## Linking your executable

Now you can build your test and link it against MSan-ified libc++ and googletest:

```
$ clang++ ${MSAN_CFLAGS} test.cc -Igoogletest/include/ gtest-msan/libgtest.a -lpthread -Wl,-rpath,/path/to/libcxx_msan/lib
$ ./a.out
[==========] Running 1 test from 1 test case.
[----------] Global test environment set-up.
[----------] 1 test from FooTest
[ RUN      ] FooTest.Foo
[       OK ] FooTest.Foo (0 ms)
[----------] 1 test from FooTest (0 ms total)

[----------] Global test environment tear-down
[==========] 1 test from 1 test case ran. (1 ms total)
[  PASSED  ] 1 test.
```

## Bugs reporting

You can verify that MSan works as expected by introducing a bug:
```
TEST(FooTest, Foo) {
  int uninitialized;
  EXPECT_GT(uninitialized, 5);
}
```

and verifying that it's properly reported:

```
$ clang++ ${MSAN_CFLAGS} test.cc -Igoogletest/include/ gtest-msan/libgtest.a -lpthread -Wl,-rpath,/path/to/libcxx_msan/lib -g
# Make sure llvm-symbolizer is in your $PATH
$ ./a.out
[==========] Running 1 test from 1 test case.
[----------] Global test environment set-up.
[----------] 1 test from FooTest
[ RUN      ] FooTest.Foo
==39032== WARNING: MemorySanitizer: use-of-uninitialized-value
    #0 0x48d73c in testing::AssertionResult testing::internal::CmpHelperGT<int, int>(char const*, char const*, int const&, int const&) googletest/include/gtest/gtest.h:1463:1
    #1 0x48ce7a in FooTest_Foo_Test::TestBody() test.cc:6:3
...
```

## Also see
* A [Dockerfile](https://github.com/awslabs/amazon-kinesis-video-streams-webrtc-sdk-c/blob/master/.github/msan-tester.Dockerfile) with libc++ and several other library built with MemorySanitizer instrumentation.
* Chromium project maintains [docker files](https://www.chromium.org/developers/testing/memorysanitizer#TOC-Running-on-other-distros-using-Docker) with a large list of system libraries built with MemorySanitizer.
