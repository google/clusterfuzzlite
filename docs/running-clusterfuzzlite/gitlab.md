---
layout: default
parent: Step 2&#58; Running ClusterFuzzLite
grand_parent: ClusterFuzzLite
title: GitLab
nav_order: 2
permalink: /running-clusterfuzzlite/gitlab/
---
# GitLab
{: .no_toc}

- TOC
{:toc}
---

This page explains how to set up ClusterFuzzLite to run on [GitLab].
To get the most of this page, you should have already set up your
[build integration] and read the more
[high-level document on running ClusterFuzzLite].

## Gitlab runner
The following examples use a `docker` gitlab runner running sibling containers:
[doc](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-socket-binding).

It should be possible to achieve the same functionalities with using a shell executor.
But then, the `.gitlab-ci.yml` should be different, and explicitly call the `docker` commands
on ClusterFuzzLite images.

## .gitlab-ci.yml
For basic ClusterFuzzLite functionality, all you need is a single job
to enable fuzzing on your merge requests.

To enable more features, we recommend having different jobs for:

- continuous builds
- batch fuzzing and corpus pruning
- coverage

### MR fuzzing

To add a fuzzing job that fuzzes all merge requests to your repo, add the
following default configurations to `.gitlab-ci.yml`:

{% raw %}
```yaml
variables:
  SANITIZER: address
  CFL_PLATFORM: gitlab

clusterfuzzlite:
  image:
    name: google/clusterfuzzlite/actions/build_fuzzers@v1
    entrypoint: [""]
  stage: fuzz
  parallel:
    matrix:
      - SANITIZER: [address, undefined]
  rules:
    # Default code change.
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      variables:
        MODE: "code-change"
  before_script:
    # Get gitlab's container id.
    - export CFL_CONTAINER_ID=`cut -c9- < /proc/1/cpuset`
  script:
    # Will build and run the fuzzers.
    - python3 "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
  artifacts:
    # Upload artifacts when a crash makes the job fail.
    when: always
    paths:
      - artifacts/
```
{% endraw %}

You may also wish to set [tags](https://docs.gitlab.com/runner/#tags) to select a relevant runner.

Optionally, edit the following variables to customize your settings:
- `SANITIZER` Select sanitizer(s)
- `LANGUAGE` Define the language of your project.
- `CFL_BRANCH` Branch to fuzz, default is `CI_DEFAULT_BRANCH`.
- `FILESTORE` storage for files: builds, corpus, coverage and crashes.
- `FUZZ_SECONDS` Change the amount of time spent fuzzing.
- `CFL_ARTIFACTS_DIR` To save your artifacts in a different directory than `artifacts`

### Batch fuzzing and corpus pruning

Batch fuzzing enables continuous, regular fuzzing on your latest HEAD and
allows a corpus of inputs to build up over time, which greatly improves the
effectiveness of fuzzing. Batch fuzzing should be run on a schedule.

To enable batch fuzzing, add the following to
`.gitlab-ci.yml`:

{% raw %}
```yaml
clusterfuzzlite-corpus:
  image:
    name: google/clusterfuzzlite/actions/build_fuzzers@v1
    entrypoint: [""]
  stage: fuzz
  rules:
    - if: $MODE == "prune"
    - if: $MODE == "batch"
  before_script:
    - export CFL_CONTAINER_ID=`cut -c9- < /proc/1/cpuset`
  script:
    - python3 "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
  artifacts:
    when: always
    paths:
      - artifacts/
```
{% endraw %}

You should then define two [schedules](https://docs.gitlab.com/ee/ci/pipelines/schedules.html)
In one, you should set the variable `MODE` to `batch` to run the actual batch fuzzing.
In the other, you should set the variable `MODE` to `prune` for corpus pruning once a day.
These schedules should target the main/default/`CFL_BRANCH` branch.

### Continuous builds

The continuous build task causes a build to be triggered and uploaded
whenever a new push is done to main/default branches.

Continuous builds are used when a crash is found during PR fuzzing to determine whether the crash was newly introduced.
If the crash was not newly introduced, PR fuzzing will not report it.
This means that there will be fewer unrelated failures when running code change
fuzzing.

To set up continuous builds, add the following to `.gitlab-ci.yml`:

{% raw %}
```yaml
clusterfuzzlite-build:
  image:
    name: google/clusterfuzzlite/actions/build_fuzzers@v1
    entrypoint: [""]
  stage: fuzz
  rules:
    # Use $CI_DEFAULT_BRANCH or $CFL_BRANCH.
    - if: $CI_COMMIT_BRANCH == $CFL_BRANCH && $CI_PIPELINE_SOURCE == "push"
      variables:
        MODE: "code-change"
        UPLOAD_BUILD: "true"
  before_script:
    - export CFL_CONTAINER_ID=`cut -c9- < /proc/1/cpuset`
  script:
    - python3 "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
  artifacts:
    when: always
    paths:
      - artifacts/
```
{% endraw %}

### Coverage reports

To generate periodic coverage reports, add the following job to
`.gitlab-ci.yml`:

{% raw %}
```yaml
clusterfuzzlite-coverage:
  image:
    name: google/clusterfuzzlite/actions/build_fuzzers@v1
    entrypoint: [""]
  stage: fuzz
  variables:
    SANITIZER: "coverage"
  rules:
    - if: $MODE == "coverage"
  before_script:
    - export CFL_CONTAINER_ID=`cut -c9- < /proc/1/cpuset`
  script:
    - python3 "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
  artifacts:
    when: always
    paths:
      - artifacts/
```
{% endraw %}

You should then define one [schedule](https://docs.gitlab.com/ee/ci/pipelines/schedules.html)
In it, you should set the variable `MODE` to `coverage`.
This schedule should target the main/default/`CFL_BRANCH` branch.

## Extra configuration

### Gitlab filestore

You can use the variable `FILESTORE: gitlab` to use Gitlab artifacts for storing
- coverage reports
- corpus
- continuous build
- crashes

Crashes get simply added as jobs artifacts.

For continuous builds, you need to use a [cache](https://docs.gitlab.com/ee/ci/caching/) in your jobs:
{% raw %}
```yaml
  variables:
    CFL_CACHE_DIR: cfl-cache
  cache:
    key: clusterfuzzlite
    paths:
      - cfl-cache/
```
{% endraw %}
The cache directory needs to defined as `CFL_CACHE_DIR` to be used by ClusterFuzzLite.
If it is not defined, the default value is `cache`.
You should ensure that the runners share the access to the cache.

For coverage reports and corpus, it is recommended to set up another git repository.
You need to create a project access token for this repository, with `read_repository` and `write_repository` rights.
And this token should be used from the fuzzed repository as a CI/CD variable.
You can name this variable as you like, in the following example it is named `CFL_TOKEN`.
This variable should be defined as masked to avoid leaks.
Last, you need to setup these variables in `.gitlab-ci.yml` :
```
  GIT_STORE_REPO: "https://oauth2:${CFL_TOKEN}@gitlab.hostname/namespace/project-cfl.git"
  GIT_STORE_BRANCH: main
  GIT_STORE_BRANCH_COVERAGE: coverage
```

If you do not set another git repository, the GitLab filestore will fall back to the cache.

For coverage reports, you may want to set up GitLab [pages](https://docs.gitlab.com/ee/user/project/pages/)
In the repository hosting your coverage, you should add a `pages` job such as:
{% raw %}
```yaml
pages:
  script:
    - mkdir .public
    - cp -r * .public
    - mv .public public
  artifacts:
    paths:
      - public 
```
{% endraw %}
This job will build a static web site with everything which is in the `public` directory.
You may then access the site at `https://baseurl/coverage/latest/report/linux/report.html` where
`baseurl` is the domain you configured for your GitLab pages.
