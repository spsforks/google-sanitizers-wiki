NOTE: _this document is about running Android applications built with the NDK under [AddressSanitizer](AddressSanitizer). For information about using [AddressSanitizer](AddressSanitizer) on Android platform components, see the [android documentation](http://source.android.com/devices/tech/debug/asan.html)._

NOTE: _ASan is supported in all versions of Android starting with KitKat, with the exception of the initial release of Lollipop (Lollipop MR1 update is fine)._

Android NDK supports [AddressSanitizer](AddressSanitizer) on arm, arm64, and x86 starting with version r10d, and x86_64 with later versions.

## Building

To build your app's native (JNI) code with [AddressSanitizer](AddressSanitizer), add the following to Android.mk:

```
LOCAL_CFLAGS    := -fsanitize=address -fno-omit-frame-pointer
LOCAL_LDFLAGS   := -fsanitize=address
LOCAL_ARM_MODE := arm
```

Note that [AddressSanitizer](AddressSanitizer) in NDK r10d does not support 64-bit ABIs, and compilation with `APP_ABI := all` will fail. Use
```
APP_ABI := armeabi armeabi-v7a x86
```
or a subset of those. This is not an issue with newer NDKs.

## Running

There are two ways of running ASan application on Android. The "wrap.sh" method is available and recommended beginning with Android O. Alternative "asan_device_setup" method requires a rooted device with writable system partition (dangerous!) but works on older devices as well as newer ones.

### Running with wrap.sh (recommended)
In Android O an application can provide a [script](https://developer.android.com/ndk/guides/wrap-script) that can wrap or replace the application process. It can be used to run ASan apps on fully locked, production devices.

A few extra steps are required to prepare an app to run with wrap.sh.

1. add "android:debuggable" to the application manifest
2. add libclang_rt.asan-${arch}-android.so library found in the NDK to the native libraries directory
3. add wrap.sh file with the following contents to the same directory.
~~~~
    #!/system/bin/sh
    HERE="$(cd "$(dirname "$0")" && pwd)"
    export ASAN_OPTIONS=log_to_syslog=false,allow_user_segv_handler=1
    export LD_PRELOAD=$HERE/libclang_rt.asan-${arch}-android.so
    "$@"
~~~~

Note that multi-arch APKs need to repeat steps 2 and 3 for each architecture.

In steps 2 and 3, substitute ${arch} with the target architecture name, which can be found by running `readelf -d native.so | grep libclang_rt` on any native library in your application. Note that architecture name is not the same as Android ABI name.

### Running with asan_device_setup
**WARNING**: this method edits the system partition on your device. It has been tested on some Pixels and Nexuses, but there is no guarantee that it will work correctly on any third party device. In the worst case the device may become unbootable.

Android NDK includes a script `asan_device_setup` that must be run once for any device you want to run ASan applications on. Once this is done, applications built with ASan can be installed and executed as any other.

Native binaries built with ASan must be prefixed with `LD_PRELOAD=libclang_rt.asan-arm-android.so` (replace `arm` with the target arch: `x86`, etc).


## Stack traces

[AddressSanitizer](AddressSanitizer) needs to unwind stack on every `malloc`/`realloc`/`free` call. There are 2 options here:

  * A "fast" frame pointer-based unwinder.
> It requires
```
  LOCAL_CFLAGS=-fno-omit-frame-pointer
  LOCAL_ARM_MODE := arm
  APP_STL := gnustl_shared  # or any other _shared option
```
> Implementations of operators new and delete in libstdc++ are usually built without frame pointer. ASan brings its own implementation, but it can't be used if the C++ stdlib is linked statically into the application.

  * A "slow" CFI unwinder.
> In this mode ASan simply calls `_Unwind_Backtrace`. It requires only `-funwind-tables`, which is normally enabled by default.
> Warning: the "slow" unwinder is SLOW (10x or more, depending on how ofter you call `malloc`/`free`).

Fast unwinder is the default for malloc/realloc/free. Slow unwinder is the default for "fatal" stack traces, i.e. in the case when an error is detected.
Slow unwinder can be enabled for all stack traces with
```
ASAN_OPTIONS=fast_unwind_on_malloc=0
```
## Under the hood

Wrap.sh is the **recommended** method of running ASan where supported by the OS because it works on production devices and does not require any special preparation and device setup steps. Correctly build APK should work out of the box on any device with a recent enough Android OS.

Any process created by the wrap.sh-enabled application performs a cold startup of the VM instead of simply forking the Zygote. This adds a certain delay at the application startup, and any time the application starts a new process.

On the other hand, Asan_device_setup uploads the ASan runtime library to `/system/lib`
and sets up the Zygote binary (`/system/bin/app_process`) in such a way that ASan runtime library is `LD_PRELOAD`-ed into it. This has the effect of replacing the system memory allocator with ASan allocator in *all applications*.

To preserve memory, asan_device_setup disabled most [AddressSanitizer](AddressSanitizer) features until the first library with ASan-instrumented code is loaded into the process memory. Normally, this happens in individual application processes (when the Zygote forks and loads the application-specific code), and does not affect the rest of applications on the device.

## Run-time flags

```
asan_device_setup --extra-options <options text>
```
> this flag can be used to set default ASAN\_OPTIONS for the Zygote process. Note that this will affect all applications on the device!

Additional per-application options can be specified in `/data/local/tmp/asan.options.%b`, where `%b` stands for the "nice name" of the application as seen in `ps` output. These additional options that are evaluated at the moment the first instrumented DSO is loaded into the Zygote process. Not all possible `ASAN_OPTIONS` can be set here. Setting options through `asan_device_setup` is preferable.

See [AddressSanitizerFlags](AddressSanitizerFlags) for the list of supported flags.