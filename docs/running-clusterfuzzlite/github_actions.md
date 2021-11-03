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

This page explains how to set up ClusterFuzzLite to run on GitHub Actions.
To get the most of this page, you should have already set up your
[build integration] and read the more
[high-level document on running ClusterFuzzLite].

For basic ClusterFuzzLite functionality, all you need is a single workflow file
to enable fuzzing on your pull requests.

- `.github/workflows/cflite_pr.yml` (for PR fuzzing)

To enable more features, we recommend having these additional files:

- `.github/workflows/cflite_build.yml` (for continuous builds)
- `.github/workflows/cflite_batch.yml` (for batch fuzzing)
- `.github/workflows/cflite_cron.yml` (for tasks done on a cron schedule)

These workflow files are used by GitHub actions to run the ClusterFuzzLite
actions.
GitHub Actions documentation can be found [here](https://docs.github.com/en/actions).
For a complete example on a real project, see
<https://github.com/oliverchang/curl>.

## PR fuzzing

To add a fuzzing workflow that fuzzes all pull requests to your repo, add the
following to `.github/workflows/cflite_pr.yml`:

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
        # Optional but recommended: used to only run fuzzers that are affected
        # by the PR.
        # See later section on "Git repo for storage".
        # storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
        # storage-repo-branch: main   # Optional. Defaults to "main"
        # storage-repo-branch-coverage: gh-pages  # Optional. Defaults to "gh-pages".
    - name: Run Fuzzers (${{ matrix.sanitizer }})
      id: run
      uses: google/clusterfuzzlite/actions/run_fuzzers@v1
      with:
        github-token: ${{ secrets.GITHUB_TOKEN }}
        fuzz-seconds: 600
        mode: 'code-change'
        sanitizer: ${{ matrix.sanitizer }}
        # Optional but recommended: used to download the corpus produced by
        # batch fuzzing.
        # See later section on "Git repo for storage".
        # storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
        # storage-repo-branch: main   # Optional. Defaults to "main"
        # storage-repo-branch-coverage: gh-pages  # Optional. Defaults to "gh-pages".
```
{% endraw %}

You can copy and paste the above file and PR fuzzing should just work.
However, you can edit the file to:
- Enable more sanitizers (`sanitizers`).
- Change the amount of time spent fuzzing (`fuzz-seconds`).
- Enable a [storage repo] (`storage-repo`, `storage-repo-branch`,
  `storage-repo-branch-coverage`), which is an optional but useful feature
  discussed [later on], feel free to skip this at first.

Merge this file into your GitHub repo and open a pull request to test it out.
Now that we have discussed setting up code change fuzzing on pull requests, let's
look at setting up other tasks to get the most out of ClusterFuzzLite.

[later on]: #storage-repo

## Continuous builds and Batch fuzzing

Running [continuous builds] and [batch fuzzing] require a few more workflow
files.

[Continuous builds] allow code change fuzzing to detect if crashes were
introduced by the change under test.
If the crash is not novel, code change/PR fuzzing will not report it.

Batch fuzzing enables continuous, regular fuzzing on your latest HEAD, and
allows a corpus of inputs to build up over time that greatly improves the
effectiveness of fuzzing.

### Continuous builds

As mentioned above, continuous builds are used whenever a crash is found during
PR fuzzing to determine if this crash was newly introduced.
This means that there will be less unrelated failures when running code change
fuzzing.
To set up continuous builds, add the following to `.github/workflows/cflite_build.yml`:

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
You can copy and paste the above file, but make sure to change the `branches`
field to whatever is your project's main branch.
If you are going to use multiple sanitizers to fuzz your project, you should
change the `sanitizer` field to enable builds for each sanitizer you are fuzzing
with.

### Batch fuzzing

To enable batch fuzzing, add the following to
`.github/workflows/cflite_batch.yml`:

This can be run on either pushes to your default branch, or on a cron schedule (or both).

{% raw %}
```yaml
name: ClusterFuzzLite batch fuzzing
on:
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
        # See later section on "Git repo for storage".
        # storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
        # storage-repo-branch: main   # Optional. Defaults to "main"
        # storage-repo-branch-coverage: gh-pages  # Optional. Defaults to "gh-pages".
```
{% endraw %}

You can copy and paste the above file and batch fuzzing should just work.
However, you can edit the file to:
- Change how frequently batch fuzzing is run (`cron`). See [GitHub's documentation] on this.
- Enable more sanitizers (`sanitizers`).
- Change the amount of time spent fuzzing (`fuzz-seconds`).
- Enable a [storage repo] (`storage-repo`, `storage-repo-branch`,
  `storage-repo-branch-coverage`).

Another edit you can make is to configure batch fuzzing to run on each push to
your main branch by changing the `on` field to:

```yaml
on:
  push:
    branches:
      - main  # Use your actual default branch here.
```
As usual, make sure to change `branches` to the actual branch(es) you wish to
fuzz.

If batch fuzzing is running, you must also run corpus pruning, which we will discuss next.

[GitHub's documentation]: https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows#schedule

### Corpus pruning

Corpus pruning minimizes the corpus produced by batch fuzzing by removing
redundant items while keeping the same code coverage. To enable this, add the
following to `.github/workflows/cflite_cron.yml`:

{% raw %}
```yaml
name: ClusterFuzzLite cron tasks
on:
  schedule:
    - cron: '0 0 * * *'  # Once a day at midnight.
permissions: read-all
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
        # Optional but recommended.
        # See later section on "Git repo for storage".
        # storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
        # storage-repo-branch: main   # Optional. Defaults to "main"
        # storage-repo-branch-coverage: gh-pages  # Optional. Defaults to "gh-pages".
```
{% endraw %}

You can copy and paste the above file and pruning should just work.
However, you can edit the file to:

- Enable a [storage repo] (`storage-repo`, `storage-repo-branch`,
  `storage-repo-branch-coverage`).

Finally, let's discuss the last task that ClusterFuzzLite can run: coverage
report generation.

## Coverage reports

To generate periodic coverage reports, add the following to the `jobs` field in
`.github/workflows/cflite_cron.yml` after the `Pruning` job:

{% raw %}
```yaml
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
        sanitizer: 'coverage'
        # Optional but recommended.
        # See later section on "Git repo for storage".
        # storage-repo: https://${{ secrets.PERSONAL_ACCESS_TOKEN }}@github.com/OWNER/STORAGE-REPO-NAME.git
        # storage-repo-branch: main   # Optional. Defaults to "main"
        # storage-repo-branch-coverage: gh-pages  # Optional. Defaults to "gh-pages".
```
{% endraw %}

You can copy and paste the above file and pruning should just work.
But here is where `storage-repo`, `storage-repo-branch`, and
`storage-repo-branch-coverage` come in handy.

If `storage-repo` is set and `storage-repo-branch-coverage` is "gh-pages" (the
default), then coverage reports can be viewed at
`https://USERNAME.github.io/STORAGE-REPO-NAME/coverage/latest/report/linux/report.html`.
This should be much nicer than downloading the coverage report as an artifact.

## Extra configuration

### Git repo for storage {#storage-repo}

As previously discussed, it's optional but recommended that you set up a
separate git repo for storing corpora and coverage reports.
It is recommended because the storage repo will make corpus management better in
some scenarios and will allow coverage reports to be viewed from the web.

An empty repository for this is sufficient.

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
GitHub artifacts instead. They should still work regardless.

[personal access token]: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token
[repository secret]: https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-an-environment
[issues with the storage repo]: https://github.com/google/oss-fuzz/issues/6668

### Private repos

In order for ClusterFuzzLite to clone private repos, the GitHub token needs to
be passed to the build steps as well.
This token is automaticallly set by GitHub actions so no extra action is
required from you except passing it to build fuzzers in each workflow file
like so:

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

[build integration]: {{ site.baseurl }}/build-integration/
[high-level document on running ClusterFuzzLite]: {{ site.baseurl }}/running-clusterfuzzlite/
[storage repo]: #storage
[continuous builds]: #continuous-builds
[batch fuzzing]: #batch-fuzzing
