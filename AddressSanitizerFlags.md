# Compiler flags
| flag| default | description |
|:----|:--------|:------------|
| -fsanitize=address |         | Enable [AddressSanitizer](AddressSanitizer) |
| -fsanitize-address-use-after-scope |         | Enable [stack user after scope](AddressSanitizerExampleUseAfterScope) check|
| -fno-omit-frame-pointer |         | Leave frame pointers. Allows the fast unwinder to function properly. |
| -fsanitize-blacklist=path |         | Pass a [blacklist file](#Turning_off_instrumentation) |
| -fno-common |         | Do not treat global variable in C as common variables (allows ASan to instrument them) |

ASan-specific compile-time flags are passed via clang flag `-mllvm <flag>`. In most cases you don't need them.

| flag| default | description |
|:----|:--------|:------------|
| -asan-stack   | 1       | Detect overflow/underflow for stack objects |
| -asan-globals | 1       | Detect overflow/underflow for global objects |
| -asan-use-private-alias | 0       | Use private aliases for global objects |

# Run-time flags

Most run-time flags are passed to [AddressSanitizer](AddressSanitizer) via `ASAN_OPTIONS` environment variable like this:
```
ASAN_OPTIONS=verbosity=1:malloc_context_size=20 ./a.out
```
but you could also embed default flags in the source code by implementing __asan_default_options function:
```
const char *__asan_default_options() {
  return "verbosity=1:malloc_context_size=20";
}
```

Note that the list below list may be (and probably is) incomplete. Also older versions of ASan may not support some of the listed flags. To get the idea of what's supported in your version, run
```
ASAN_OPTIONS=help=1 ./a.out
```

For the list of common sanitizer options see [SanitizerCommonFlags](SanitizerCommonFlags)

| **Flag** | **Default value** | **Description** |
|:---------|:------------------|:----------------|
|quarantine_size | -1       | Deprecated, please use quarantine_size_mb.|
|quarantine_size_mb | -1       | Size (in Mb) of quarantine used to detect use-after-free errors. Lower value may reduce memory usage but increase the chance of false negatives.|
|redzone | 16       | Minimal size (in bytes) of redzones around heap objects. Requirement: redzone >= 16, is a power of two.|
|max_redzone | 2048     | Maximal size (in bytes) of redzones around heap objects.|
|debug | false    | If set, prints some debugging information and does additional checks.|
|report_globals | 1        | Controls the way to handle globals (0 - don't detect buffer overflow on globals, 1 - detect buffer overflow, 2 - print data about registered globals).|
|check_initialization_order | false    | If set, attempts to catch initialization order issues.|
|replace_str | true     | If set, uses custom wrappers and replacements for libc string functions to find more errors.|
|replace_intrin | true     | If set, uses custom wrappers for memset/memcpy/memmove intinsics.|
|detect_stack_use_after_return | false    | Enables stack-use-after-return checking at run-time.|
|min_uar_stack_size_log | 16       | Minimum fake stack size log.|
|max_uar_stack_size_log | 20       | Maximum fake stack size log.|
|uar_noreserve | false    | Use mmap with 'noreserve' flag to allocate fake stack.|
|max_malloc_fill_size | 4096     | ASan allocator flag. max_malloc_fill_size is the maximal amount of bytes that will be filled with malloc_fill_byte on malloc.|
|malloc_fill_byte | 0xbe     | Value used to fill the newly allocated memory.|
|allow_user_poisoning | true     | If set, user may manually mark memory regions as poisoned or unpoisoned.|
|sleep_before_dying | true     | Number of seconds to sleep between printing an error report and terminating the program. Useful for debugging purposes (e.g. when one needs to attach gdb).|
|check_malloc_usable_size | true     | Allows the users to work around the bug in Nvidia drivers prior to 295.*.|
|unmap_shadow_on_exit | false    | If set, explicitly unmaps the (huge) shadow at exit.|
|protect_shadow_gap | true     | If set, mprotect the shadow gap|
|print_stats | false    | Print various statistics after printing an error message or if atexit=1.|
|print_legend | true     | Print the legend for the shadow bytes.|
|atexit | false    | If set, prints ASan exit stats even after program terminates successfully.|
|print_full_thread_history | true     | If set, prints thread creation stacks for the threads involved in the report and their ancestors up to the main thread.|
|poison_heap | true     | Poison (or not) the heap memory on [de]allocation. Zero value is useful for benchmarking the allocator or instrumentator.|
|poison_partial | true     | If true, poison partially addressable 8-byte aligned words (default=true). This flag affects heap and global buffers, but not stack buffers.|
|poison_array_cookie | true     | Poison (or not) the array cookie after operator new[].|
|alloc_dealloc_mismatch | true (false on Darwin and Windows) | Report errors on malloc/delete, new/free, new/delete[], etc.|
|new_delete_type_mismatch | true     | Report errors on mismatch betwen size of new and delete.|
|strict_init_order | false    | If true, assume that dynamic initializers can never access globals from other modules, even if the latter are already initialized.|
|start_deactivated | false    | If true, ASan tweaks a bunch of other flags (quarantine, redzone, heap poisoning) to reduce memory consumption as much as possible, and restores them to original values when the first instrumented module is loaded into the process. This is mainly intended to be used on Android. |
|detect_invalid_pointer_pairs | 0        | If non-zero, try to detect operations like <, <=, >, >= and - on invalid pointer pairs (e.g. when pointers belong to different objects). The bigger the value the harder we try.|
|detect_container_overflow | true     | If true, honor the container overflow  annotations. See https://github.com/google/sanitizers/wiki/AddressSanitizerContainerOverflow|
|detect_odr_violation | 2        | If >=2, detect violation of One-Definition-Rule (ODR); If ==1, detect ODR-violation only if the two variables have different sizes|
|dump_instruction_bytes | false    | If true, dump 16 bytes starting at the instruction that caused SEGV|
|suppressions | ""       | Suppressions file name.|
|halt_on_error | true     | Crash the program after printing the first error report (WARNING: USE AT YOUR OWN RISK!)|
|log_path|stderr|Write logs to `log_path.pid`. The special values are `stdout` and `stderr`|
|use_odr_indicator | false       | Use special ODR indicator symbol for ODR violation detection.|
|allocator_frees_and_returns_null_on_realloc_zero|true|realloc(p, 0) is equivalent to free(p) by default (Same as the POSIX standard). If set to false, realloc(p, 0) will return a pointer to an allocated space which can not be used.|
|verify_asan_link_order|true|Check position of ASan runtime in library list (needs to be disabled when other library has to be preloaded system-wide)|