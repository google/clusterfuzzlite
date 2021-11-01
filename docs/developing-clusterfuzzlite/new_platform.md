---
layout: default
parent: Developing ClusterFuzzLite
grand_parent: ClusterFuzzLite
title: Adding New Platform Support
nav_order: 1
permalink: /developing-clusterfuzzlite/new-platform/
---
# GitHub Actions
{: .no_toc}

- TOC
{:toc}
---

Ideally, new platforms can be supported without requiring any changes to the
ClusterFuzzLite code base. In most cases however some changes are required or
desirable to make configuration work. For example, a new CI platform might call
one of the configuration variables needed by ClusterFuzzLite by a different
name. For example, GitHub actions refers to `GIT_BASE_COMMIT` as
`GITHUB_BASE_COMMIT`. Handling configuration differences is easy in
ClusterFuzzLite. Add a new module to `infra/cifuzz/platform_config/` that
implements a `PlatformConfig` class. You can refer to the [platform_config
module for Google Cloud Build (GCB)] as an example.
Then specify the name of your module using the `CFL_PLATFORM` environment
variable when running ClusterFuzzLite.

[platform_config module for Google Cloud Build (GCB)]: https://github.com/google/oss-fuzz/blob/master/infra/cifuzz/platform_config/gcb.py
