---
layout: default
parent: ClusterFuzzLite
title: Overview
has_children: false
nav_order: 2
permalink: /overview/
---
# Overview

## Introducing fuzzing with libFuzzer and Sanitizers.

This section provides an overview of the fuzzing process and defines common terms. 
If you are already familiar with libFuzzer and Sanitizers, feel free to skip to [Build Integration] 
to begin writing fuzzers and integrating with ClusterFuzzLite's build system.

### Fuzzing

[Fuzzing] is a technique where randomized inputs are automatically created and
fed as input to a (target) program in order to find bugs in that program.
The program that creates the inputs is called a fuzzer.
Fuzzing is highly effective at finding bugs missed by manually written tests,
code review, or auditing.
Fuzzing has found thousands of bugs in mature software such as Chrome, OpenSSL,
and Curl. 
When done well, fuzzing is able to find bugs in virtually any code.

### libFuzzer

[LibFuzzer] is a fuzzer (sometimes called a fuzzing engine) that mutates inputs
and feeds them to target code in a loop.
During execution of the target on the input, libFuzzer observes the coverage of
the code under test using instrumentation inserted by the compiler.
LibFuzzer uses this coverage feedback to "evolve" progressively more interesting
inputs and reach deeper program states, allowing it to find interesting bugs
with little developer effort.

### Fuzz target

To fuzz target code, you must define a function called a [fuzz target] with the
following API:

```C++
// fuzz_target.cc
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  DoSomethingInterestingWithMyAPI(Data, Size);
  return 0;  // Non-zero return values are reserved for future use.
}
```
Clang's `-fsanitizer=fuzzer` option will link this fuzz target function against
libFuzzer, producing a fuzzer binary that will fuzz your target code when run.
Note that in ClusterFuzzLite, you will not use this flag directly. Instead, you should
use the `$LIB_FUZZING_ENGINE` environment variable, which is discussed in more detail in
[Build Integration].

### Sanitizers

Sanitizers are tools that detect bugs in code (typically "native code" such as
C/C++, Rust, Go, and Swift) and report bugs by crashing.
ClusterFuzzLite relies on sanitizers to detect bugs that would otherwise be
missed.
Sanitizers work by instructing clang to add compile-time instrumentation, so different
builds are needed to use different sanitizers.

The sanitizers ClusterFuzzLite uses are:
- [AddressSanitizer (ASan)] : For detecting memory safety issues. This is the
  most important sanitizer to fuzz with. AddressSanitizer also detects memory
  leaks.
- [UndefinedBehaviorSanitizer (UBSan)] : For detecting undefined behavior such
  as integer overflows.
- [MemorySanitizer (MSan)] : For detecting use of uninitialized memory. MSan
  is the hardest sanitizer to use because an MSan instrumented binary must be
  entirely instrumented with MSan. If any part of the binary is not instrumented
  with MSan, MSan will report false positives.

The ClusterFuzzLite codebase uses shorter names for the sanitizers. When
referring to a sanitizer as an input to ClusterFuzzLite, ASan is
`address`, UBSan is `ubsan` and MSan is `memory`.

Next: [Step 1: Build Integration] for directions on integrating your project with ClusterFuzzLite's build system. 

[Fuzzing]: https://en.wikipedia.org/wiki/Fuzzing
[LibFuzzer]: https://llvm.org/docs/LibFuzzer.html
[fuzz target]: https://github.com/google/fuzzing/blob/master/docs/glossary.md#fuzz-target
[AddressSanitizer (ASan)]: https://clang.llvm.org/docs/AddressSanitizer.html
[UndefinedBehaviorSanitizer (UBSan)]: https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html
[MemorySanitizer (MSan)]: https://clang.llvm.org/docs/MemorySanitizer.html
[Step 1: Build Integration]: {{ site.baseurl }}/build-integration/
