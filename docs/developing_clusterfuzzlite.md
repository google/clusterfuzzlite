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
[OSS-Fuzz repo].
This is because ClusterFuzzLite grew out of [CIFuzz], which itself grew out of
OSS-Fuzz.
Note that because ClusterFuzzLite grew out of OSS-Fuzz, you might see references
to projects being "internal" or "external", internal refers to OSS-Fuzz projects
using CIFuzz, which are in some places handled differently than "external"
projects which use ClusterFuzzLite.

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

# Install ClusterFuzzLite dependencies.
pip install -r infra/cifuzz/requirements.txt

# Install development requirements.
pip install -r infra/ci/requirements.txt
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
`-p` will run tests in parallel and is recommended.
`-s` will skip some non-ClusterFuzzLite tests (tests for OSS-Fuzz's build infra).
You can specify specific tests to run using [pytest's -k option].
Here's an example:

```bash
END_TO_END_TESTS=1 CIFUZZ_TEST=1 INTEGRATION_TESTS=1 pytest -s -vv -k GetGitUrlTest
```

Note that you must specify `CIFUZZ_TEST=1`.

### End-to-end testing using GitHub actions

One of the best ways to test changes end-to-end is to create your own GitHub
repo that uses ClusterFuzzLite github actions.
<!-- TOOD: Change to ClusterFuzzLite and clean up repo. -->
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

## Architecture

Now let's look ClusterFuzzLite's architecture and explain some key details on
how it is implemented.

### Configuration

All of ClusterFuzzLite's configuration is handled through environment variables.
This is mostly handled by the `config_utils.py` module, but some configuration
that is specific to different platforms (i.e. CI systems) is handled in
`platform_config`, see the guide on [adding support for a new platform] for more
details on this.

### Building and Running fuzzers

Everything ClusterFuzzLite does falls under the two main functions it performs:
- Building fuzzers.
- Running fuzzers (not necessarily for fuzzing).

These two functions are performed by two different docker images:
- `gcr.io/oss-fuzz-base/clusterfuzzlite-build-fuzzers`
- `gcr.io/oss-fuzz-base/clusterfuzzlite-run-fuzzers`

#### Building fuzzers

Building fuzzers is the simpler of the two functions and is pretty much the same
whether ClusterFuzzLite is code change fuzzing, batch fuzzing, pruning,
generating coverage reports or saving continuous builds.
There are four steps that can go into "Building fuzzers" though not of them are
run every time fuzzers are built.
These steps are:
- [Building the builder image and the fuzzers].
- [Deleting unaffected fuzzers].
- [Checking fuzzers for common mistakes].
- [Uploading builds to the filestore].

##### Building the builder image and the fuzzers

The first step for ClusterFuzzLite is building the docker image defined by
the user project's `.clusterfuzzlite/Dockerfile`.
The next step is building the fuzzers.
The main configuration variable that change how building the fuzzers works are
`SANITIZER` and `LANGUAGE`.
However, ClusterFuzzLite does not really use these variables directly, it simply
passes them to the `compile` script that is run in the OSS-Fuzz builder images
(e.g. `gcr.io/oss-fuzz-base/base-builder`, see [the build integration docs] for
more details on this image) which actually handle building the fuzzers.
This step produces fuzzers that can be used for fuzzing, pruning, or coverage
reports if the sanitizer selected is `coverage`.
But, before ClusterFuzzLite starts running the fuzzers, the "build fuzzers" step
does a little processing of these fuzzers.

##### Deleting unaffected fuzzers

The next step in building fuzzers is deleting unaffected fuzzers.
This is done only during code change fuzzing.
ClusterFuzzLite does this by diffing the current state of the project repo
against either the `base_ref` or the `base_commit` to find files that were
modified by the change under test.
Then ClusterFuzzLite downloads coverage data produced by coverage report
generation (if it is available) and finds the set of files covered by each
fuzzer.
If any of the files covered by a fuzzer are among the changed files, the fuzzer
is considered "affected" by the current code change, otherwise the fuzzer is
considered "unaffected".
We realized this understanding is slightly simplistic.
It's possible that a fuzzer could be affected by a changed file that it hasn't
covered for two reasons:
1. The file could define a global variable used in code that is covered by the
   fuzzer.
2. The fuzzer could theoretically cover a file that it hasn't covered in the
   past during batch fuzzing.
We think 1 is uncommon enough that we ignore this possibility.
We think 2 is extremely unlikely.
If a fuzzer could not cover a file in the much longer time it has to do batch
fuzzing, it is unlikely to cover it during the much shorter time it has to do
code change fuzzing.
Fuzzing isn't about soundness or proof, it's about doing what works, and we
think this simplistic understanding allows us to find the most amount of bugs.

With this understanding of affected and unaffected fuzzers, ClusterFuzzLite will
delete any unaffected fuzzers.
The exceptions to this are:
- Fuzzers for which there is no coverage data are not deleted (as these can be
  new).
- If no fuzzers are affected, none of them are deleted.

The next step is checking fuzzers for common mistakes.

##### Checking fuzzers for common mistakes

This is done using OSS-Fuzz's bad build check functionality.
It does the following:
- Runs fuzzers for a short amount of time (~10 seconds) on no corpus to see if
  they crash trivially.
- Checks that the fuzzers actual sanitizer instrumentation martches the expected
  sanitizer instrumentation (specified by `SANITIZER`).

If enough fuzzers (more than 20%) fail these checks, the build is considered
"bad", fails in CI, and does not continue.

##### Uploading builds to the filestore

Finally, if the `UPLOAD_BUILD` environment variable is set, ClusterFuzzLite will
upload the fuzzers to the filestore (this is done in "continuous builds").

Once all of these steps have completed the build is used by the "run fuzzers"
step of ClusterFuzzLite.

#### Run fuzzers

The run fuzzers function of ClusterFuzzLite will do one of four things depending
on the `MODE` environment variable:
- Code change fuzzing.
- Batch fuzzing.
- Corpus pruning.
- Coverage report generation.

These are discussed more in the [docs on running ClusterFuzzLite].

[`infra/cifuzz` directory]: https://github.com/google/oss-fuzz/tree/master/infra/cifuzz
[OSS-Fuzz repo]: https://github.com/google/oss-fuzz
[CIFuzz]: https://google.github.io/oss-fuzz/getting-started/continuous-integration/
[pytest's -k option]: https://docs.pytest.org/en/6.2.x/example/markers.html#using-k-expr-to-select-tests-based-on-their-name
[cifuzz-external-example]: https://github.com/jonathanmetzman/cifuzz-external-example
[Travis CI]: https://travis-ci.org/
[docker]: https://docs.docker.com/get-docker/
[adding support for a new platform]: {{ site.baseurl }}/developing-clusterfuzz/new-platform/
[the build integration docs]: {{ site.baseurl }}/build-integration/
[Building the builder image and the fuzzers]: #Building-the-builder-image-and-the-fuzzers
[Deleting unaffected fuzzers]: #deleting-unaffected-fuzzers
[Checking fuzzers for common mistakes]: #checking-fuzzers-for-common-mistakes
[Uploading builds to the filestore]: #uploading-builds-to-the-filestore
[docs on running ClusterFuzzLite]: {{ site.baseurl }}/running-clusterfuzzlite/#clusterfuzzlite-modes
