Current aosp/o-preview and aosp/master branches support running applications under AddressSanitizer on fully locked production devices.

See [this patch](https://android-review.googlesource.com/#/c/platform/frameworks/base/+/264478/) and [documentation](https://developer.android.com/ndk/guides/wrap-script.html) for the details of the implementation. To use this feature,
1. follow instructions in [[AddressSanitizerOnAndroid#building]] 
2. add "android:debuggable" to the application manifest
3. add libclang_rt.asan-${arch}-android.so library found in the NDK to the native libraries directory
4. add wrap.sh file (see contents below) to the same directory.

Note that multi-arch APKs need to repeat steps 3 and 4 for each architecture.

In steps 3 and 4, substitute ${arch} with the target architecture name, which can be found by running `readelf -d native.so | grep libclang_rt` on any native library in your application. Note that architecture name is not the same as Android ABI name.

## wrap.sh
    #!/system/bin/sh
    HERE="$(cd "$(dirname "$0")" && pwd)"
    export ASAN_OPTIONS=log_to_syslog=false,allow_user_segv_handler=1
    export LD_PRELOAD=$HERE/libclang_rt.asan-${arch}-android.so
    $@

## Comparison with [[asan_device_setup|AddressSanitizerOnAndroid]]

Wrap.sh is the **recommended** method of running ASan where supported by the OS because it works on production devices and does not require any special preparation and device setup steps. Correctly build APK should work out of the box on any device with a recent enough Android OS.

Any process created by the wrap.sh-enabled application performs a cold startup of the VM instead of simply forking the Zygote. This adds a certain delay at the application startup, and any time the application starts a new process. Asan_device_setup method preloads ASan runtime library directly into Zygote, and does not suffer from this issue.
