---
layout: default
title: ClusterFuzzLite
has_children: true
nav_order: 8
permalink: /
---

# ClusterFuzzLite

ClusterFuzzLite makes [fuzzing] part of [Continuous Integration (CI)].
ClusterFuzzLite is based on [ClusterFuzz].

By fuzzing your code, ClusterFuzzLite can make it more secure as fuzzing is
highly effective technique for finding bugs in software.

ClusterFuzzLite supports [GitHub Actions] and [Google Cloud Build].
Suppport for more CI systems is in-progress and
[extending support to other CI systems] is easy.

ClusterFuzzLite offers features that will make fuzzing an important part of your software development lifecycle, including:
1. Quickly fuzzing code changes (pull requests) before they land and allowing
   users to download the crashing testcases.
1. Asynchronous, long running fuzzing (batch fuzzing) that finds deeper bugs
   missed during code change fuzzing.
1. Coverage reports, so users can see which parts of their code is fuzzed.

See [Overview] for a more detailed description of how ClusterFuzzLite works and
how you can use it.

[Continuous Integration (CI)]: https://en.wikipedia.org/wiki/Continuous_integration
[fuzzing]: https://en.wikipedia.org/wiki/Fuzzing
[ClusterFuzz]: https://google.github.io/clusterfuzz/
[GitHub Actions]: {{ site.baseurl }}/running-clusterfuzzlite/github-actions/
[Overview]: {{ site.baseurl }}/overview/
[Google Cloud Build]: {{ site.baseurl }}/running-clusterfuzzlite/google-cloud-build/
[extending support to other CI systems]:{{ site.baseurl }}/developing-clusterfuzzlite/new-platform/
