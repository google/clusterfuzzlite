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

For basic ClusterFuzzLite functionality, all you need is a single workflow file
to enable fuzzing on your pull requests.

- `.github/workflows/cflite_pr.yml` (for PR fuzzing)

To enable more features, we recommend having these additional files:

- `.github/workflows/cflite_build.yml` (for building/continuous fuzzing)
- `.github/workflows/cflite_batch.yml` (for batch fuzzing)
- `.github/workflows/cflite_cron.yml` (for tasks done on a cron schedule)

For a complete example on a real project, see
<https://github.com/oliverchang/curl>. GitHub Actions documentation can be
found [here](https://docs.github.com/en/actions).

## PR fuzzing

To add a fuzzing workflow that runs on all pull requests to your project, add
the following to `.github/workflows/cflite_pr.yml`:

{% raw %}
```yaml
name: ClusterFuzzLite PR fuzzing
on:
  pull_request:
    paths:
      - '**'
permissions: read-all
jobs:
  PR:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        sanitizer:
        - address
        # Override this with the sanitizers you want.
        # - undefined
        # - memory
    steps:
    - name: Build Fuzzers (${{ matrix.sanitizer }})
      id: build
      uses: google/clusterfuzzlite/actions/build_fuzzers@v1
      with:
        sanitizer: ${{ matrix.sanitizer }}
        # Optional but recommended: used to only run fuzzers that are affected by
        # the PR. See later section on "Git repo for storage"
        # storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
        # storage-repo-branch-coverage: gh-pages  # Optional. Defaults to "gh-pages".
    - name: Run Fuzzers (${{ matrix.sanitizer }})
      id: run
      uses: google/clusterfuzzlite/actions/run_fuzzers@v1
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        fuzz-seconds: 600
        mode: 'code-change'
        sanitizer: ${{ matrix.sanitizer }}
```
{% endraw %}

## Batch fuzzing and continuous builds

Running batch fuzzing and continuous builds requires a few more workflow files
to be set up.

Batch fuzzing enables continuous, regular fuzzing on your latest HEAD, and
allows a corpus of inputs to build up over time that greatly improves the
effectiveness of fuzzing.

### Continuous builds

Continuous builds are used whenever a crash is found during PR fuzzing to
determine if this crash was newly introduced.

Add the following to `.github/workflows/cflite_build.yml`:

{% raw %}
```yaml
name: ClusterFuzzLite continuous builds
on:
  push:
    branches:
      - main  # Use your actual default branch here.
permissions: read-all
jobs:
  Build:
   runs-on: ubuntu-latest
   strategy:
     fail-fast: false
     matrix:
        sanitizer:
        - address
        # Override this with the sanitizers you want.
        # - undefined
        # - memory
   steps:
   - name: Build Fuzzers (${{ matrix.sanitizer }})
     id: build
     uses: google/clusterfuzzlite/actions/build_fuzzers@v1
     with:
       sanitizer: ${{ matrix.sanitizer }}
       upload-build: true
```
{% endraw %}

This causes a build to be triggered and uploaded as a GitHub Actions artifact
whenever a new push is done to main/default branch.

### Batch fuzzing

To enable batch fuzzing, add the following to
`.github/workflows/cflite_batch.yml`:

This can be run on either pushes to your default branch, or on a cron schedule (or both).

{% raw %}
```yaml
name: ClusterFuzzLite batch fuzzing
on:
  push:
    branches:
      - main  # Use your actual default branch here.
  schedule:
    - cron: '0 0/6 * * *'  # Every 6th hour. Change this to whatever is suitable.
permissions: read-all
jobs:
  BatchFuzzing:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        sanitizer:
        - address
        # Override this with the sanitizers you want.
        # - undefined
        # - memory
    steps:
    - name: Build Fuzzers (${{ matrix.sanitizer }})
      id: build
      uses: google/clusterfuzzlite/actions/build_fuzzers@v1
      with:
        sanitizer: ${{ matrix.sanitizer }}
    - name: Run Fuzzers (${{ matrix.sanitizer }})
      id: run
      uses: google/clusterfuzzlite/actions/run_fuzzers@v1
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        fuzz-seconds: 3600
        mode: 'batch'
        sanitizer: ${{ matrix.sanitizer }}
        # Optional but recommended: For storing certain artifacts from fuzzing.
        # See later section on "Git repo for storage"
        # storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
        # storage-repo-branch: main   # Optional. Defaults to "main"
        # storage-repo-branch-coverage: gh-pages  # Optional. Defaults to "gh-pages".
```
{% endraw %}

#### Git repo for storage

It's optional but recommended that you set up a separate git repo for storing
corpora and coverage reports.

This can be added to the "Run fuzzers" step of all your jobs:

{% raw %}
```yaml
jobs:
  BatchFuzzing:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        sanitizer:
        - address
        # Override this with the sanitizers you want.
        # - undefined
        # - memory
    steps:
    - name: Build Fuzzers (${{ matrix.sanitizer }})
      id: build
      uses: google/clusterfuzzlite/actions/build_fuzzers@v1
      with:
        sanitizer: ${{ matrix.sanitizer }}
    - name: Run Fuzzers (${{ matrix.sanitizer }})
      id: run
      uses: google/clusterfuzzlite/actions/run_fuzzers@v1
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        fuzz-seconds: 600
        mode: 'batch'
        sanitizer: ${{ matrix.sanitizer }}
        # Git storage repo options.
        storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
        storage-repo-branch: main   # Optional. Defaults to "main"
        storage-repo-branch-coverage: gh-pages  # Optional. Defaults to "gh-pages".
```
{% endraw %}

You'll need to set up a [personal access token] with write permissions to the
storage repo and add it as a [repository secret] called
`PERSONAL_ACCESS_TOKEN`. This is because the default GitHub auth token is not
able to write to other repositories.
Note that if you run into [issues with the storage repo], you may have
accidentally set the secret as an *environment secret* instead of a
*repository secret*.

If you would like PR fuzzing to only run fuzzers affected by the current
change, you'll need to add these same options to the ["Build Fuzzers" step
above](#pr-fuzzing). The way "affected fuzzers" are determined is by using
coverage reports.

If this isn't specified, corpora and coverage reports will be uploaded as
GitHub artifacts instead.

[personal access token]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token
[repository secret]: https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-an-environment
[issues with the storage repo]: https://github.com/google/oss-fuzz/issues/6668

### Coverage reports

Periodic coverage reports can also be generated using the latest corpus. To
enable this, add the following to `.github/workflows/cflite_cron.yml`:

{% raw %}
```yaml
name: ClusterFuzzLite cron tasks
on:
  schedule:
    - cron: '0 0 * * *'  # Once a day at midnight.
permissions: read-all
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

### Corpus pruning

Corpus pruning minimizes a corpus by removing redundant items while keeping the
same code coverage. To enable this, add the following to `.github/workflows/cflite_cron.yml`:

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
