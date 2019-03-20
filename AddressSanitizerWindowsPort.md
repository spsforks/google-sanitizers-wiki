LLVM AddressSanitizer has been ported to Windows, but there are a number of limitations and differences in deployment.

## Basic usage

First, typically on Posix systems, users enable ASan by passing `-fsanitize=address` to `CFLAGS` and `LDFLAGS`. The compiler is usually called to perform the link step, so it will see `-fsanitize=address` in the link phase and add the ASan runtime library.

In a typical build system set up to work with MSVC, the system will instead invoke the linker directly. This means you have to manually pass the path to the asan runtime library to the linker. These are the flags you need to add:

* `/LIBPATH:path/to/install/lib/clang/$clang_version/lib/windows` - This sets up the library search path to find clang runtime libraries, such as asan. $clang_version is "9.0.0", "8.0.1", or whatever your version of clang is.
* For a statically linked executable: `clang_rt.asan-$arch.lib` - $arch is typically `x86_64` or `i386`.
* For a DLL: `clang_rt.asan_dll_thunk-$arch.lib` - $arch is the same.

You can pass `-fsanitize=address` to the compiler (clang or clang-cl, whichever command syntax you prefer) just as you would on Posix, and that will enable the compiler instrumentation in the usual way.

## Debug CRT incompatibility

ASan does not support linking with the debug CRT versions because they make interposing new, delete, malloc, & co more difficult. Typically, the debug version is enabled with `/MDd` or `/MTd`. To test with ASan, use `/MT` or `/MD` instead.

## LSan

LeakSanitizer is not supported on Windows yet. LeakSanitizer requires being able to stop the process at exit or some other point to scan for live pointers. This is called "StopTheWorld", and the posix implementation uses ptrace, which is not available on Windows.

## Use-after-return

As of this writing, I don't believe UAR detection is supported on Windows, although there is no major technical reason for it. If you are adventurous, try it and file any bugs you run into. :)

## Debugging

If you wish to use an interactive debugger on an x64 process that uses ASan, you will immediately notice that the process raises many access violations during process startup. These are unfortunately necessary and expected. If you are using windbg or cdb, you can ignore access violation exceptions to continue past them (`sxi av`). You may need to set some other type of breakpoint or reenable AV exceptions (`sxe av`) after initialization is finished in order to stop the debugger at the right place. Alternatively, you can disable all first chance exception handling, or use __try / __except with __debugbreak to try to stop the debugger when the bug happens. This makes debugging ASan crashes quite challenging, so read on for more background about the current situation.

One of the first things ASan does on startup is to reserve large sections (20TB at this time) of virtual memory for shadow memory and other reasons. Most Unix kernels will happily satisfy requests to map large quantities of zero pages, and will allocate physical memory (and swap if necessary) on demand. If none is available, the process may be OOM killed, or something else may fail, but the initial reservation succeeds. On Windows, the kernel proactively allocates swap if you try to do the analogous thing (commit the memory). Therefore, ASan on Windows this strategy:

* First, the address space is marked reserved.
* When reserved addresses are first accessed, the OS will raise an access violation exception.
* ASan has a vectored exception handler, which catches the exception.
* If the AV is for memory that ASan reserved, ASan requests the block of memory in question be committed (allocated).
* The exception is marked "handled" and execution resumes.