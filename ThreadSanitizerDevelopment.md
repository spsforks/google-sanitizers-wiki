# LLVM Development

To download the latest sources:
```shell
# cd somewhere
svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm
cd llvm
R=$(svn info | grep Revision: | awk '{print $2}')
(cd tools && svn co -r $R http://llvm.org/svn/llvm-project/cfe/trunk clang)
(cd projects && svn co -r $R http://llvm.org/svn/llvm-project/compiler-rt/trunk compiler-rt)
```

To build (requires cmake>=2.8.8):
```shell
# in llvm dir
mkdir build && cd build
CC=gcc CXX=g++ cmake -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_ASSERTIONS=ON /path/to/llvm/checkout
make -j10
```

To run tests:
```
cd build
make check-all -j10 # build and run all tests
make check-tsan -j10 # build and run ThreadSanitizer tests
```

Additional build types:

* Pass `-DCOMPILER_RT_DEBUG=ON` to CMake to build debug version of TSan runtime library.
* Additionally pass `-DCOMPILER_RT_TSAN_DEBUG_OUTPUT=ON` to enable stats collection
  and extended debug output (including memory accesses and function entry/exit events, be
  careful it can be enormous).

To run an arbitrary program with `ThreadSanitizer`:
```
your/fresh/clang test.c -fsanitize=thread -g -O1
```

# GCC Development

Note that the runtime library is developed in LLVM repository, it is only imported into GCC.

To download the latest sources:
```
svn co svn://gcc.gnu.org/svn/gcc/trunk gcc
```

To build:
```
cd gcc && mkdir build && cd build
../configure --prefix=/path/to/gcc/install
make -j10 && make install -j10
```

If you add '--enable-languages=c,c++ --disable-bootstrap --enable-checking=no' to the configure, the process will be way faster.  Sometimes it's also necessary to add '--with-gnu-as --with-gnu-ld'.  If it can't find some libraries run:
```
sudo apt-get install flex bison libc6-dev libc6-dev-i386 libgmp3-dev libmpfr-dev libmpc-dev
```

To run tests:
```
cd build
make check-gcc
```

To run an arbitrary program with `ThreadSanitizer`:
```
your/fresh/gcc test.c -fsanitize=thread -g -O1 -fPIE -pie
```