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

The following documentation is primarily meant for Gitlab.com with the use of shared runners.
It works also for self-managed GitLab instances.

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

For self-managed GitLab instances, this configuration requires at least GitLab 13.3 to be run.
With older versions, the `parallel` keywords does not exist, but you can define `SANITIZER` as a GitLab CI variable.

{% raw %}
```yaml
variables:
  SANITIZER: address
  CFL_PLATFORM: gitlab
  DOCKER_HOST: "tcp://docker:2375" # may be removed in self-managed GitLab instances
  DOCKER_IN_DOCKER: "true" # may be removed in self-managed GitLab instances


clusterfuzzlite:
  image:
    name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    entrypoint: [""]
  services:
    - docker:dind # may be removed in self-managed GitLab instances
    
  stage: test
  parallel:
    matrix:
      - SANITIZER: [address, undefined]
  rules:
    # Default code change.
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      variables:
        MODE: "code-change"
  before_script:
    # Get GitLab's container id.
    - export CFL_CONTAINER_ID=`docker ps -q -f "label=com.gitlab.gitlab-runner.job.id=$CI_JOB_ID" -f "label=com.gitlab.gitlab-runner.type=build"`
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

For self-managed GitLab instances, you may also wish to set [tags](https://docs.gitlab.com/runner/#tags)
to select a relevant runner.

Optionally, edit the following variables to customize your settings:
- `SANITIZER` Select sanitizer(s)
- `LANGUAGE` Define the language of your project.
- `CFL_BRANCH` Branch to fuzz, default is `CI_DEFAULT_BRANCH`.
- `FILESTORE` storage for files: builds, corpus, coverage and crashes.
- `FUZZ_SECONDS` Change the amount of time spent fuzzing.
- `PARALLEL_FUZZING` Use all available cores when fuzzing.
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
    name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    entrypoint: [""]
  services:
    - docker:dind # may be removed in self-managed GitLab instances
  stage: test
  rules:
    - if: $MODE == "prune"
    - if: $MODE == "batch"
  before_script:
    - export CFL_CONTAINER_ID=`docker ps -q -f "label=com.gitlab.gitlab-runner.job.id=$CI_JOB_ID" -f "label=com.gitlab.gitlab-runner.type=build"`
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

Note that you can also use gitlab-ci.yml [extends](https://docs.gitlab.com/ee/ci/yaml/#extends)
keyword to avoid duplicating most of the common parameters between the different type of jobs.

![gitlab-schedule-mode]

### Continuous builds

The continuous build task causes a build to be triggered and uploaded
whenever a new push is done to main/default branches.

Continuous builds are used when a crash is found during MR fuzzing to determine whether the crash was newly introduced.
If the crash was not newly introduced, MR fuzzing will not report it.
This means that there will be fewer unrelated failures when running code change
fuzzing.

To set up continuous builds, add the following to `.gitlab-ci.yml`:

{% raw %}
```yaml
clusterfuzzlite-build:
  image:
    name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    entrypoint: [""]
  services:
    - docker:dind # may be removed in self-managed GitLab instances
  stage: test
  rules:
    # Use $CI_DEFAULT_BRANCH or $CFL_BRANCH.
    - if: $CI_COMMIT_BRANCH == $CFL_BRANCH && $CI_PIPELINE_SOURCE == "push"
      variables:
        MODE: "code-change"
        UPLOAD_BUILD: "true"
  before_script:
    - export CFL_CONTAINER_ID=`docker ps -q -f "label=com.gitlab.gitlab-runner.job.id=$CI_JOB_ID" -f "label=com.gitlab.gitlab-runner.type=build"`
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
    name: gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1
    entrypoint: [""]
  services:
    - docker:dind # may be removed in self-managed GitLab instances
  stage: test
  variables:
    SANITIZER: "coverage"
  rules:
    - if: $MODE == "coverage"
  before_script:
    - export CFL_CONTAINER_ID=`docker ps -q -f "label=com.gitlab.gitlab-runner.job.id=$CI_JOB_ID" -f "label=com.gitlab.gitlab-runner.type=build"`
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

### GitLab runners on self-managed GitLab instances

The previous examples used [Docker in docker](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-in-docker)

From a performance point of view, it is recommended to use a `docker` gitlab runner running sibling containers:
See this [doc](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-socket-binding)
for more information.

To do so, if you have such a runner ready, you simply need to remove the following lines from the configuration:
{% raw %}
```yaml
variables:
  DOCKER_HOST: "tcp://docker:2375" # may be removed in self-managed GitLab instances
  DOCKER_IN_DOCKER: "true" # may be removed in self-managed GitLab instances

  services:
    - docker:dind # may be removed in self-managed GitLab instances
```

Note that it should be possible to achieve the same functionality using a shell
executor, though this is unsupported.
In that case, the `.gitlab-ci.yml` will different. For one you must explicitly
call the `docker` commands on ClusterFuzzLite images.

### Gitlab filestore

You can use the variable `FILESTORE: gitlab` to use GitLab artifacts for storing
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
The cache directory needs to be defined as `CFL_CACHE_DIR` to be used by ClusterFuzzLite.
If it is not defined, the default value is `cache`.
You should ensure that the runners share the access to the cache.

For coverage reports and corpus, it is recommended to set up another
git repository for storage.
You need to create a project access token for this repository, with
`read_repository` and `write_repository` rights.

You can also use a personal access token if you do not have access to
project access token, due to your GitLab license.

![gitlab-project-token]

Add the token as a CI/CD variable to your GitLab project.
You can name this variable as you like, in the following example it is named
`CFL_TOKEN`. This variable should be defined as masked to avoid leaks.
It should not be protected if you need it on unprotected branches.

![gitlab-variable-token]

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

[GitLab]: https://about.gitlab.com/
[build integration]: {{ site.baseurl }}/build-integration/
[high-level document on running ClusterFuzzLite]: {{ site.baseurl }}/running-clusterfuzzlite/
[gitlab-schedule-mode]: https://raw.githubusercontent.com/google/clusterfuzzlite/refs/heads/bucket/images/gitlab-schedule-mode.png
[gitlab-project-token]: https://raw.githubusercontent.com/google/clusterfuzzlite/refs/heads/bucket/images/gitlab-project-token.png
[gitlab-variable-token]: https://raw.githubusercontent.com/google/clusterfuzzlite/refs/heads/bucket/images/gitlab-variable-token.png
