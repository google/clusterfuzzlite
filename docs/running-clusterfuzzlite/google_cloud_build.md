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
To get the most of this page, you should have already set up your
[build integration] and read the more
[high-level document on running ClusterFuzzLite].
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
`<your-cloud-bucket>` in the config examples in this document.
The bucket will be used to store crashes, builds, corpora, and coverage reports.
You will also need to  set the `REPOSITORY` environment variable.
The value for `REPOSITORY` will be denoted by `<your-repo-name>` in this
document.
Note that this must be the same as the directory in `/src` where your project's
code is located.
Also note that the configuration examples sets the environment variable:
`CFL_PLATFORM` this value are general for all GCB users.
Do not change them.

<!-- TODO: Host a clean, complete example somewhere. -->

## Continuous builds (required)

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
      - 'SANITIZER=address' # This can be changed to other sanitizers you use.
      - 'LANGUAGE=c++' # Change this to your project's language.
      - 'UPLOAD_BUILD=True'
      - 'CFL_PLATFORM=gcb'
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
      - 'SANITIZER=address' # This can be changed to other sanitizers you use.
      - 'LANGUAGE=c++' # Change this to your project's language.
      - 'CFL_PLATFORM=gcb'

  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'LANGUAGE=c++' # Change this to your project's language.
      - 'CFL_PLATFORM=gcb'
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
      - 'SANITIZER=address' # This can be changed to other sanitizers you use.
      - 'LANGUAGE=c++' # Change this to your project's language.
      - 'CFL_PLATFORM=gcb'

  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'SANITIZER=address' # This can be changed to other sanitizers you use.
      - 'LANGUAGE=c++' # Change this to your project's language.
      - 'CFL_PLATFORM=gcb'
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
      - 'SANITIZER=address' # This can be changed to other sanitizers you use.
      - 'LANGUAGE=c++' # Change this to your project's language.
      - 'CFL_PLATFORM=gcb'

  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'SANITIZER=address' # This can be changed to other sanitizers you use.
      - 'LANGUAGE=c++' # Change this to your project's language.
      - 'CFL_PLATFORM=gcb'
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
      - 'LANGUAGE=c++' # Change this to your project's language.
      - 'SANITIZER=coverage'
      - 'CFL_PLATFORM=gcb'

  - name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'REPOSITORY=<your-repo-name>'
      - 'LANGUAGE=c++' # Change this to your project's language.
      - 'MODE=coverage
      - 'SANITIZER=coverage'
      - 'CFL_PLATFORM=gcb'
```
{% endraw %}

To run this workflow periodically: set up a trigger that runs this workflow
*manually*.
Then follow the
[documentation](https://cloud.google.com/build/docs/automating-builds/create-scheduled-triggers)
on making the workflow run on a schedule.
We recommend scheduling coverage reports to run once a day.

## Viewing crashes

Google Cloud Build does not offer an easy way to download files associated with
a build. Therefore, to download crashes that were found by ClusterFuzzLite,
inspect the logs to find the name of the crash file and then download it them
from `<your-cloud-bucket>/crashes/<fuzzer>/<sanitizer>/<crash-file>`
where `<crash-file>` is the name of the crash that was found by ClusterFuzzLite,
`<fuzzer>` is the fuzzer that found the crash, and `<sanitizer>` is the
sanitizer used to find the crash.
Note that these files can be downloaded using a web browser by nagivating to
`https://console.cloud.google.com/storage/browser/<your-cloud-bucket-without-gs>/crashes/<fuzzer>/<sanitizer>`
Where `<your-cloud-bucket-without-gs>` is `<your-cloud-bucket>` without `gs://`
at the beginning.
For example, if `<your-cloud-bucket>` is
`gs://clusterfuzzlite-storage` `<your-cloud-bucket-without-gs>` is
`clusterfuzzlite-storage`.

## Testing it Out

You can test each of the workflows by triggering the builds manually either
using the [web interface](https://console.cloud.google.com/cloud-build/triggers)
or on the command line using [gcloud](https://cloud.google.com/sdk/gcloud).
`gcloud` can either be used to
[submit the build to run on Google Cloud](https://cloud.google.com/build/docs/running-builds/start-build-command-line-api),
or you can simulate the build locally using
[cloud-build-local](https://cloud.google.com/build/docs/build-debug-locally).
If you want to submit the builds to run on Google Cloud manually, please
create an empty file named `.gcloudignore` in the root of your repository
or see [this github issue] because Google Cloud Build will not by default
include the `.git` folder of your repository for manually submitted builds.

## Storage Layout

Artifacts produced by ClusterFuzzLite are stored in `<your-cloud-bucket>`.
The bucket has the following layout.
- Crashes:
    - `<your-cloud-bucket>/crashes/<fuzzer>/<sanitizer>/<crash-file>`
    - `<your-cloud-bucket>/crashes/<fuzzer>/<sanitizer>/<crash-file>.summary` (output from crash).
- Coverage report: `<your-cloud-bucket>/coverage/latest/report/linux/index.html`
- Corpora: `<your-cloud-bucket>/corpus/<fuzzer>/`
- Builds: `<your-cloud-bucket>/build/<sanitizer>-<commit>/`
Note that the bucket can be explored using a web browser by navigating to:
`https://console.cloud.google.com/storage/browser/<your-cloud-bucket-without-gs>`
Where `<your-cloud-bucket-without-gs>` is `<your-cloud-bucket>` without `gs://`
at the beginning.
For example, if `<your-cloud-bucket>` is
`gs://clusterfuzzlite-storage` `<your-cloud-bucket-without-gs>` is
`clusterfuzzlite-storage`.

<!-- TODO(ochang): Investigate if it's possible to configure private bucket to
allow viewing coverage reports in browser. -->

[this github issue]: https://github.com/GoogleCloudPlatform/cloud-builders/issues/236#issuecomment-374629200
[build integration]: {{ site.baseurl }}/build-integration/
[high-level document on running ClusterFuzzLite]: {{ site.baseurl }}/running-clusterfuzzlite/
