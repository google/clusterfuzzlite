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

## Configuration

Ideally, new platforms can be supported without requiring any changes to the
ClusterFuzzLite code base.
In most cases however some changes are required or desirable to make
configuration work.
For example, a new CI platform might call one of the configuration variables
needed by ClusterFuzzLite by a different name.
For example, GitHub actions refers to `GIT_BASE_COMMIT` as `GITHUB_BASE_COMMIT`.
Handling configuration differences is easy in ClusterFuzzLite.
Add a new module to [`infra/cifuzz/platform_config/`] that implements a
`PlatformConfig` class.
Your `PlatformConfig` must:
- Be named `PlatformConfig`
- Be defined in a new submodule within [`infra/cifuzz/platform_config/`]
- Inherit from `BasePlatformConfig` which is defined in
[`infra/cifuzz/platform_config/__init__.py`]
You can refer to the [platform_config module for Google Cloud Build (GCB)] as an
example.
Then specify the name of your module using the `CFL_PLATFORM` environment
variable when running ClusterFuzzLite.
One of the more important configuration properties your `PlatformConfig` class
should define is a [filestore](#filestore).
It may be better to let users override this, but it is not required.

## Filestore

The filestore configuration property determines which filestore implementation
is used by ClusterFuzzLite. The filestore implementation, as the name suggests,
is responsible for storing files that persist after ClusterFuzzLite finishes
running a task.
These files include:
- Crashes
- Corpuses
- Builds
- Coverage
Implementing a filestore may be necessary or desirable for your
platform.

### Existing filestores
If your platform does not offer a way of storing files you can either
reuse one of the existing filestores such as `gsutil` which supports [Google
Cloud Storage Buckets] and [Amazon S3 Buckets] or use the `no_filestore`
implementation.
The `no_filestore` implementation of every filestore operation is a no-op.
Using ClusterFuzzLite with `no_filestore` should allow you to run
ClusterFuzzLite on your platform without any exceptions, but without
most of the important features of ClusterFuzzLite.

### Implementing a new filestore

You can refer to [the filestore module for gsutil] as an example for
implementing your filestore.
To implement a new filestore:
- Create a new subdirectory in [`infra/cifuzz/filestore`] named for your
  filestore.
- Create a file `__init__.py` in this subdirectory that implements the filestore
  class (which can be named anything).
- Your filestore class must inherit from `BaseFilestore` which is defined in
  [`infra/cifuzz/filestore/__init__.py`].
- Add a mapping from the filestore name to the class you implemented in `FILESTORE_MAPPING` in 
  [`infra/cifuzz/filestore_utils.py`]

[platform_config module for Google Cloud Build (GCB)]: https://github.com/google/oss-fuzz/blob/master/infra/cifuzz/platform_config/gcb.py
[`infra/cifuzz/platform_config/`]: https://github.com/google/oss-fuzz/tree/master/infra/cifuzz/platform_config
[`infra/cifuzz/platform_config/__init__.py`]: https://github.com/google/oss-fuzz/tree/master/infra/cifuzz/platform_config/__init__.py
[the filestore module for gsutil]: https://github.com/google/oss-fuzz/blob/master/infra/cifuzz/filestore/gsutil/__init__.py
[`infra/cifuzz/filestore`]: https://github.com/google/oss-fuzz/tree/master/infra/cifuzz/filestore
[`infra/cifuzz/filestore/__init__.py`]: https://github.com/google/oss-fuzz/blob/master/infra/cifuzz/filestore/__init__.py
[`infra/cifuzz/filestore_utils.py`]: https://github.com/google/oss-fuzz/blob/master/infra/cifuzz/filestore_utils.py
[Google Cloud Storage Buckets]: https://cloud.google.com/storage/docs/introduction
[Amazon S3 Buckets]: https://aws.amazon.com/s3/
