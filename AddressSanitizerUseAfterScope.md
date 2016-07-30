
# Introduction

**Stack-use-after-scope** bug appears when a stack object is used outside the scope
it was defined.
Example (see also [AddressSanitizerExampleUseAfterScope](AddressSanitizerExampleUseAfterScope)):
```
void f() {
  int *p;
  if (b) {
    int x[10];
    p = x;
  }
  *p = 1;
}
```

[AddressSanitizer](AddressSanitizer) currently does not enable detection of these bugs by default.
To enable this check you can use clang flag -fsanitize-address-use-after-scope.

# Algorithm
[AddressSanitizer](AddressSanitizer) detects this kind of bugs by marking memory used by local variables
as good when control reached variable definitions. Then it marks memory as bad when control reaches the
end of the scope of definition. Implementation relies on @llvm.lifetime.start and @llvm.lifetime.end.

Example above we will be changed into a code similar to the following:
```
void f() {
  int *p;
  if (b) {
    __asan_unpoison_stack_memory(x);
    int x[10];
    p = x;
    __asan_poison_stack_memory(x);
  }
  *p = 1;
   __asan_unpoison_stack_memory(frame);
}
```
Before a function returned, its stack memory need to be unpoisoned to avoid false reports for
non-instrumented code.

# Memory consumption
Memory consumption is the same as with default set of [AddressSanitizerFlags](AddressSanitizerFlags).

# Performance
TODO

# Compatibility
TODO