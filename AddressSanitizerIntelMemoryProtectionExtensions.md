

# Introduction
On July 2013 Intel released documentation on the upcoming instruction set extensions,
including the Memory Protection Extensions (MPX). Here we will discuss the applicability of MPX for memory error detection.
Links: [MPX-enabled GCC wiki](http://gcc.gnu.org/wiki/Intel%20MPX%20support%20in%20the%20GCC%20compiler);
[Using the MPX-enabled GCC and SDE (emulator)](http://software.intel.com/en-us/articles/using-intel-mpx-with-the-intel-software-development-emulator);
[Fresh documentation on Intel ISA which includes MPX](http://download-software.intel.com/sites/default/files/319433-015.pdf);
[Intel Pointer Checker](http://software.intel.com/en-us/articles/pointer-checker-feature-in-intel-parallel-studio-xe-2013-how-is-it-different-from-static).
Some external feedback: [1](https://groups.google.com/d/msg/comp.arch/iKAACmTrTQs/bzqG5Dp-FPEJ), [2](http://www.cs.rutgers.edu/news/articles/2013/07/24/intel-memory-protection-extensions), [3](http://lists.cs.uiuc.edu/pipermail/cfe-dev/2014-September/039088.html).

**NEW** As of January 2016, Intel MPX is available in hardware and one can actually try how it works!

# Setup
* To build with MPX all you need is a fresh GCC, e.g. 5.3. Now you can build your code with MPX and execute the binary on any x86_64 machine. The MPX instructions will be treated as NOPs, but you will be able to measure e.g. code size increase and MPX-NOP overhead. Follow the [GCC instructions](https://gcc.gnu.org/wiki/Intel%20MPX%20support%20in%20the%20GCC%20compiler********) to build your code. 
* To run with the actual checks acquire a new machine with an MPX-enabled CPU,
e.g. [i7-6700](http://ark.intel.com/products/88196/Intel-Core-i7-6700-Processor-8M-Cache-up-to-4_00-GHz).
Also Make sure your Linux kernel is built with `CONFIG_X86_INTEL_MPX=y`

# Run
```
% cat heap-buffer-overflow.c
#include <stdlib.h>
int main(int argc, char **argv) {
  char *x = (char*)malloc(argc * 10);
  x[argc + 10] = 0;
  int res = x[10];
  free(x);
  return res;
}
% $GCC_ROOT/bin/gcc -fcheck-pointer-bounds -mmpx heap-buffer-overflow.c -static
```

Now, if you run this on non-MPX-enabled machine, the application will exit silently.
However, if you run it on a proper MPX machine, you will get this:
```
% ./a.out
Saw a #BR! status 1 at 0x401d0a
Saw a #BR! status 1 at 0x401d27
% objdump -d a.out | grep "401d0a\|401d27"
  401d0a:       f2 0f 1a 08             bndcu  (%rax),%bnd1
  401d27:       f2 0f 1a 18             bndcu  (%rax),%bnd3
```

As you can see, `fcheck-pointer-bounds` found the buffer overflows using the `bndcu` instructions.
The error message could have been more verbose, but that's not the hardware task.

Let's now try something more interesting, but sill simple enough: bzip2.
```
wget http://www.bzip.org/1.0.6/bzip2-1.0.6.tar.gz
tar xf bzip2-1.0.6.tar.gz
(
cd bzip2-1.0.6
make clean
make  all LDFLAGS=-static  -j CC="$GCC_ROOT/bin/gcc  \
 -fcheck-pointer-bounds -mmpx -Wl,-rpath=$GCC_ROOT/lib64"
mv bzip2 ../bzip2-mpx
make clean
make  all LDFLAGS=-static  -j CC="$GCC_ROOT/bin/gcc"
mv bzip2 ../bzip2-plain
)
```

Now, find some large file (50Mb+), copy it to `inp` and run this:
```
for f in plain mpx; do time ./bzip2-$f -c inp > /dev/null ; done
```

One a non-MPX machine, the MPX binary will execute NOPs.
On my Xeon E5-2680 I observe 80% (!!!!) slowdown from MPX-NOPs.

Profile w/o mpx:
```
 44.29%  bzip2-plain  bzip2-plain        [.] mainSort
 43.07%  bzip2-plain  bzip2-plain        [.] BZ2_compressBlock
  5.32%  bzip2-plain  bzip2-plain        [.] handle_compress.isra.2
  4.82%  bzip2-plain  bzip2-plain        [.] mainGtU.part.0       
```
Profile with MPX:
```
 35.27%  bzip2-mpx  bzip2-mpx          [.] generateMTFValues
 21.24%  bzip2-mpx  bzip2-mpx          [.] mainSort         
 11.89%  bzip2-mpx  bzip2-mpx          [.] sendMTFValues    
 11.27%  bzip2-mpx  bzip2-mpx          [.] mainSimpleSort   
  9.39%  bzip2-mpx  bzip2-mpx          [.] copy_input_until_stop
  5.32%  bzip2-mpx  bzip2-mpx          [.] mainGtU.chkp.part.0  
  2.98%  bzip2-mpx  bzip2-mpx          [.] copy_output_until_stop
```

Comparing the profiles, it looks like MPX disables inlining in the compiler, or at least forces the inliner to make different decisions.
This may partially explain the performance difference.

Also, `perf` attributes lots of CPU cycles to the `bnd` instructions (not sure if we can trust perf here):
```
  8.44 │       bndcl  (%rax),%bnd1
  6.42 │       bndmov 0x10(%rsp),%bnd2  ...
 12.49 │       bndcu  (%rax),%bnd2
```

Maybe the compiler implementation in GCC is too naive.
For example, I observe same bound information being loaded twice into different bnd registers: 
```
  401cd4:       66 0f 1a 0c 24          bndmov (%rsp),%bnd1   <<<<<<<<<<
  401cd9:       f2 0f 1a 0a             bndcu  (%rdx),%bnd1
  401cdd:       c6 02 00                movb   $0x0,(%rdx)
  401ce0:       48 8d 50 0a             lea    0xa(%rax),%rdx
  401ce4:       66 0f 1a 04 24          bndmov (%rsp),%bnd0   <<<<<<<<<<
```

Now, we may run the same binaries on a proper MPX-enabled machine. (Stay tuned)


# Performance
MPX has several different instructions that have very different performance properties:
  * BNDCU/BNDCL/BNDMK -- pure arithmetic, supposedly very fast.
  * BNDMOV -- move the BNDx registers from/to memory, mostly used to spill/fill registers. When accessing memory on stack should hit the L1 cache and thus be fast.
  * BNDLDX/BNDSTX -- access the Bound Table, a 2-layer cache-like data structure. Supposedly very slow (accesses two different cache lines).

Every memory access that needs to be checked will be instrumented with BNDCU/BNDCL (compiler optimizations may apply).
Since BNDCU/BNDCL are expected to be very fast, the slowdown will be defined by the ratio of the number of executed BNDCU/BNDCL vs BNDLDX/BNDSTX/BNDMOV.

The SDE allows to collect the number of executed instructions using the [`-mix` switch](http://software.intel.com/en-us/articles/intel-software-development-emulator).
We've collected stats on SPEC 2006, see the [spreadsheet](https://docs.google.com/spreadsheet/ccc?key=0Asb2KOp6b-AkdFAtWW5IYl9rNXk5OGM0YmROMmtnMXc#gid=0).

For some benchmarks (e.g. 444.namd or 462.libquantum)  dozenes of BNDCU/BNDCL are executed per single BNDLDX/BNDSTX.
For similar applications we can expect that MPX-based bug detection tools will be lightning fast.

For many other benchmarks (e.g. 400.perlbench, 429.mcf, 483.xalancbmk, 471.omnetpp)
the number of expensive BNDMOV and very expensive BNDLDX/BNDSTX instruction is comparable to the number of checks.
For these applications we expect MPX to be slower than alternative software-only solutions, such as [AddressSanitizer](AddressSanitizer).

Note that the data is preliminary because we've built the benchmarks without the MPX-enabled glibc.

# False positives
## False positive with atomic pointers
http://software.intel.com/en-us/forums/topic/413959
```
% cat cxx11_ptr_check.cc
// Example of a false positive with Pointer Checker or MPX.
// The false report happens because the pointer update
// and the metadata update together do not happen atomically.
#include <atomic>
#include <thread>
#include <iostream>
#include <assert.h>
std::atomic<int *> p;
int A, B;
void Thread1() { for (int i = 0; i < 100000; i++) p = &A; }
void Thread2() { for (int i = 0; i < 100000; i++) p = &B; }
void Thread3() { for (int i = 0; i < 100000; i++) assert(*p == 0); }

int main() {
  std::cout << "A=" << &A << " B=" << &B << std::endl;
  p = &A;
  std::thread t1(Thread1);
  std::thread t2(Thread2);
  std::thread t3(Thread3);
  t1.join();
  t2.join();
  t3.join();
}
% $MPX_GCC/bin/g++ -std=c++0x -fcheck-pointers -mmpx -L$MPX_RUNTIME_LIB -B$MPX_BINUTILS/bin \
 -lmpx-runtime64 -Wl,-rpath,$MPX_RUNTIME_LIB cxx11_ptr_check.cc 
% $SDE_KIT/sde   -mpx-mode -- ./a.out
A=0x606d28 B=0x606d2c
Bound violation detected,status 0x1 at 0x4012b8
```

## False positive with un-instrumented code
This limitation is acknowledged by Intel developers (http://gcc.gnu.org/wiki/Intel%20MPX%20support%20in%20the%20GCC%20compiler#Mixing_instrumented_and_legacy_code ):

> Note that in rare cases Bounds Table mechanism may miss bounds changes.
> We may model a case when legacy code rewrites a pointer in a memory with pointer of the same value but with different bounds.
> In such case false bound violation may occur. User is responsible for avoiding such cases.
> To get higher level of protection try to use instrumentation for modules generating external data.

```
==> mpxfp1.c <==
#include <stdio.h>
int *p;
extern int Foo(int idx);
extern int Bar(int idx);
void Set(int *x) { p = x; }
int Get(int idx) { return p[idx]; }
int main() {
  return Foo(5) + Bar(15);
}

==> mpxfp2.c <==
#include <stdio.h>
void Set(int *x);
int Get(int idx);
int Foo(int idx) {
  int b[22];
  int a[10];  // Address of 'a' must match 'a' from Bar()
  printf("Foo: a=%p b=%p\n", a, b);
  Set(a);
  return Get(idx);
}

==> mpxfp3.c <==
#include <stdio.h>
extern int *p;
int Get(int idx);
int Bar(int idx) {
  int a[20];  // Address of 'a' must match 'a' from Bar()
  int b[20];
  printf("Bar: a=%p b=%p\n", a, b);
  p = a;
  return Get(idx);
}
% $MPX_GCC/bin/g++ -O  -fcheck-pointers -mmpx -L$MPX_RUNTIME_LIB -B$MPX_BINUTILS/bin -lmpx-runtime64 -c mpxfp[12].c
% $MPX_GCC/bin/g++ -O -c mpxfp3.c  # No instrumentation
% $MPX_GCC/bin/g++ -fcheck-pointers -mmpx -L$MPX_RUNTIME_LIB -B$MPX_BINUTILS/bin -lmpx-runtime64 \
 -Wl,-rpath,$MPX_RUNTIME_LIB mpxfp[123].o
% $SDE_KIT/sde   -mpx-mode -- ./a.out
Foo: a=0x7ffff074d8f0 b=0x7ffff074d920
Bar: a=0x7ffff074d8f0 b=0x7ffff074d940
Bound violation detected,status 0x1 at 0x4006ce
```

Another example is when a heap memory address is reused: http://software.intel.com/en-us/forums/topic/413960

## False positives caused by compiler optimizations
We expect that the compiler optimizations will be causing a major headache to implementers of MPX-based checkers.

One example (mpx-gcc r201896 (on Google Code)):
```
% cat two_arrays.cc 
struct Bar {
  virtual ~Bar() { }
};
struct Foo {
  Bar x[3], y[3];
};
int main() {
  Foo f;
}
% $MPX_GCC/bin/g++ -fcheck-pointers -mmpx -O2  -L$MPX_RUNTIME_LIB -B$MPX_BINUTILS/bin -lmpx-runtime64 \
  -Wl,-rpath,$MPX_RUNTIME_LIB two_arrays.cc && $SDE_KIT/sde -mpx-mode -mpx_stats -- ./a.out
   Bound violation detected,status 0x1 at 0x4007ed
```
Here the compiler creates 2 loops to destruct `x` and `y`.
The second loop (destruction of `x`) uses the pointer that starts from the beginning of `y`,
i.e. which is out of bounds right away.

This class of false positives is avoidable with careful analysis of compiler optimizations.

## Variable size fields
Variable size fields typically cause false positives with MPX, which is quite expected.
The compiler has a special attribute to mark variable size fields as such: `__attribute__((bnd_variable_size))`.
We had to apply this attribute
in 8 places in [Chromium](http://dev.chromium.org) sources (7 in  the ICU code) to get Chromium's `base_unittests` running.
Example (third\_party/icu/source/common/ucmndata.h)
```
  typedef struct {
       uint32_t count;
       UDataOffsetTOCEntry entry[2] __attribute__((bnd_variable_size));    /* Actual size of array is from count. */
  } UDataOffsetTOC;
```


# Comparison with AddressSanitizer
MPX has just recently become available in hardware.

Most of this section is speculation based on our earlier evaluation of
[Intel Pointer Checker](http://software.intel.com/en-us/articles/pointer-checker-feature-in-intel-parallel-studio-xe-2013-how-is-it-different-from-static)
(software-only implementation of MPX-like checker) and the MPX-enabled gcc (which can be run under emulator).

## MPX strengths
MPX-based tool can find in-struct buffer overflows:
```
  struct X {int a[10], b[20]; }; ...
  X x; ...
  x.a[15];  // Overflows to x.b[5]
```

MPX-based tool can find buffer overflows of any size since it does not rely on redzones:
```
int a[10]; ...
a[1000000] = 0;  // Will be detected by MPX
```

MPX is expected to be very fast if the ratio of BNDCU/BNDLDX is large (i.e. for programs with long loops iterating over arrays).

## MPX weaknesses
  * MPX can not find use-after-free bugs
  * MPX has false positives with atomic pointers
  * MPX has false positives if some of the code is not instrumented
  * MPX is (as we expect) very slow for code working with lots of pointers (trees, lists, graphs, etc). This is partially confirmed by very small ratio of BNDCU/BNDLDX on 483.xalancbmk (above).
  * MPX has up to 4x overhead in RAM if the program has lots of pointers (trees, lists, graphs, etc).
  * MPX may be hard to deploy on legacy code where pointers to members are used to access other members (e.g. at least 7 SPEC benchmarks have errors).

# Biased conclusion
A **very biased** conclusion: Intel MPX might be useful for in-struct buffer overflow detection, and for general buffer overflow detection in programs with lots of arrays and few pointers.
However [AddressSanitizer](AddressSanitizer) (and, if implemented, [AddressSanitizerInHardware](AddressSanitizerInHardware)) is more useful: faster, finds more bugs, easier to deploy.

# Random Thoughts
BNDLDX/BNDSTX are (I guess) very slow since they access two cache lines.
Instead of BNDLDX/BNDSTX a tool may use a simple directly mapped shadow and use BNDMOV to read/write bounds.
If we instrument the entire program (with all libraries) we don't need to check the pointer value (as in BNDLDX)
and can have 2x shadow instead of 4x shadow.
TODO: rephrase this better.
