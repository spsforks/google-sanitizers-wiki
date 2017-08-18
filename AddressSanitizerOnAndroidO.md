Current aosp/o-preview and aosp/master branches support running applications under AddressSanitizer on fully locked production devices.

See [this patch](https://android-review.googlesource.com/#/c/platform/frameworks/base/+/264478/) for the details of the implementation. To use this feature,
1. add "android:debuggable" to the application manifest
2. add libclang_rt.asan-${arch}-android.so library found in the NDK to the native libraries directory
3. add wrap.sh file (see contents below) to the same directory.

Note that multi-arch APKs need to repeat steps 2 and 3 for each architecture.
In steps 2 and 3, substitute ${arch} with the target architecture name, which can be found by running `readelf -d native.so | grep libclang_rt` on any native library in your application. Note that architecture name is not the same as Android ABI name.

## wrap.sh
    #!/system/bin/sh
    HERE="$(cd "$(dirname "$0")" && pwd)"
    export ASAN_OPTIONS=log_to_syslog=false,allow_user_segv_handler=1
    export LD_PRELOAD=$HERE/libclang_rt.asan-${arch}-android.so
    $@