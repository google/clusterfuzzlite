---
layout: default
parent: Running ClusterFuzzLite
grand_parent: ClusterFuzzLite
title: Prow
nav_order: 3
permalink: /running-clusterfuzzlite/prow/
---
# Prow
{: .no_toc}

- TOC
{:toc}
---

This document explains how to set up ClusterFuzzLite on
[Prow](https://github.com/kubernetes/test-infra/tree/master/prow#readme).
This document assumes the reader has set up Prow on their repo.

First, you must
[create a Google Cloud Storage Bucket](https://cloud.google.com/storage/docs/creating-buckets).
This will be refered to as
`<your-cloud-bucket>` in the config examples in this document. If
The bucket will be used to store crashes, builds, corpora, and coverage reports.

Then you must create a service account with access to both `<your-cloud-bucket>` and the Prow Results Bucket
To do this
  1) Create a Cloud Service account in your Project

    gcloud beta iam service-accounts create `<SA_NAME>` --project=`<PROJECT_ID>` --description=`<SA_DESCRIPTION>` --display-name=`<SA_DISPLAY_NAME>`

  2) Give the Cloud Service account access to BOTH the k8s Prow Results Bucket AND `<your-cloud-bucket>` using the command

    gsutil iam ch "serviceAccount:${SA}:roles/storage.objectAdmin" "${BUCKET}"
    
  3) Create a new K8s service account with a distinct name (called `<K8s_SA>` for the rest of this documentation)

  4) Add the Google Cloud Service account to the annotations for the K8s service account like so:
      ```
      apiVersion: v1
      kind: ServiceAccount
      metadata:
        annotations:
          iam.gke.io/gcp-service-account: <YOUR_CLOUD_SERVICE_ACCOUNT>
        name: `<K8s_SA>`
        namespace: test-pods
      ```
  5) Run [bind-service-accounts.sh](https://github.com/kubernetes/test-infra/tree/f7e21a3c18f4f4bbc7ee170675ed53e4544a0632/workload-identity) to bind `<K8s_SA>` to your Google Cloud Service Account

TODO(mpherman): Create a script to automatically ensure Service Accounts and Buckets to reduce setup steps.

Note that the configuration examples set the environment variable
`CFL_PLATFORM` these values are general for all GCB users. Do not
change them.

## Presubmit/Postsubmit fuzzing

To add a fuzzing workflow that runs on all pull requests to your project, add
the following to your job config under either Presubmits or Postsubmits:

{% raw %}
```yaml
- name: <YOUR-PROWJOB-NAME>
  labels:
    preset-dind-enabled: "true"
  decorate: true
  spec:
    serviceAccountName: <K8s-SA>
    containers:
    - image: gcr.io/k8s-testimages/ci_fuzz:<VERSION>
      command:
        - runner.sh
      args:
        - python3
        - "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
      # docker-in-docker needs privileged mode
      securityContext:
        privileged: true
      env:
      - name: MODE
        value: code-change
      - name: SANITIZER
        value: <SANITIZER>
      - name: CLOUD_BUCKET
        value: <YOUR-CLOUD-BUCKET>
      - name: CFL_PLATFORM
        value: prow
```
{% endraw %}

You can set the `FUZZ_SECONDS` env var when running the fuzzers to change the
amount of time ClusterFuzzLite will fuzz PRs for.

## Periodic Fuzzing

To enable batch fuzzing, add the following to your prow job config for periodic jobs.

{% raw %}
```yaml
- name: <YOUR-PROWJOB-NAME>
  labels:
    preset-dind-enabled: "true"
  decorate: true
  extra_refs:
  - org: <ORG>
    repo: <REPO>
    base_ref: <BRANCH>
  interval: 1h # THIS CAN BE A CRON AS WELL
  spec:
    serviceAccountName: <K8s-SA>
    containers:
    - image: gcr.io/k8s-testimages/ci_fuzz:<VERSION>
      command:
        - runner.sh
      args:
        - python3
        - "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
      # docker-in-docker needs privileged mode
      securityContext:
        privileged: true
      env:
      - name: MODE
        value: batch
      - name: SANITIZER
        value: <SANITIZER>
      - name: CLOUD_BUCKET
        value: gs://prow-cifuzz-test/
      - name: CFL_PLATFORM
        value: prow
```

{% endraw %}

You can set the `FUZZ_SECONDS` value when running the fuzzers to change the
amount of time ClusterFuzzLite will fuzz PRs for.

## Corpus pruning

Corpus pruning minimizes a corpus by removing redundant items while keeping the
same code coverage. To enable this, add the following to your prow job config for periodic jobs.

{% raw %}
```yaml
- name: <YOUR-PROWJOB-NAME>
  labels:
    preset-dind-enabled: "true"
  decorate: true
  extra_refs:
  - org: <ORG>
    repo: <REPO>
    base_ref: <BRANCH>
  interval: 1h # THIS CAN BE A CRON AS WELL
  spec:
    serviceAccountName: <K8s-SA>
    containers:
    - image: gcr.io/k8s-testimages/ci_fuzz:<VERSION>
      command:
        - runner.sh
      args:
        - python3
        - "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
      # docker-in-docker needs privileged mode
      securityContext:
        privileged: true
      env:
      - name: MODE
        value: prune
      - name: SANITIZER
        value: <SANITIZER>
      - name: CLOUD_BUCKET
        value: gs://prow-cifuzz-test/
      - name: CFL_PLATFORM
        value: prow
```
{% endraw %}


## Coverage reports

Periodic coverage reports can also be generated using the latest corpus. To enable this, add the following to your prow job config for periodic jobs.

{% raw %}
```yaml
- name: <YOUR-PROWJOB-NAME>
  labels:
    preset-dind-enabled: "true"
  decorate: true
  extra_refs:
  - org: <ORG>
    repo: <REPO>
    base_ref: <BRANCH>
  interval: 1h # THIS CAN BE A CRON AS WELL
  spec:
    serviceAccountName: <K8s-SA>
    containers:
    - image: gcr.io/k8s-testimages/ci_fuzz:<VERSION>
      command:
        - runner.sh
      args:
        - python3
        - "/opt/oss-fuzz/infra/cifuzz/cifuzz_combined_entrypoint.py"
      # docker-in-docker needs privileged mode
      securityContext:
        privileged: true
      env:
      - name: MODE
        value: coverage
      - name: SANITIZER
        value: <SANITIZER>
      - name: CLOUD_BUCKET
        value: gs://prow-cifuzz-test/
      - name: CFL_PLATFORM
        value: prow
```
{% endraw %}

We reccommend scheduling coverage reports to run once a day.
Note that it's better to schedule this to run after pruning has
completed, but not required.

## Testing it Out

You can test each of the workflows by [Running the Prowjob Locally](https://github.com/kubernetes/test-infra/blob/master/prow/build_test_update.md#running-a-prowjob-locally)


## Storage Layout

Artifacts produced by ClusterFuzzLite are stored in `<your-cloud-bucket>`.
The bucket has the following layout.
1. Coverage report: <your-cloud-bucket>/coverage/latest/report/linux/index.html
1. Builds: <your-cloud-bucket>/build/<sanitizer>-<commit>/
1. Corpora: <your-cloud-bucket>/corpus/<fuzz-target>/
1. Crashes: <your-cloud-bucket>/crashes/<fuzz-target>/<sanitizer>/<crash-file> and <your-cloud-bucket>/crashes/<fuzz-target>/<sanitizer>/<crash-file>.summary (output from crash).

## Viewing Crashes

You can view your crashes on Deck by clicking the Artifacts button on the top of the page for your prowjob run. Crashes will be located in 
`artifacts/out/artifacts/fuzzer/`