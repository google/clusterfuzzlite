---
layout: default
parent: Step 2&#58; Running ClusterFuzzLite
grand_parent: ClusterFuzzLite
title: GitLab
nav_order: 1
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
on clusterfuzz-lite images.

Docker-in-Docker does not seem possible as clusterfuzz-lite images
do not have the docker tools installed.

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
  SANITIZER: "address"
  WORKSPACE: $CI_PROJECT_DIR
  REPOSITORY: $CI_PROJECT_NAME
  CFL_PLATFORM: gitlab

clusterfuzzlite:
  image:
    name: google/clusterfuzzlite/actions/build_fuzzers@v1
    entrypoint: [""]
  stage: fuzz
  rules:
    # default code change
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      variables:
        MODE: "code-change"
  # to select a relevant runner
  tags:
    - fuzz
  before_script:
    # gitlab's container name is not its hostname !
    - export CFL_CONTAINER_ID=`cut -c9- < /proc/1/cpuset`
  script:
    # will build and run the fuzzers
    - python3 "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
  artifacts:
    paths:
      - artifacts/crashes/
```
{% endraw %}

Optionally, edit the following variables to customize your settings:
- `SANITIZER` Change or enable more sanitizers.
- `LANGUAGE` Define the language of your project.
- `CFL_BRANCH` Branch to fuzz, default is `CI_DEFAULT_BRANCH`.
- `FILESTORE` To use corpus produced by other jobs.
- `FUZZ_SECONDS` Change the amount of time spent fuzzing.
- `CFL_ARTIFACTS_DIR` : To save your artifacts in a different directory than `artifacts`


### Batch fuzzing and corpus pruning

Batch fuzzing enables continuous, regular fuzzing on your latest HEAD and
allows a corpus of inputs to build up over time, which greatly improves the
effectiveness of fuzzing. Batch fuzzing can be run on a schedule.

To enable batch fuzzing, add the following to
`.gitlab-ci.yml`:

{% raw %}
```yaml
# this name is hardcoded and cannot be changed for gitlab artifacts filestore
clusterfuzzlite-corpus:
  image:
    name: google/clusterfuzzlite/actions/build_fuzzers@v1
    entrypoint: [""]
  stage: fuzz
  rules:
    - if: $MODE == "prune"
    - if: $MODE == "batch"
  tags:
    - fuzz
  before_script:
    # gitlab's container name is not its hostname !
    - export CFL_CONTAINER_ID=`cut -c9- < /proc/1/cpuset`
  script:
    - python3 "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
  artifacts:
    paths:
      - artifacts/corpus/
      - artifacts/crashes/
```
{% endraw %}

You should then define two [schedules](https://docs.gitlab.com/ee/ci/pipelines/schedules.html)
In one, you should set the variable `MODE` to `batch` to run the actual batch fuzzing.
In the other, you should set the variable `MODE` to `prune` for corpus pruning once a day.
These schedules should target the main/default/`CFL_BRANCH` branch.

### Continuous builds

The continuous build task causes a build to be triggered and uploaded
whenever a new push is done to main/default branch.

Continuous builds are used when a crash is found during PR fuzzing to determine whether the crash was newly introduced.
If the crash is not novel, PR fuzzing will not report it.
This means that there will be fewer unrelated failures when running code change
fuzzing.

To set up continuous builds, add the following to `.gitlab-ci.yml`:

{% raw %}
```yaml
# this name is hardcoded and cannot be changed for gitlab artifacts filestore
clusterfuzzlite-build:
  image:
    name: google/clusterfuzzlite/actions/build_fuzzers@v1
    entrypoint: [""]
  stage: fuzz
  rules:
    # use $CI_DEFAULT_BRANCH or $CFL_BRANCH
    - if: $CI_COMMIT_BRANCH == $CFL_BRANCH && $CI_PIPELINE_SOURCE == "push"
      variables:
        MODE: "code-change"
        UPLOAD_BUILD: "true"
  tags:
    - fuzz
  before_script:
    # gitlab's container name is not its hostname !
    - export CFL_CONTAINER_ID=`cut -c9- < /proc/1/cpuset`
  script:
    - python3 "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
  artifacts:
    paths:
      - artifacts/build/
      - artifacts/crashes/
```
{% endraw %}

### Coverage reports

To generate periodic coverage reports, add the following job to
`.gitlab-ci.yml`:

{% raw %}
```yaml
# this name is hardcoded and cannot be changed for gitlab artifacts filestore
clusterfuzzlite-coverage:
  image:
    name: google/clusterfuzzlite/actions/build_fuzzers@v1
    entrypoint: [""]
  stage: fuzz
  variables:
    SANITIZER: "coverage"
  rules:
    - if: $MODE == "coverage"
  tags:
    - fuzz
  before_script:
    # gitlab's container name is not its hostname !
    - export CFL_CONTAINER_ID=`cut -c9- < /proc/1/cpuset`
  script:
    - python3 "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
  artifacts:
    paths:
      - artifacts/coverage/
      - artifacts/crashes/
```
{% endraw %}

You should then define one [schedule](https://docs.gitlab.com/ee/ci/pipelines/schedules.html)
In it, you should set the variable `MODE` to `coverage`.
This schedule should target the main/default/`CFL_BRANCH` branch.


## Extra configuration

### Gitlab artifacts filestore

You can use the variable `FILESTORE: gitlab` to use Gitlab artifacts for storing
- coverage reports
- corpus
- continuous build

To do so, you need to define an [access token](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)
with the scope `read_api`.

Then, you need to use its value in a variable which you can define in your CI/CD settings.
You should define it as masked to avoid leaks.
The variable's name should be `CFL_PRIVATE_TOKEN` and its value should be the token.
