# Summary

|                                         | [AddressSanitizer](AddressSanitizer)           | [Valgrind/Memcheck](http://valgrind.org) | [Dr. Memory](http://dynamorio.org/drmemory.html) | [Mudflap](http://gcc.gnu.org/wiki/Mudflap_Pointer_Debugging) | Guard Page  | [gperftools](https://github.com/gperftools/gperftools) |
|:----------------------------------------|:-----------------------------------------------|:-----------------------------------------|:-------------------------------------------------|:-------------------------------------------------------------|:------------|:----------------------------------------------------|
| technology                              | CTI                                            | DBI                                      | DBI                                              | CTI                                                          | Library     | Library                                             |
| ARCH                                    | x86, ARM, PPC                              | x86, ARM, PPC, MIPS, S390X, TILEGX                             | x86                                              | all(?)                                                       | all(?)      | all(?)                                              |
| OS                                      | Linux, OS X, Windows, FreeBSD, Android, iOS Simulator          | Linux, OS X, Solaris, Android                              | Windows, Linux                                   | Linux, Mac(?)                                                | All (1)     | Linux, Windows                                      |
| Slowdown                                | **2x**                                         | 20x                                      | 10x                                              | 2x-40x                                                       | ?           | ?                                                   |
| Detects:                                |                                                |                                          |                                                  |                                                              |             |                                                     |
| [Heap OOB](AddressSanitizerExampleHeapOutOfBounds)       | **yes**                                        | yes                                      | yes                                              | yes                                                          | some        | some                                                |
| [Stack OOB](AddressSanitizerExampleStackOutOfBounds)     | **yes**                                        | no                                       | no                                               | some                                                         | no          | no                                                  |
| [Global OOB](AddressSanitizerExampleGlobalOutOfBounds)   | **yes**                                        | no                                       | no                                               | ?                                                            | no          | no                                                  |
| [UAF](AddressSanitizerExampleUseAfterFree)               | **yes**                                        | yes                                      | yes                                              | yes                                                          | yes         | yes                                                 |
| [UAR](AddressSanitizerExampleUseAfterReturn)             | **yes** (see [AddressSanitizerUseAfterReturn](AddressSanitizerUseAfterReturn))     | no                                       | no                                               | no                                                           | no          | no                                                  |
| UMR                                     | no (see MemorySanitizer)                        | yes                                      | yes                                              | ?                                                            | no          | no                                                  |
| Leaks                                   | **yes** (see LeakSanitizer)                    | yes                                      | yes                                              | ?                                                            | no          | yes                                                 |


**DBI**: dynamic binary instrumentation  
**CTI**: compile-time instrumentation  
**UMR**: uninitialized memory reads  
**UAF**: use-after-free (aka dangling pointer)  
**UAR**: use-after-return  
**OOB**: out-of-bounds  
**x86**: includes 32- and 64-bit.  
**mudflap** was removed in GCC 4.9, as it has been superseded by AddressSanitizer.  
**Guard Page**: a family of memory error detectors ([Electric fence](https://en.wikipedia.org/wiki/Electric_Fence) or [DUMA](http://duma.sourceforge.net/) on Linux, Page Heap on Windows, libgmalloc on OS X)  
**gperftools**: various performance tools/error detectors bundled with TCMalloc. [Heap checker](http://htmlpreview.github.io/?https://github.com/gperftools/gperftools/blob/master/doc/heap_checker.html) (leak detector) is only available on Linux. [Debug allocator](https://github.com/gperftools/gperftools/blob/master/src/debugallocation.cc) provides both guard pages and canary values for more precise detection of OOB writes, so it's better than guard page-only detectors.


# Performance numbers on SPEC cpu2006
  * [AddressSanitizer](AddressSanitizer): see [AddressSanitizerPerformanceNumbers](AddressSanitizerPerformanceNumbers)
  * Valgrind/Memcheck and Dr. Memory: see [Dr. Memory paper](http://static.googleusercontent.com/external_content/untrusted_dlcp/research.google.com/en/us/pubs/archive/37274.pdf)
  * Mudflap: slowdown varies from 2x to 40x
  * Guard Page: 2/3 cpu2006 benchmarks fail (tested using [DUMA](http://duma.sourceforge.net/) on a Linux box with 24G RAM)