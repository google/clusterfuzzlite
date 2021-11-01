---
layout: default
parent: Running ClusterFuzzLite
grand_parent: ClusterFuzzLite
title: Google Cloud Build
nav_order: 2
permalink: /running-clusterfuzzlite/google-cloud-build/
---
# Google Cloud Build
{: .no_toc}

- TOC
{:toc}
---

This document explains how to set up ClusterFuzzLite on
[Google Cloud Build (GCB)](https://cloud.google.com/build).
This document assumes the reader has set up builds using GCB before.
Documentation on using GCB is available
[here](https://cloud.google.com/build/docs/quickstart-build).
This document also assumes your code is hosted on GitHub, but it should be
mostly applicable if your code is hosted elsewhere.

We recommend having separate workflow files for each part of ClusterFuzzLite:

- `.clusterfuzzlite/cflite_build.yml` (for building/continuous fuzzing)
- `.clusterfuzzlite/cflite_pr.yml` (for PR fuzzing)
- `.clusterfuzzlite/cflite_batch.yml` (for batch fuzzing)
- `.clusterfuzzlite/cflite_prune.yml` (for corpus pruning)
- `.clusterfuzzlite/cflite_coverage.yml` (for coverage reports)

First, you must
[create a Google Cloud Storage Bucket](https://cloud.google.com/storage/docs/creating-buckets).
This will be refered to as
`<your-cloud-bucket>` in the config examples in this document. If
The bucket will be used to store crashes, builds, corpora, and coverage reports.
You will also need to  set the `REPOSITORY` environment variable.
The value for `REPOSITORY` will be denoted by `<your-repo-name>` in this
document.
Note that this must be the same as the directory in `/src` where your project's
code is located.
Also note that the configuration examples set the environment variables
`WORKSPACE`, `FILESTORE`, and `PROJECT_SRC_PATH`, these values are general for
all GCB users. Do not change them.

TODO: Host a clean, complete example somewhere.  TODO: multiple sanitizers

## Continuous builds (required)

TODO: Maybe lets get rid of this and have pruning or batch fuzzing upload a
build instead?
Continuous builds are used whenever a crash is found during PR or batch fuzzing
to determine if this crash was newly introduced.

Add the following to `.clusterfuzzlite/cflite_build.yml`:

{% raw %}
```yaml
steps:
  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-build-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'LANGUAGE=c++'
      - 'UPLOAD_BUILD=True'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'
```
{% endraw %}

Set up a [trigger](https://cloud.google.com/build/docs/automating-builds/create-manage-triggers)
to run this job using the [console](https://console.cloud.google.com/cloud-build/triggers/add).
Select the "Push to a branch" event and select your repo's main branch.
This trigger will cause a build to be saved for each commit to your repo's main
branch.


## PR fuzzing

To add a fuzzing workflow that runs on all pull requests to your project, add
the following to `.clusterfuzzlite/cflite_pr.yml`:

{% raw %}
```yaml
steps:
  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-build-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'LANGUAGE=c++'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'

  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'LANGUAGE=c++'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'
```
{% endraw %}

Set up a trigger to run this workflow on the "pull request" event so that it can
fuzz code changes when they are in review.
You can set the `FUZZ_SECONDS` value when running the fuzzers to change the
amount of time ClusterFuzzLite will fuzz PRs for.

## Batch fuzzing

To enable batch fuzzing, add the following to
`.clusterfuzzlite/cflite_batch.yml`:

This can be run on either pushes to your default branch, but we recommend
running it on a cron schedule.

{% raw %}
```yaml
steps:
  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-build-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'LANGUAGE=c++'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'

  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'LANGUAGE=c++'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'
      - 'MODE=batch
      - 'FUZZ_SECONDS=3600'  # You can change this to a value you prefer.
```

{% endraw %}

Set up a trigger that runs this workflow *manually*.
Then follow the
[documentation](https://cloud.google.com/build/docs/automating-builds/create-scheduled-triggers)
on making the workflow run on a schedule.

You can set the `FUZZ_SECONDS` value when running the fuzzers to change the
amount of time ClusterFuzzLite will fuzz PRs for.

## Corpus pruning

Corpus pruning minimizes a corpus by removing redundant items while keeping the
same code coverage. To enable this, add the following to `.clusterfuzzlite/cflite_cron.yml`:

{% raw %}
```yaml
jobs:
steps:
  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-build-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'LANGUAGE=c++'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'

  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'LANGUAGE=c++'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'
      - 'MODE=prune'
```
{% endraw %}


## Coverage reports

Periodic coverage reports can also be generated using the latest corpus. To
enable this, add the following to `.clusterfuzzlite/cflite_coverage.yml`:

{% raw %}
```yaml
steps:
  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-build-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'LANGUAGE=c++'
      - 'SANITIZER=coverage'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'

  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'LANGUAGE=c++'
      - 'MODE=coverage
      - 'SANITIZER=coverage'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'
```
{% endraw %}

To run this workflow periodically: set up a trigger that runs this workflow *manually*.
Then follow the
[documentation](https://cloud.google.com/build/docs/automating-builds/create-scheduled-triggers)
on making the workflow run on a schedule.
We reccommend scheduling coverage reports to run once a day.
Note that it's better to schedule this to run after pruning has
completed, but not required.

## Testing it Out

You can test each of the workflows by triggering the builds manually either
using the [web interface](https://console.cloud.google.com/cloud-build/triggers)
or on the command line using [gcloud](https://cloud.google.com/sdk/gcloud).
`gcloud` can either be used to
[submit the build to run on Google Cloud](https://cloud.google.com/build/docs/running-builds/start-build-command-line-api),
or you can simulate the build locally using
[cloud-build-local](https://cloud.google.com/build/docs/build-debug-locally).


## Storage Layout

Artifacts produced by ClusterFuzzLite are stored in `<your-cloud-bucket>`.
The bucket has the following layout.
1. Coverage report: <your-cloud-bucket>/coverage/latest/report/linux/index.html
1. Builds: <your-cloud-bucket>/build/<sanitizer>-<commit>/
1. Corpora: <your-cloud-bucket>/corpus/<fuzz-target>/
1. Crashes: <your-cloud-bucket>/crashes/<fuzz-target>/<sanitizer>/<crash-file> and <your-cloud-bucket>/crashes/<fuzz-target>/<sanitizer>/<crash-file>.summary (output from crash).

TODO: Viewing crashes
TODO: Configure private bucket to view coverage report.
TODO: empty .gcloudignore https://github.com/GoogleCloudPlatform/cloud-builders/issues/236
