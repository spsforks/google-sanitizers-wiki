This is the list of common sanitizer options as of r254719.
Each tool parses the common options from the corresponding environment variable (`ASAN_OPTIONS`, `TSAN_OPTIONS`, `MSAN_OPTIONS`, `LSAN_OPTIONS`) together with the tool-specific options.

| **Flag** | **Default value** | **Description** |
|:---------|:------------------|:----------------|
|symbolize | true     | If set, use the online symbolizer from common sanitizer runtime to turn virtual addresses to file/line locations.|
|external_symbolizer_path | ""         | Path to external symbolizer. If empty, the tool will search $PATH for the symbolizer.|
|allow_addr2line | false    | If set, allows online symbolizer to run addr2line binary to symbolize stack traces (addr2line will only be used if llvm-symbolizer binary is unavailable.|
|strip_path_prefix | ""       | Strips this prefix from file paths in error reports.|
|fast_unwind_on_check | false    | If available, use the fast frame-pointer-based unwinder on internal CHECK failures.|
|fast_unwind_on_fatal | false    | If available, use the fast frame-pointer-based unwinder on fatal errors.|
|fast_unwind_on_malloc | true     | If available, use the fast frame-pointer-based unwinder on malloc/free.|
|handle_ioctl | false    | Intercept and handle ioctl requests.|
|malloc_context_size | 1        | Max number of stack frames kept for each allocation/deallocation.|
|log_path | stderr   | Write logs to "log_path.pid". The special values are "stdout" and "stderr". The default is "stderr".|
|log_exe_name | false    | Mention name of executable when reporting error and append executable name to logs (as in "log_path.exe_name.pid").|
|log_to_syslog | false (true on Android and Darwin)    | Write all sanitizer output to syslog in addition to other means of logging.|
|verbosity | 0        | Verbosity level (0 - silent, 1 - a bit of output, 2+ - more output).|
|detect_leaks | true     | Enable memory leak detection.|
|leak_check_at_exit | true     | Invoke leak checking in an atexit handler. Has no effect if detect_leaks=false, or if __lsan_do_leak_check() is called before the handler has a chance to run.|
|allocator_may_return_null | false    | If false, the allocator will crash instead of returning 0 on out-of-memory.|
|print_summary | true     | If false, disable printing error summaries in addition to error reports.|
|check_printf | true     | Check printf arguments.|
|handle_segv | true (false on Windows)  | If set, registers the tool's custom SIGSEGV/SIGBUS handler.|
|handle_abort | false    | If set, registers the tool's custom SIGABRT handler.|
|handle_sigfpe | true     | If set, registers the tool's custom SIGFPE handler.|
|allow_user_segv_handler | false    | If set, allows user to register a SEGV handler even if the tool registers one.|
|use_sigaltstack | true     | If set, uses alternate stack for signal handling.|
|detect_deadlocks | false    | If set, deadlock detection is enabled.|
|clear_shadow_mmap_threshold | 64 * 1024| Large shadow regions are zero-filled using mmap(NORESERVE) instead of memset(). This is the threshold size in bytes.|
|color | auto     | Colorize reports: (always|never|auto).|
|legacy_pthread_cond | false    | Enables support for dynamic libraries linked with libpthread 2.2.5.|
|intercept_tls_get_addr | false    | Intercept __tls_get_addr.|
|help | false    | Print the flag ptions.|
|mmap_limit_mb | 0        | Limit the amount of mmap-ed memory (excluding shadow) in Mb; not a user-facing flag, used mosly for testing the tools|
|hard_rss_limit_mb | 0        | Hard RSS limit in Mb. If non-zero, a background thread is spawned at startup which periodically reads RSS and aborts the process if the limit is reached|
|soft_rss_limit_mb | 0        | Soft RSS limit in Mb. If non-zero, a background thread is spawned at startup which periodically reads RSS. If the limit is reached all subsequent malloc/new calls will fail or return NULL (depending on the value of allocator_may_return_null) until the RSS goes below the soft limit. This limit does not affect memory allocations other than malloc/new.|
|can_use_proc_maps_statm | true     | If false, do not attempt to read /proc/maps/statm. Mostly useful for testing sanitizers.|
|coverage | false    | If set, coverage information will be dumped at program shutdown (if the coverage instrumentation was enabled at compile time).|
|coverage_pcs | true     | If set (and if 'coverage' is set too), the coverage information will be dumped as a set of PC offsets for every module.|
|coverage_order_pcs | false    | If true, the PCs will be dumped in the order they've appeared during the execution.|
|coverage_bitset | false    | If set (and if 'coverage' is set too), the coverage information will also be dumped as a bitset to a separate file.|
|coverage_counters | false    | If set (and if 'coverage' is set too), the bitmap that corresponds to coverage counters will be dumped.|
|coverage_direct | false (true on Android) | If set, coverage information will be dumped directly to a memory mapped file. This way data is not lost even if the process is suddenly killed.|
|coverage_dir | "."        | Target directory for coverage dumps. Defaults to the current directory.|
|full_address_space | false     | Sanitize complete address space; by default kernel area on 32-bit platforms will not be sanitized|
|print_suppressions | true     | Print matched suppressions at exit.|
|disable_coredump | true (false on non-64-bit systems)  | Disable core dumping. By default, disable_core=1 on 64-bit to avoid dumping a 16T+ core file. Ignored on OSes that don't dump core bydefault and for sanitizers that don't reserve lots of virtual memory.|
|use_madv_dontdump | true     | If set, instructs kernel to not store the (huge) shadow in core file.|
|symbolize_inline_frames | true     | Print inlined frames in stacktraces. Defaults to true.|
|symbolize_vs_style | false    | Print file locations in Visual Studio style (e.g:  file(10,42): ...|
|stack_trace_format | DEFAULT  | Format string used to render stack frames. See sanitizer_stacktrace_printer.h for the format description. Use DEFAULT to get default format.|
|no_huge_pages_for_shadow | true      | If true, the shadow is not allowed to use huge pages. |
|strict_string_checks | false    | If set check that string arguments are properly null-terminated|
|intercept_strstr | true     | If set, uses custom wrappers for strstr and strcasestr functions to find more errors.||symbolize | true     | If set, use the online symbolizer from common sanitizer runtime to turn virtual addresses to file/line locations.|
|intercept_strspn | true     | If set, uses custom wrappers for strspn and strcspn function to find more errors.|
|intercept_strpbrk | true     | If set, uses custom wrappers for strpbrk function to find more errors.|
|intercept_memcmp | true     | If set, uses custom wrappers for memcmp function to find more errors.|
|strict_memcmp | true     | If true, assume that memcmp(p1, p2, n) always reads n bytes before comparing p1 and p2.|
|decorate_proc_maps | false    | If set, decorate sanitizer mappings in /proc/self/maps with user-readable names|
|exitcode | 1        | Override the program exit status if the tool found an error|
|abort_on_error | false (true on Darwin)         | If set, the tool calls abort() instead of _exit() after printing the error report.|
|include |   ""       | read more options from the given file|
|include_if_exists | ""         | read more options from the given file (if it exists)|
|suppress_equal_pcs|  true      | Deduplicate multiple reports for single source location in halt_on_error=false mode (asan only).|
|print_cmdline|  false      | Print command line on crash (asan only).|
### See also
* [AddressSanitizerFlags](AddressSanitizerFlags)
* [ThreadSanitizerFlags](ThreadSanitizerFlags)