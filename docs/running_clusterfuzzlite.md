---
layout: default
parent: ClusterFuzzLite
title: Running ClusterFuzzLite
has_children: true
nav_order: 4
permalink: /running-clusterfuzzlite/
---
# Running ClusterFuzzLite
{: .no_toc}

- TOC
{:toc}
---

## Overview
![overview]({{ site.baseurl }}/assets/overview.png)

Once your project's fuzzers can be built and run by the OSS-Fuzz helper script,
it is ready to be fuzzed by ClusterFuzzLite.

The exact method for doing this will depend on the how you are running
ClusterFuzzLite. For guides on how to run ClusterFuzzLite in your particular
environment (e.g. GitHub Actions) see the subguides.
The rest of this page will explain concepts and configuration options that are
agnostic to how ClusterFuzzLite is being run.

## ClusterFuzzLite modes

ClusterFuzzLite has several modes, which run as workflows on your CI.

### Code Change Fuzzing ("code-change")

One of the core ways for ClusterFuzzLite to be used is for fuzzing code changes
which were introduced in a pull request or commit.

This allows ClusterFuzzLite to find bugs before they are commited into your
code and while they are easiest to fix.

If [Batch Fuzzing] is enabled, Code Review Fuzzing will report only
newly introduced bugs and use the corpus developed during batch fuzzing.
If [Code Coverage report generation] is enabled, Code Review Fuzzing will try
to only run the fuzzers affected by the code change.

### Batch Fuzzing ("batch")

ClusterFuzzLite can also run in a batch fuzzing mode where all fuzzers are run
for a long amount of time. This allows ClusterFuzzLite to build up a [corpus]
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
[this explanation]({{ site.baseurl }}/build-integration/#language)
for more details.

`fuzz-time`: Determines how long ClusterFuzzLite spends fuzzing your project in
seconds. The default is 600 seconds.

`sanitizer`: Determines a sanitizer to build and run fuzz targets with. The
choices are `'address'`, and `'undefined'`. The default is `'address'`.

`mode`: The mode for ClusterFuzzLite to execute. `code-change`
by default. See [ClusterFuzzLite modes](#clusterfuzzlite-modes) for more details on
how to run different modes.

`dry-run`: Determines if ClusterFuzzLite surfaces bugs/crashes. The default
value is `false`. When set to `true`, ClusterFuzzLite will never report a
failure even if it finds a crash in your project. This requires the user to
manually check the logs for detected bugs.

## Supported Continous Integration systems

- [GitHub Actions]({{ site.baseurl }}/running-clusterfuzzlite/github-actions/)

[Batch Fuzzing]: #batch-fuzzing-batch
[Code Coverage report generation]: #code-coverage-report-generation-coverage
