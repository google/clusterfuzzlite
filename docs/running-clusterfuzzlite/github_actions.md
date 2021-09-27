---
layout: default
parent: Running ClusterFuzzLite
grand_parent: ClusterFuzzLite
title: GitHub Actions
nav_order: 1
permalink: /running-clusterfuzzlite/github-actions/
---
# GitHub Actions
{: .no_toc}

- TOC
{:toc}
---


ClusterFuzzLite can be configured in a number of ways to enable both PR and
batch fuzzing.

We recommend having two workflow files:

- `.github/workflows/cflite_continuous.yaml` (for building and batch fuzzing)
- `.github/workflows/cflite_pr.yaml` (for PR fuzzing)

TODO: Host a clean, complete example somewhere.

## Continuous builds (required)

Continuous builds are used whenever a crash is found during PR or batch fuzzing
to determine if this crash was newly introduced.

Add the following to `cflite_continuous.yaml`:

{% raw %}
```yaml
name: CIFuzz continuous
on:
  push:
    branches:
      - main
jobs:
  Build:
   runs-on: ubuntu-latest
   steps:
   - name: Build Fuzzers
     id: build
     uses: google/clusterfuzzlite/actions/build_fuzzers@v1
     with:
       upload-build: true
```
{% endraw %}

This causes a build to be triggered and uploaded as a GitHub Actions artifact
whenever a new push is done to main. 

## PR fuzzing

To add a fuzzing workflow that runs on all pull requests to your project, add
the following to `cflite_pr.yaml`:

{% raw %}
```yaml
name: CIFuzz PR fuzzing
on:
  pull_request:
    paths:
      - '**'
jobs:
  PR:
    runs-on: ubuntu-latest
    steps:
    - name: Build Fuzzers
      id: build
      uses: google/clusterfuzzlite/actions/build_fuzzers@v1
    - name: Run Fuzzers
      id: run
      uses: google/clusterfuzzlite/actions/run_fuzzers@v1
      with:
        fuzz-seconds: 600
        github-token: ${{ secrets.GITHUB_TOKEN }}
        run-fuzzers-mode: 'ci'
```
{% endraw %}

## Batch fuzzing

To enable batch fuzzing, add the following to
`cflite_continuous.yaml`:

{% raw %}
```yaml
jobs:
  Batch:
    runs-on: ubuntu-latest
    steps:
    - name: Build Fuzzers
      id: build
      uses: google/clusterfuzzlite/actions/build_fuzzers@v1
    - name: Run Fuzzers
      id: run
      uses: google/clusterfuzzlite/actions/run_fuzzers@v1
      with:
        fuzz-seconds: 600
        github-token: ${{ secrets.GITHUB_TOKEN }}
        run-fuzzers-mode: 'batch'
```
{% endraw %}

### Git repo for storage

It's optional but recommended that you set up a separate git repo for storing
corpora and coverage reports. 

This can be added to the "Run fuzzers" step of all your jobs:

{% raw %}
```yaml
    - name: Run Fuzzers
      id: run
      uses: google/clusterfuzzlite/actions/run_fuzzers@v1
      with:
        fuzz-seconds: 600
        github-token: ${{ secrets.GITHUB_TOKEN }}
        run-fuzzers-mode: 'batch'
        # Git storage repo options.
        storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
        storage-repo-branch: main   # Optional. Defaults to "main"
        storage-repo-branch-coverage: gh-pages  # Optional. Defaults to "gh-pages".
```
{% endraw %}

You'll need to set up a [personal access token] with write permissions to the
storage repo and add it as an [environment secret] called `PERSONAL_ACCESS_TOKEN`. This
is because the default GitHub auth token is not able to write to other
repositories.

If this isn't specified, corpora and coverage reports will be uploaded as
GitHub artifacts instead.

[personal access token]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token
[environment secret]: https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-an-environment

### Coverage reports

Periodic coverage reports can also be generated using the latest corpus. To
enable this, add the following to `cflite_continuous.yaml`:

{% raw %}
```yaml
jobs:
  Coverage:
    runs-on: ubuntu-latest
    steps:
    - name: Build Fuzzers
      id: build
      uses: google/clusterfuzzlite/actions/build_fuzzers@v1
    - name: Run Fuzzers
      id: run
      uses: google/clusterfuzzlite/actions/run_fuzzers@v1
      with:
        sanitizer: coverage
        fuzz-seconds: 600
        github-token: ${{ secrets.GITHUB_TOKEN }}
        run-fuzzers-mode: 'coverage'
        storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
```
{% endraw %}

If `storage-repo` is set and `storage-repo-branch-coverage` is "gh-pages" (the
default), then coverage reports can be viewed at
`https://USERNAME.github.io/STORAGE-REPO-NAME/coverage/latest/report/linux/report.html`.

### Corpus pruning

Corpus pruning minimizes a corpus by removing redundant items while keeping the
same code coverage. To enable this, add the following to `cflite_continuous.yaml`:

{% raw %}
```
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
        fuzz-seconds: 600
        github-token: ${{ secrets.GITHUB_TOKEN }}
        run-fuzzers-mode: 'prune'
        storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
```
{% endraw %}
