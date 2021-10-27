---
layout: default
parent: Running ClusterFuzzLite
grand_parent: ClusterFuzzLite
title: Google Cloud Build
nav_order: 2
permalink: /running-clusterfuzzlite/google-cloud-build/
---
# GitHub Actions
{: .no_toc}

- TOC
{:toc}
---

This document explains how to set up ClusterFuzzLite to run on [Google Cloud Build (GCB)](https://cloud.google.com/build).
This document assumes the reader has set up builds using GCB before. 
Documentation on doing that is available [here](https://cloud.google.com/build/docs/quickstart-build).
This document also assumes your code is hosted on GitHub, but it should be
mostly applicable if your code is hosted elsewhere.

We recommend having separate workflow files for each part of ClusterFuzzLite:

- `.clusterfuzzlite/cflite_build.yml` (for building/continuous fuzzing)
- `.clusterfuzzlite/cflite_batch.yml` (for batch fuzzing)
- `.clusterfuzzlite/cflite_pr.yml` (for PR fuzzing)
- `.clusterfuzzlite/cflite_regular.yml` (for other regular tasks)

You should create a Google Cloud Storage Bucket. This will be refered to as
<your-cloud-bucket> in the config examples in this document. The bucket
will be used to store crashes, builds, corpora, and coverage reports.
You will also need to  set the `PROJECT_NAME` environment variable.
The value for your project will be denoted by `<your-project-name>` in this
document. Note that this must be the same as the directory in `/src` where
your project's code is located.
Note that in the configuration examples set the environment variables
`WORKSPACE` or `PROJECT_SRC_PATH`, these are general for all GCB users and you
should not change them.

TODO: Host a clean, complete example somewhere.  TODO: multiple sanitizers

## Continuous builds (required)

Continuous builds are used whenever a crash is found during PR or batch fuzzing
to determine if this crash was newly introduced.

Add the following to `.clusterfuzzlite/cflite_build.yml`:

{% raw %}
```yaml
steps:
  - name: gcr.io/oss-fuzz-base/cifuzz-build-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'PROJECT_NAME=<your-project-name>'
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


## PR fuzzing

To add a fuzzing workflow that runs on all pull requests to your project, add
the following to `.clusterfuzzlite/cflite_pr.yml`:

{% raw %}
```yaml
steps:
  - name: gcr.io/oss-fuzz-base/cifuzz-build-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'PROJECT_NAME=<your-project-name>'
      - 'LANGUAGE=c++'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'

  - name: gcr.io/oss-fuzz-base/cifuzz-run-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=<your-cloud-bucket>'
      - 'PROJECT_NAME=<your-project-name>'
      - 'LANGUAGE=c++'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'
      - 'MODE=code-change'
```
{% endraw %}

Set up a trigger to run this workflow on the "pull request" event so that it can fuzz
code changes when they are in review.

## Batch fuzzing

To enable batch fuzzing, add the following to
`.clusterfuzzlite/cflite_batch.yml`:

This can be run on either pushes to your default branch, or on a cron schedule (or both).

{% raw %}
```yaml
steps:
  - name: gcr.io/oss-fuzz-base/cifuzz-build-fuzzers:v1
    env:
      - 'CLOUD_BUCKET=gs://metzman-test/'
      - 'PROJECT_NAME=cifuzz-external-example'
      - 'LANGUAGE=c++'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'

  - name: gcr.io/oss-fuzz-base/cifuzz-run-fuzzers:v1
    env:
    
      - 'CLOUD_BUCKET=gs://metzman-test/'
      - 'PROJECT_NAME=cifuzz-external-example'
      - 'LANGUAGE=c++'
      - 'WORKSPACE=/builder/home'
      - 'PROJECT_SRC_PATH=/workspace'
      - 'FILESTORE=gsutil'
      - 'MODE=batch
      - 'FUZZ_SECONDS=3600'
```

{% endraw %}

Set up a trigger that runs this workflow *manually*.
Then follow the [documentation](https://cloud.google.com/build/docs/automating-builds/create-scheduled-triggers) on making the workflow run on a schedule.

## Coverage reports

Periodic coverage reports can also be generated using the latest corpus. To
enable this, add the following to `.clusterfuzzlite/cflite_cron.yml`:

{% raw %}
```yaml
name: ClusterFuzzLite regular tasks
on:
  schedule:
    - cron: '0 0 * * *'  # Once a day at midnight.
jobs:
  Coverage:
    runs-on: ubuntu-latest
    steps:
    - name: Build Fuzzers
      id: build
      uses: google/clusterfuzzlite/actions/build_fuzzers@v1
      with:
        sanitizer: coverage
    - name: Run Fuzzers
      id: run
      uses: google/clusterfuzzlite/actions/run_fuzzers@v1
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        fuzz-seconds: 600
        mode: 'coverage'
        storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
        storage-repo-branch: main   # Optional. Defaults to "main"
        storage-repo-branch-coverage: gh-pages  # Optional. Defaults to "gh-pages".
```
{% endraw %}

If `storage-repo` is set and `storage-repo-branch-coverage` is "gh-pages" (the
default), then coverage reports can be viewed at
`https://USERNAME.github.io/STORAGE-REPO-NAME/coverage/latest/report/linux/report.html`.

## Corpus pruning

Corpus pruning minimizes a corpus by removing redundant items while keeping the
same code coverage. To enable this, add the following to `.clusterfuzzlite/cflite_cron.yml`:

{% raw %}
```yaml
jobs:
  Pruning:
    runs-on: ubuntu-latest
    steps:
    - name: Build Fuzzers
      id: build
      uses: google/clusterfuzzlite/actions/build_fuzzers@v1
    - name: Run Fuzzers
      id: run
      uses: google/clusterfuzzlite/actions/run_fuzzers@v1
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        fuzz-seconds: 600
        mode: 'prune'
        storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
        storage-repo-branch: main   # Optional. Defaults to "main"
        storage-repo-branch-coverage: gh-pages  # Optional. Defaults to "gh-pages".
```
{% endraw %}

## Private repos

In order for ClusterFuzzLite to clone private repos, the GitHub token needs to be passed to the build steps as well:

{% raw %}
```yaml
    steps:
    - name: Build Fuzzers
      id: build
      uses: google/clusterfuzzlite/actions/build_fuzzers@v1
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
```
{% endraw %}

TODO: Configure private bucket to view coverage report.
