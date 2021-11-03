---
layout: default
parent: ClusterFuzzLite
title: Running ClusterFuzzLite
has_children: true
nav_order: 3
permalink: /running-clusterfuzzlite/
---
# Running ClusterFuzzLite
{: .no_toc}

- TOC
{:toc}
---

## Overview
![overview]({{ site.baseurl }}/assets/overview.png)

This document assumes you have read the document on
[integrating with ClusterFuzzLite's build system].
Once your project's fuzzers can be built and run by the OSS-Fuzz/ClusterFuzzLite
helper script, it is ready to be fuzzed by ClusterFuzzLite.

The exact method for doing this will depend on the how you are running
ClusterFuzzLite (i.e. which CI system).
The rest of this page will explain general concepts and configuration options
that are important to understand but are agnostic to how ClusterFuzzLite is run.
After reading this page, you can check out the [subguides] for instructions how
to run them for your particular CI system (e.g. [GitHub Actions]).

## ClusterFuzzLite modes

ClusterFuzzLite offers two flavors of fuzzing, code-change fuzzing and batch
fuzzing as well as two helper modes that don't fuzz but provide useful
functionality: "prune" and "coverage".
ClusterFuzzLite can be instructed to perform any of these functions using the
"mode" option.

### Code Change Fuzzing ("code-change")

One of the core ways for using ClusterFuzzLite is for fuzzing code changes which
were introduced in a pull request or commit.

This allows ClusterFuzzLite to find bugs before they are commited into your
code and while they are easiest to fix.

Because Code Change Fuzzing needs to find bugs quickly (so that developers
aren't stuck waiting for it to complete), it defaults to running for 10 minutes,
[though this can be changed](#configuration-options)

If [Batch Fuzzing] is enabled, Code Change Fuzzing will report only
newly introduced bugs and use the corpus developed during batch fuzzing.
If [Code Coverage report generation] is enabled, Code Change Fuzzing will try to
only run the fuzzers affected by the code change, instead of running all of the
fuzzers built by your project.

### Batch Fuzzing ("batch")

ClusterFuzzLite can also run in a batch fuzzing mode where all fuzzers are run
for a longer amount of time. This allows ClusterFuzzLite to build up a [corpus]
for each of your fuzz targets, leading to more code coverage and better bug
discovery. This corpus will be used in [Code Coverage report generation]
as well as Code Review Fuzzing.

[corpus]: https://github.com/google/fuzzing/blob/master/docs/glossary.md#corpus

### Corpus Pruning ("pruning")

Corpus pruning minimizes a corpus by removing redundant items while keeping the
same code coverage.

Over time redundant testcases may get introduced into your corpus during
fuzzing. Running corpus pruning once a day will prevent buildup of these
redundant testcases and keep fuzzing efficient.

### Code Coverage report generation ("coverage")

ClusterFuzzLite also provides code coverage report generation. This will run
your fuzzers on the corpus developed during [Batch Fuzzing] and will
generate an HTML report that shows you which part of your code is covered by
batch fuzzing.

## Configuration Options

Below are some configuration options that you can set when running
ClusterFuzzLite.
We will explain how to set these in each of the subguides.

`language`: (optional) The language your target program is written in. Defaults
to `c++`. This should be the same as the value you set in `project.yaml`. See
[this explanation] for more details.

`fuzz-time`: Determines how long ClusterFuzzLite spends fuzzing your project in
seconds. The default is 600 seconds.

`sanitizer`: Determines a sanitizer to build and run fuzz targets with. The
choices are `'address'`, and `'undefined'`. The default is `'address'`.

`mode`: The mode for ClusterFuzzLite to execute. `code-change`
by default. See [ClusterFuzzLite modes] for more details on
how to run different modes.

`dry-run`: Determines if ClusterFuzzLite surfaces bugs/crashes. The default
value is `false`. When set to `true`, ClusterFuzzLite will never report a
failure even if it finds a crash in your project. This requires the user to
manually check the logs for detected bugs.

## Supported Continous Integration systems {#subguides}

- [GitHub Actions]
- [Google Cloud Build]

[subguides]: #subguides
[Google Cloud Build]: {{ site.baseurl }}/google-cloud-build/
[integrating with ClusterFuzzLite's build system]: {{ site.baseurl }}/build-integration/
[Batch Fuzzing]: #batch-fuzzing-batch
[Code Coverage report generation]: #code-coverage-report-generation-coverage
[this explanation]: {{ site.baseurl }}/build-integration/#language
[ClusterFuzzLite modes]: #clusterfuzzlite-modes
[GitHub Actions]: {{ site.baseurl }}/running-clusterfuzzlite/github-actions/
