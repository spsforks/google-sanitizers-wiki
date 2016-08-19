This page describes how to use ThreadSanitizer on Android devices.
The instructions are raw. Ideally all that is integrated into llvm/android build systems. But that's what we have now.

download aosp_master by following instructions in
https://source.android.com/source/downloading.html

download compiler-­rt from github https://github.com/llvm­mirror/compiler­rt

1. replace aosp_master/external/compiler_rt with compiler­rt in upstream.
```
$mv aosp_master/external/compiler_rt aosp_master_compiler_rt
$cd aosp_master/external
$ln ­s ../../compiler_rt compiler_rt
```

2. copy build files from aosp_master compiler_rt to upstream compiler_rt.
```
$cd aosp_master_compiler_rt
$find . ­name Android.bp
./Android.bp
./lib/tsan/Android.bp
./lib/profile/Android.bp
./lib/sanitizer_common/Android.bp
./lib/sanitizer_common/tests/Android.bp
./lib/interception/Android.bp
./lib/ubsan/Android.bp
./lib/asan/Android.bp
./lib/lsan/Android.bp
```
copy all Android.bp from aosp_master_compiler_rt to corresponding directories in upstream
compiler_rt. except lib/asan/Android.bp (because it has a wrap file which doesn’t appear in
upstream)
```
$ cp Android.bp ../aosp_master/external/compiler­rt/
$ cp lib/tsan/Android.bp ../aosp_master/external/compiler­rt/lib/tsan/
$ cp lib/profile/Android.bp ../aosp_master/external/compiler­rt/lib/profile/
$ cp lib/sanitizer_common/Android.bp ../aosp_master/external/compiler­rt/lib/sanitizer_common/
$ cp lib/interception/Android.bp ../aosp_master/external/compiler­rt/lib/interception/
$ cp lib/ubsan/Android.bp ../aosp_master/external/compiler­rt/lib/ubsan/
$ cp lib/lsan/Android.bp ../aosp_master/external/compiler­rt/lib/lsan/
```

3. change build files
add following lines in lib/tsan/Android.bp, and remove tests build:
```
tsan_rtl_cppflags2 = [
    "­std=c++11",
    "­Wall",
    "­Werror",
    "­Wno­unused­parameter",
    "­Wno­non­virtual­dtor",
    "­fno­rtti",
    "­fno­builtin",
    "­DTSAN_CONTAINS_UBSAN=0",
]
cc_library_shared {
    name: "libtsan_shared",
    include_dirs: ["external/compiler­rt/lib"],
    cppflags: tsan_rtl_cppflags2,
    srcs: [
        "rtl/tsan_clock.cc",
        "rtl/tsan_flags.cc",
        "rtl/tsan_fd.cc",
        "rtl/tsan_ignoreset.cc",
        "rtl/tsan_interceptors.cc",
        "rtl/tsan_interface_ann.cc",
        "rtl/tsan_interface_atomic.cc",
        "rtl/tsan_interface.cc",
        "rtl/tsan_interface_java.cc",
        "rtl/tsan_md5.cc",
        "rtl/tsan_mman.cc",
        "rtl/tsan_mutex.cc",
        "rtl/tsan_mutexset.cc",
        "rtl/tsan_report.cc",
        "rtl/tsan_rtl.cc",
        "rtl/tsan_rtl_mutex.cc",
        "rtl/tsan_rtl_report.cc",
        "rtl/tsan_rtl_thread.cc",
        "rtl/tsan_stack_trace.cc",
        "rtl/tsan_stat.cc",
        "rtl/tsan_suppressions.cc",
        "rtl/tsan_symbolize.cc",
        "rtl/tsan_sync.cc",
        "rtl/tsan_platform_linux.cc",
        "rtl/tsan_platform_posix.cc",
        "rtl/tsan_new_delete.cc",
        "rtl/tsan_rtl_proc.cc",
        "rtl/tsan_rtl_aarch64.S",
    ],
    stl: "none",
    sanitize: {
        never: true,
    },
    compile_multilib: "64",
    whole_static_libs: [
        "libinterception_device",
        "libsan_device",
    ],
    shared_libs: ["libdl"],
    enabled: false,
    target: {
      android_arm64: {
        enabled: true,
      },
    },
}
```
remove tests build:
```
//cc_test_host {
//    name: "libtsan_unit_test",
//
//    include_dirs: ["external/compiler­rt/lib"],
//    local_include_dirs: ["rtl"],
//    cppflags: tsan_rtl_cppflags,
//    srcs: [
//        "tests/unit/tsan_clock_test.cc",
//        "tests/unit/tsan_dense_alloc_test.cc",
//        "tests/unit/tsan_flags_test.cc",
//        "tests/unit/tsan_mman_test.cc",
//        "tests/unit/tsan_mutex_test.cc",
//        "tests/unit/tsan_mutexset_test.cc",
//        "tests/unit/tsan_shadow_test.cc",
//        "tests/unit/tsan_stack_test.cc",
//        "tests/unit/tsan_sync_test.cc",
//        "tests/unit/tsan_unit_test_main.cc",
//        "tests/unit/tsan_vector_test.cc",
//    ],
//    sanitize: {
//        never: true,
//    },
//    compile_multilib: "64",
//    static_libs: [
//        "libtsan",
//        "libubsan",
//    ],
//    host_ldlibs: [
//        "­lrt",
//        "­ldl",
//    ],
//    target: {
//        darwin: {
//            enabled: false,
//        },
//    },
//}
//
//cc_test_host {
//    name: "libtsan_rtl_test",
//
//    include_dirs: ["external/compiler­rt/lib"],
//    local_include_dirs: ["rtl"],
//    cppflags: tsan_rtl_cppflags,
//    srcs: [
//        "tests/rtl/tsan_bench.cc",
//        "tests/rtl/tsan_mop.cc",
//        "tests/rtl/tsan_mutex.cc",
//        "tests/rtl/tsan_posix.cc",
//        "tests/rtl/tsan_string.cc",
//        "tests/rtl/tsan_test_util_posix.cc",
//        "tests/rtl/tsan_test.cc",
//        "tests/rtl/tsan_thread.cc",
//    ],
//    sanitize: {
//        never: true,
//    },
//    compile_multilib: "64",
//    static_libs: [
//        "libtsan",
//        "libubsan",
//    ],
//    host_ldlibs: [
//        "­lrt",
//        "­ldl",
//    ],
//    target: {
//        darwin: {
//            enabled: false,
//        },
//    },
//}
```

add following lines in `lib/interception/Android.bp`:
```
cc_library_static {
    name: "libinterception_device",
    clang: true,
    sdk_version: "19",
    include_dirs: ["external/compiler­rt/lib"],
    cppflags: [
        "­fvisibility=hidden",
        "­fno­exceptions",
        "­std=c++11",
        "­Wall",
        "­Werror",
        "­Wno­unused­parameter",
    ],
    srcs: [
        "interception_linux.cc",
        "interception_mac.cc",
        "interception_type_test.cc",
        "interception_win.cc",
    ],
    stl: "none",
    sanitize: {
        never: true,
    },
    compile_multilib: "both",
}
```

add following lines in `lib/sanitizer_common/Android.bp`:
```
cc_library_static {
    name: "libsan_device",
    clang: true,
    sdk_version: "19",
    include_dirs: ["external/compiler­rt/lib"],
    cppflags: [
        "­fvisibility=hidden",
        "­fno­exceptions",
        "­std=c++11",
        "­Wall",
        "­Werror",
        "­Wno­non­virtual­dtor",
        "­Wno­unused­parameter",
    ],
    srcs: [
        // rtl
        "sanitizer_allocator.cc",
        "sanitizer_common.cc",
        "sanitizer_deadlock_detector1.cc",
        "sanitizer_deadlock_detector2.cc",
        "sanitizer_flags.cc",
        "sanitizer_flag_parser.cc",
        "sanitizer_libc.cc",
        "sanitizer_libignore.cc",
        "sanitizer_linux.cc",
        "sanitizer_mac.cc",
        "sanitizer_persistent_allocator.cc",
        "sanitizer_platform_limits_linux.cc",
        "sanitizer_platform_limits_posix.cc",
        "sanitizer_posix.cc",
        "sanitizer_printf.cc",
        "sanitizer_procmaps_common.cc",
        "sanitizer_procmaps_freebsd.cc",
        "sanitizer_procmaps_linux.cc",
        "sanitizer_procmaps_mac.cc",
        "sanitizer_stackdepot.cc",
        "sanitizer_stacktrace.cc",
        "sanitizer_stacktrace_printer.cc",
        "sanitizer_suppressions.cc",
        "sanitizer_symbolizer.cc",
        "sanitizer_symbolizer_libbacktrace.cc",
        "sanitizer_symbolizer_win.cc",
        "sanitizer_tls_get_addr.cc",
        "sanitizer_thread_registry.cc",
        "sanitizer_win.cc",
        // cdep
        "sanitizer_common_libcdep.cc",
        "sanitizer_coverage_libcdep.cc",
        "sanitizer_coverage_mapping_libcdep.cc",
        "sanitizer_linux_libcdep.cc",
        "sanitizer_posix_libcdep.cc",
        "sanitizer_stacktrace_libcdep.cc",
        "sanitizer_stoptheworld_linux_libcdep.cc",
        "sanitizer_symbolizer_libcdep.cc",
        "sanitizer_symbolizer_posix_libcdep.cc",
        "sanitizer_unwind_linux_libcdep.cc",
        "sanitizer_termination.cc",
    ],
    stl: "none",
    sanitize: {
        never: true,
    },
    compile_multilib: "both",
}
```

4. bulid libtsan_shared.so
```
$cd aosp_master
$. build/envsetup.sh
$lunch aosp_arm64­userdebug
$mmma external/compiler­rt/lib/tsan ­j30
```
after building successfully, libtsan_shared.so for aarch64 is in
aosp_master/out/target/product/generic_arm64/system/lib64/libtsan_shared.so

5. Link app with libtsan_shared.so
Then you can use tsan by link libtsan_shared.so.
Note that libtsan_shared.so only works with libc.so on N devices or built in aosp_master. So you
may need to find a way to link libc.so built in aosp_master instead of the one running on device.

6. you may also need to build `llvm­-symbolizer` and download `llvm-­symbolizer` on device.
```
$mmma external/llvm/tools/llvm­-symbolizer/ ­j20
```