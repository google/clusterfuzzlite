---
layout: default
title: ClusterFuzzLite
has_children: true
nav_order: 1
permalink: /
---

# ClusterFuzzLite

ClusterFuzzLite makes [fuzzing] part of [Continuous Integration (CI)].
ClusterFuzzLite is based on [ClusterFuzz].

Fuzzing is a highly effective technique for finding bugs in software.
ClusterFuzzLite makes your code more secure by fuzzing changes before they even
enter the codebase.

ClusterFuzzLite supports [GitHub Actions] and [Google Cloud Build], so getting
the benefits of continuous fuzzing is simple if you can use these CI systems.
Suppport for more CI systems is in-progress and
[extending support to other CI systems] is easy. ClusterFuzzLite supports programming languages and runtimes beyond C and C++, including Java (and other JVM-based languages), Go, Python, Rust, and Swift.

ClusterFuzzLite offers many useful features, including:
- Quickly fuzzing code changes (pull requests) before they land and allowing
   you to download the crashing testcases.
- Continuous longer running fuzzing (batch fuzzing) that asynchronously find
   deeper bugs missed during code change fuzzing and build a minimal corpus for
   use in code change fuzzing.
- Coverage reports, so users can see which parts of their code is fuzzed.

ClusterFuzzLite is modular, so you can decide which features you want to use.

ClusterFuzzLite uses [libFuzzer] and important sanitizers for finding bugs in
C/C++ code:
- [AddressSanitizer], for finding memory safety issues.
- [MemorySanitizer], for finding use of uninitialized memory.
- [UndefinedBehaviorSanitizer], for finding undefined behavior (e.g. integer
  overflows).

If you're new to using libFuzzer and sanitizers, start with the [Overview] for an explanation of terms and the fuzzing process. 

If you're already familiar with using libFuzzer and sanitizers, you're ready to begin fuzzing with ClusterFuzzLite. Using ClusterFuzzLite is simple and requires two major steps:
1. [Writing fuzzers and integrating with ClusterFuzzLite's build system]
1. [Configuring ClusterFuzzLite to run in your CI]

[Continuous Integration (CI)]: https://en.wikipedia.org/wiki/Continuous_integration
[fuzzing]: https://en.wikipedia.org/wiki/Fuzzing
[ClusterFuzz]: https://google.github.io/clusterfuzz/
[GitHub Actions]: {{ site.baseurl }}/running-clusterfuzzlite/github-actions/
[Overview]: {{ site.baseurl }}/overview/
[Google Cloud Build]: {{ site.baseurl }}/running-clusterfuzzlite/google-cloud-build/
[extending support to other CI systems]:{{ site.baseurl }}/developing-clusterfuzzlite/new-platform/
[libFuzzer]: https://llvm.org/docs/LibFuzzer.html
[AddressSanitizer]: https://clang.llvm.org/docs/AddressSanitizer.html
[UndefinedBehaviorSanitizer]: https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html
[Writing fuzzers and integrating with ClusterFuzzLite's build system]: {{ site.baseurl }}/build-integration/
[Running ClusterFuzzLite]: {{ site.baseurl }}/running-clusterfuzzlite/
[Configuring ClusterFuzzLite to run in your CI]: {{ site.baseurl }}/running-clusterfuzzlite/
[MemorySanitizer]: https://clang.llvm.org/docs/MemorySanitizer.html
