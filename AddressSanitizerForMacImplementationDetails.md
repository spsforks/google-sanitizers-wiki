# [AddressSanitizer](AddressSanitizer) for Mac: don’t be afraid.

**ATTENTION**: most of the implementation details of ASan on Mac have changed in the past years and aren't reflected in this document. Please refer to the source. In particular, we are no longer using mach\_override (which used to be a maintenance hell and didn't work on iOS) in favor of dyld interposition (see http://llvm.org/viewvc/llvm-project/compiler-rt/trunk/lib/interception/interception.h?view=markup) and re-exec()ution. This allows us to hook the allocation/deallocation routines early and eliminates the need to replace CFAllocator and wrap various CF... functions.
Overall, we've stopped relying on most of the undocumented functions and assumptions.

This document should cover the implementation details of [AddressSanitizer](AddressSanitizer) for Mac OS, because there are notable differences from Linux. We are almost sure that not all the possible issues are covered here (and in the code), so any comments and suggestions are welcome.



## Overview

[AddressSanitizer](AddressSanitizer) is a dynamic detector of addressability bugs like buffer overflows (on the heap, stack and global variables) use-after-free errors for heap allocations and use-after-return errors for stack allocations.
The [AddressSanitizer](AddressSanitizer) algorithm is described in [AddressSanitizerAlgorithm](AddressSanitizerAlgorithm), but we’ll repeat it here for the sake of completeness.

Every 8 bytes of application memory are backed up with a byte of shadow memory which tells how many of those 8 bytes are addressable. Each memory access in a program is instrumented with code that checks the addressability of the memory region being accessed before the actual memory access. If the memory is unaddressable, an error is reported.
[AddressSanitizer](AddressSanitizer) consists of an LLVM instrumentation pass and a runtime library.
The instrumentation pass adds the code that checks the shadow memory before each memory access. It also rearranges the variables on the stack and in the data segment by adding “poisoned” (unaddressable) redzones to them. For each module in the client program the instrumentation pass emits a static constructor that calls `__asan_init()` (the routine that initializes the runtime library if it wasn’t initialized yet) and `__asan_register_globals()` (tells the runtime about the globals in the module and initializes their redzones). Similarly, a static destructor that tears the globals down by calling `__asan_unregister_globals()` is emitted.

The runtime library manages the shadow memory for the application. It replaces the client memory allocation routines so that poisoned redzones are added to each allocated chunk of memory. It also poisons the redzones around global variables (on program startup) and around stack variables (on function entry).
The runtime library intercepts a number of system library functions (http://llvm.org/viewvc/llvm-project/compiler-rt/trunk/lib/asan/asan_interceptors.cc?view=markup). This is needed for two reasons:
to keep track of the program state (e.g. thread creation and destruction, stack context manipulation with `setjmp()/longjmp()`, signal handling)
to improve the tool’s precision by replacing non-instrumented functions that are popular causes of memory errors (`memcpy()`, `memmove()`, `strcmp()` etc.)

We’ve added the `-fsanitize=address` flag to Clang, the LLVM front-end for C language family. For the source files compiled with Clang it produces the instrumented object files. For the object files it invokes ld, which links them together with the runtime library into a single binary. Instrumented and non-instrumented modules may be linked together if the two conditions are met:
the modules keep their binary compatibility with each other (and the system libraries), otherwise something may crash at startup;
code from non-instrumented modules that is executed before `__asan_init()` cannot call code from the instrumented modules, otherwise a segmentation violation will occur while trying to access the shadow memory. In fact it is almost impossible to ensure this at compile time, so the best option is to instrument the whole program.

An important design decision taken in the early days of [AddressSanitizer](AddressSanitizer) is that it should be as transparent to the user as possible. Therefore it’s usually enough to pass `-fsanitize=address` to `clang`/`clang++` in order to compile and link the instrumented program, and the resulting binary should just work as it would normally do, and report memory errors in the case they occur. We try to minimize the use of any additional options needed to run the instrumented program, because it usually complicates the deployment process and thwart the users.

## [AddressSanitizer](AddressSanitizer) and other memory tools

Some comparison of existing memory tools is done at [AddressSanitizerComparisonOfMemoryTools](AddressSanitizerComparisonOfMemoryTools).

### Valgrind Memcheck

[Memcheck](http://valgrind.org/docs/manual/mc-manual.html) is based on [Valgrind](http://valgrind.org/), a binary instrumentation framework, so all the advantages and disadvantages of DBI apply to it. It does not require code recompilation and is able to find the memory errors in the shared libraries and JITted code as well as in the main executable. On the other hand, it is unable to find bugs on global and stack variables. Memcheck is also quite slow, mainly because of the translation/instrumentation overhead and the program threads being serialized.
Memcheck also allows the user to detect problems related to uninitialized values, which requires a more complex shadow state and is not the goal of [AddressSanitizer](AddressSanitizer).

### libgmalloc

Some people compare [AddressSanitizer](AddressSanitizer) to [`libgmalloc`](https://developer.apple.com/library/mac/#documentation/darwin/reference/manpages/man3/libgmalloc.3.html). These two tools are a bit similar, but they may find different bugs. libgmalloc places each heap allocation on its own virtual memory page, which is unmapped when the allocated memory chunk is freed. This allows libgmalloc to easily detect use-after-free bugs without instrumenting the program or the system libraries, but at the same time page-level granularity does not allow to catch the accesses right behind the array bounds.
libgmalloc does not manage stack memory and the globals, thus it can’t detect stack and global buffer overflows.

## Clang and compile-time stuff

### Instrumentation

Mac-specific bits of the ASan instrumentation pass (http://llvm.org/viewvc/llvm-project/llvm/trunk/lib/Transforms/Instrumentation/AddressSanitizer.cpp?view=markup) are related to handling ObjC variables and functions.

**`.cstring` section.** All the globals put into this section are compressed at link time: the linker removes all the spare \0 symbols after the string terminator, thus breaking the redzones. We do not instrument these globals.

**`__OBJC` section and `__DATA,_objc_` segment.** The ObjC runtime assumes that the global objects from these places conform to /usr/include/objc/runtime.h, so we can’t add redzones to them either.

**Constant CFString instances.** Each constant CFString consists of two parts: the string buffer is emitted into the `__TEXT,__cstring,cstring_literals` section, and the constant NSConstantString structure referencing that buffer is placed into `__DATA,__cfstring`.
There’s no point in adding redzones to that NSConstantString structure, and it causes the linker to crash on OS X 10.7 (see http://code.google.com/p/address-sanitizer/issues/detail?id=32), so we do not instrument those, too.

**ObjC load methods.** An ObjC class may have a +load method, which is called at the time the module is loaded into the memory by dyld. This is pretty early, and at this moment `__asan_init()` may have not been called yet. To avoid the pre-initialization crashes, we insert a call to `__asan_init()` at the beginning of each function which name contains `“load]”`.

Generally speaking, it’s possible to compile ObjC code on Linux, so there is nothing specific to Mac OS here. But so far these features were tested on Mac only, as long as they were needed only for it.

### Compiling and linking

In the case when `-faddress-sanitizer` is passed to the linker when building an executable binary, we link it with `libclang_rt.asan_osx.a`, the ASan runtime library (a fat binary containing both 32- and 64-bit versions of our runtime), and the CoreFoundation framework, on which the runtime library depends (see http://llvm.org/viewvc/llvm-project/cfe/trunk/lib/Driver/ToolChains.cpp?view=markup).
If one links dynamic libraries or Mach-O bundles (selected by the `-dynamiclib` and `-bundle` command line flags), the resulting binary will probably lack the symbols provided by the runtime library (at least there’ll be no `__asan_init()`). We mark those symbols as dynamic\_lookup, so that the linking succeeds (see http://llvm.org/viewvc/llvm-project/cfe/trunk/lib/Driver/Tools.cpp?view=markup). This may result in hiding problems with unresolved symbols in other libraries, but those will be detected at runtime or when doing a regular build without ASan.

## Runtime library

### Early initialization

Unlike `ld.so`, `dyld` on Mac OS does not support the `.preinit_array` section (the way we’re calling `__asan_init()` before any of the static constructors is invoked on Android).
For the sake of user experience, we deliberately do not want to use any wrappers and preloaded libraries to jump in early, so the only remaining option is to instrument each static constructors’ section (see above).
Another approach worth to consider is to patch the resulting executable after it has been linked. This may work, but will certainly complicate the tool usage.

### Overriding functions

Because of the two-level namespaces used by many applications, it is generally impossible to reliably intercept the functions by providing replacements for them and obtaining the original function pointers via `dlsym()`. 

In the past we used the mach\_star framework (https://github.com/rentzsch/mach_star/tree/master/mach_override), but it turned out to be hard to maintain and required code patching.

The current interposition mechanism is based on the `“__DATA,__interpose”` section in the runtime library (http://code.google.com/p/address-sanitizer/issues/detail?id=64). A library with `__DATA,__interpose` must be preloaded into the running process in order for function interposition to work fine. To achieve that, at ASan startup we check the contents of DYLD_INSERT_LIBRARIES for the presence of ASan runtime library. If it is not listed, we add the runtime library to DYLD_INSERT_LIBRARIES and re-execute the program.

### Hooking the memory allocations

Replacing the memory allocation routines is trickier on Mac OS than on Linux, because there can be several heap allocators in a program, and each of them has to be intercepted.
Mac OS uses so-called malloc zones, which are described in `/usr/include/malloc/malloc.h`
Each malloc zone has a corresponding malloc\_zone\_t instance describing it.
This is a data structure that contains pointers to interface functions like `malloc_zone_malloc()`, `malloc_zone_calloc()`, `malloc_zone_free()` (similar to `malloc()`/`calloc()`/`free()`, but with an additional `malloc_zone_t` parameter).

```
typedef struct _malloc_zone_t {
    void        *reserved1;
    void        *reserved2;
    size_t      (*size)[...]
    void        *(*malloc)[...]
    void        *(*calloc)[...]
    void        *(*valloc)[...]
    void        (*free)[...]
    void        *(*realloc)[...]
    void        (*destroy)[...]
    const char  *zone_name;
    unsigned    (*batch_malloc)[...]
    void        (*batch_free)[...]
    struct malloc_introspection_t *introspect;
    unsigned    version;
    void *(*memalign)(struct _malloc_zone_t *zone, size_t alignment, size_t size);
    void (*free_definite_size)(struct _malloc_zone_t *zone, void *ptr, size_t size);
    size_t      (*pressure_relief)[...]
} malloc_zone_t;
```

Unlike on Linux, it is possible to ask if the zone owns a particular allocation and what is its size.
This theoretically allows to find the owner of a given allocation by calling `malloc_zone_from_ptr()`. However not all the system allocators implement the `size()` function correctly. Because of this it is impossible to tell whether a particular address has been allocated by some unknown memory zone (this is not a problem on Linux, where there’s typically only one allocator), or if it is a random address. We have to disable reporting double frees on Mac to avoid false positives.

Our code replacing the malloc zones (http://llvm.org/viewvc/llvm-project/compiler-rt/trunk/lib/asan/asan_malloc_mac.cc?view=markup) was mainly taken from Google Perftools (http://code.google.com/p/gperftools/source/browse/trunk/src/libc_override_osx.h). 

We do not do anything with the garbage collected allocators.

### Threading and Grand Central Dispatch

[AddressSanitizer](AddressSanitizer) needs to keep track of the threads that exist during the program execution.
For every thread an `AsanThreadSummary` object is created.

Starting at OS X 10.6 Apple introduced Grand Central Dispatch, a mechanism for running parallel tasks on a thread pool. The threads in that pool are created by bsdthread\_create, the same system call used by `pthread_create()`. But intercepting a system call is harder than intercepting a function, so we choose to wrap a number of libdispatch API functions instead of that. This allows us to hijack the callbacks sent to the worker threads and tell the runtime library that a new thread has appeared.

### File mappings and symbolization

[AddressSanitizer](AddressSanitizer) sometimes needs to know the list of memory mappings in the process address space. There are two cases when it is necessary: to detect which image does a stack frame belong to, and to check whether any of the mapped libraries overlap with the shadow memory range at startup (this is possible if ASLR is on, see below).

Initially we had been using an interface class from gperftools (https://github.com/gperftools/gperftools/) that queried /proc/self/maps on Linux and emulated the same behavior on Mac using the dyld API (_dyld\_get\_image_{name,header,vmaddr\_slide}()). This means we can only get the list of images mapped by the process, not all the existing mappings. However this hadn’t been an issue, because the mappings that do not correspond to binary images typically do not contain program code (ASan can’t symbolize JITted code), and there are no such mappings at program startup.

We've rewritten the gperftools code heavily now (http://llvm.org/viewvc/llvm-project/compiler-rt/trunk/lib/sanitizer_common/sanitizer_procmaps.h?view=markup), but it is still using the same API.

### Various problems

#### `memcpy()` and `memmove()`

[AddressSanitizer](AddressSanitizer) wraps `memcpy()` and `memmove()` in order to check that the memory regions being accessed are addressable, and that the parameters of `memcpy()` do not overlap.
Such kind of bugs doesn't apply to OS X, where `memcpy()` is an alias to `memmove()`. Therefore we only need to wrap `memmove()` on OS X.