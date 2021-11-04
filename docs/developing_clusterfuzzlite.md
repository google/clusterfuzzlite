---
layout: default
parent: ClusterFuzzLite
title: Developing ClusterFuzzLite
has_children: true
nav_order: 5
permalink: /developing-clusterfuzzlite/
---
# Developing ClusterFuzzLite
{: .no_toc}

- TOC
{:toc}
---

## Code location

The code for ClusterFuzzLite is located in the [`infra/cifuzz` directory] of the
[OSS-Fuzz repo]. This is because ClusterFuzzLite grew out of [CIFuzz].
All of the commands discussed in the development docs are assumed to be running
in a checkout of the OSS-Fuzz repo, which you can get with the following
command:

```bash
git clone git@github.com:google/oss-fuzz.git
```

## Prerequisites

ClusterFuzzLite uses Python 3.8 or higher and [docker].
It is unknown if development works on operating systems other than Linux.

## Getting started

Create a virtual env and install the requirements:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r infra/cifuzz/requirements.txt
```

The development docs assume all commands are run within this virtual
environment.

## Testing

The right way to test your changes will depend on what kind of feature you are
adding. At a minimum, you should format and lint your code as well as run the
unnittests.
Thes can be done with the following commands:

```bash
python infra/presubmit.py format
python infra/presubmit.py lint
# Run tests in parallel with -p option. Use -s to skip irrelevant tests.
END_TO_END_TESTS=1 INTEGRATION_TESTS=1 python infra/presubmit.py infra-tests -p -s
```

You can specify specific tests to run using [pytest's -k option].
Here's an example:

```bash
END_TO_END_TESTS=1 CIFUZZ_TEST=1 INTEGRATION_TESTS=1 pytest -s -vv -k GetGitUrlTest
```

Note that you must specify `CIFUZZ_TEST=1`.

### End-to-end testing using GitHub actions

One of the best ways to test changes end-to-end is to create your own GitHub
repo that uses ClusterFuzzLite github actions.
TOOD: Change to ClusterFuzzLite and clean up repo.
You can use [cifuzz-external-example] for this, just create a new repo with the
same code.
Then do the following to test your changes:
1. Push the changes in the OSS-Fuzz repository to github repository (your own
   fork is fine).
1. Change the lines in cifuzz.yml that reference `google/oss-fuzz...@master` to
   point to your repo and the branch you want to test. E.g.
   ```yaml
   uses: <your-github-username>/oss-fuzz/infra/cifuzz/external-actions/build_fuzzers@<branch-to-test>
   ```
   Make sure to change all of the references in the repo (e.g. run_fuzzers as
   well).
1. Push a change to your test repo to run the actions from your OSS-Fuzz repo.

Note that the workflow in cifuzz-external-example uses actions from
`google/oss-fuzz/infra/cifuzz/external-actions` instead of
`google/oss-fuzz/infra/cifuzz/actions/build_fuzzers@main`.
These actions must be used for testing because the rebuild the clusterfuzzlite
docker images from the repo and branch specified, unlike the actions in
clusterfuzzlite which will just use images from gcr.io/oss-fuzz-base`

### End-to-end testing locally or on other platforms

To test end-to-end locally you must rebuild the ClusterFuzzLite docker images to
include your source code changes. You can doing this using the following
command:

```bash
./infra/cifuzz/build-images.sh
```

If you are trying to test on a remote system (e.g. you are porting
ClusterFuzzLite to [Travis CI]) you need to make sure that the system is using
your docker images rather than the images published by ClusterFuzzLite for
end-users.
You can do this by doing the following:
1. Re-tagging the docker images after building them and the

```bash
docker tag \
    gcr.io/oss-fuzz-base/clusterfuzzlite-build-fuzzers:v1 \
    <your-docker-repo>/clusterfuzzlite-build-fuzzers:v1

docker tag \
    gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1 \
    <your-docker-repo>/clusterfuzzlite-run-fuzzers:v1
```
2. Pushing your newly tagged images:
```bash
docker push <your-docker-repo>/clusterfuzzlite-build-fuzzers:v1
docker push <your-docker-repo>/clusterfuzzlite-run-fuzzers:v1
```
3. Configuring your test repository to use
   `<your-docker-repo>/clusterfuzzlite-build-fuzzers:v1` and
   `<your-docker-repo>/clusterfuzzlite-run-fuzzers:v1` instead of
   `gcr.io/oss-fuzz-base/clusterfuzzlite-build-fuzzers:v1` and
   `gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers:v1`

[`infra/cifuzz` directory]: https://github.com/google/oss-fuzz/tree/master/infra/cifuzz
[OSS-Fuzz repo]: https://github.com/google/oss-fuzz
[CIFuzz]: https://google.github.io/oss-fuzz/getting-started/continuous-integration/
[pytest's -k option]: https://docs.pytest.org/en/6.2.x/example/markers.html#using-k-expr-to-select-tests-based-on-their-name
[cifuzz-external-example]: https://github.com/jonathanmetzman/cifuzz-external-example
[Travis CI]: https://travis-ci.org/
[docker]: https://docs.docker.com/get-docker/
