---
layout: default
title: ClusterFuzzLite
has_children: true
nav_order: 8
permalink: /
---

# ClusterFuzzLite
ClusterFuzzLite is a lightweight, easy-to-use, fuzzing infrastructure that is
based off [ClusterFuzz]. ClusterFuzzLite is designed to run as workflows on
[continuous integration] (CI) systems, which means it is easy to set up and
provides a familiar interface for users.

Currently ClusterFuzzLite fully supports [GitHub Actions]. Extending support to
other CI systems is in the works.

See [Overview] for a more detailed description of how ClusterFuzzLite works and
how you can use it.

[continuous integration]: https://en.wikipedia.org/wiki/Continuous_integration
[fuzzing]: https://en.wikipedia.org/wiki/Fuzzing
[ClusterFuzz]: https://google.github.io/clusterfuzz/
[GitHub Actions]: https://docs.github.com/en/actions
[Overview]: {{ site.baseurl }}/overview/
