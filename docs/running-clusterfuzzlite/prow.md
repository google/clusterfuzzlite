---
layout: default
parent: Running ClusterFuzzLite
grand_parent: ClusterFuzzLite
title: Prow
nav_order: 3
permalink: /running-clusterfuzzlite/prow/
---
# Prow (beta)
{: .no_toc}

- TOC
{:toc}
---

**NOTE: Clusterfuzzlite for Prow is in beta. If there are any issues please file at github.com/google/oss-fuzz/issues**

This document explains how to set up ClusterFuzzLite on [Prow].
This document assumes the reader has set up Prow on their repo.

First, you must [create a Google Cloud Storage Bucket].
This will be refered to as `<your-cloud-bucket>` in the config examples in this
document.
The bucket will be used to store crashes, builds, corpora, and coverage reports.

Then you must create a service account with access to both `<your-cloud-bucket>`
and the Prow Results Bucket.
To do this:
1. Create a Cloud Service account in your Project
   ```
   gcloud beta iam service-accounts create `<sa-name>` --project=`<project-id>` --description=`<sa-description>` --display-name=`<sa-display-name>`
   ```

2. Give the Cloud Service account access to **BOTH** the k8s Prow Results Bucket
   AND `<your-cloud-bucket>` using these commands:

  ```bash
  gsutil iam ch "serviceAccount:<sa-name>:roles/storage.objectAdmin" "<your-cloud-bucket>"
  gsutil iam ch "serviceAccount:<sa-name>:roles/storage.objectAdmin" "<k8s-Prow-results-bucket>"
  ```

3. Create a new K8s service account with a distinct name (referred `<K8s-SA>`
   for the rest of this document):

4. Add the Google Cloud Service account to the annotations for the K8s service
   account like so:
   ```
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     annotations:
       iam.gke.io/gcp-service-account: <your-cloud-service-account>
     name: `<K8s-SA>`
     namespace: test-pods
   ```
5. Run
   [bind-service-accounts.sh]
   to bind `<K8s-SA>` to your Google Cloud Service Account.

<!--
TODO(mpherman): Create a script to automatically ensure Service Accounts and Buckets to reduce setup steps.
-->

Note that the configuration examples set the environment variable `CFL_PLATFORM`
this value is general for all prow users. Do not change it.

## Presubmit/Postsubmit fuzzing

To add a fuzzing workflow that runs on all pull requests to your project, add
the following to your job config under either Presubmits or Postsubmits:

{% raw %}
```yaml
- name: <your-prowjob-name>
  labels:
    preset-dind-enabled: "true"
  decorate: true
  spec:
    serviceAccountName: <K8s-SA>
    containers:
    - image: gcr.io/k8s-testimages/clusterfuzzlite:latest
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
        value: address # Change this to whatever sanitizer you want to use.
      - name: CLOUD_BUCKET
        value: <your-cloud-bucket>
      - name: CFL_PLATFORM
        value: prow
```
{% endraw %}

You can copy and paste this example, but make sure to set
`<your-prowjob-name>`, `<K8s-SA>`, and `<your-cloud-bucket>` to the appropriate
values.

You can set the `FUZZ_SECONDS` env var to change the amount of time
ClusterFuzzLite will fuzz PRs for.
You can modify the `SANITIZER` env var to pick the sanitizer you want to use.

## Batch Fuzzing

To enable batch fuzzing, add the following to your prow job config for periodic
jobs:

{% raw %}
```yaml
- name: <your-prowjob-name>
  labels:
    preset-dind-enabled: "true"
  decorate: true
  extra_refs:
  - org: <org>
    repo: <repo>
    base_ref: <branch> # This should be the branch you want to fuzz.
  interval: 24h  # This can be a cron as well.
  spec:
    serviceAccountName: <K8s-SA>
    containers:
    - image: gcr.io/k8s-testimages/clusterfuzzlite:latest
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
        value: address # Change this to whatever sanitizer you want to use.
      - name: CLOUD_BUCKET
        value: <your-cloud-bucket>
      - name: FUZZ_SECONDS
        value: 3600  # 1 Hour. You  can change this.
      - name: CFL_PLATFORM
        value: prow
```

{% endraw %}

You can copy and paste this example, but make sure to set
`<your-prowjob-name>`, `<K8s-SA>`, `<your-cloud-bucket>`, `<org>`, `<repo>` and
`<branch>` to the appropriate values.

You can modify the `FUZZ_SECONDS` env var to change the amount of time
ClusterFuzzLite will fuzz for.
You can modify the `SANITIZER` env var to pick the sanitizer you want to use.

## Corpus pruning

Corpus pruning minimizes a corpus by removing redundant items while keeping the
same code coverage. To enable this, add the following to your prow job config for periodic jobs.

{% raw %}
```yaml
- name: <your-prowjob-name>
  labels:
    preset-dind-enabled: "true"
  decorate: true
  extra_refs:
  - org: <org>
    repo: <repo>
    base_ref: <branch> # This should be the branch you want to fuzz.
  interval: 24h # This can be a cron as well.
  spec:
    serviceAccountName: <K8s-SA>
    containers:
    - image: gcr.io/k8s-testimages/clusterfuzzlite:latest
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
      - name: CLOUD_BUCKET
        value: <your-cloud-bucket>
      - name: CFL_PLATFORM
        value: prow
```
{% endraw %}

You can copy and paste this example, but make sure to set
`<your-prowjob-name>`, `<K8s-SA>`, `<your-cloud-bucket>`, `<org>`, `<repo>` and
`<branch>` to the appropriate values.
We recommend scheduling pruning to run once a day.

## Coverage reports

Periodic coverage reports can also be generated using the latest corpus.
To enable this, add the following to your prow job config for periodic jobs:

{% raw %}
```yaml
- name: <your-prowjob-name>
  labels:
    preset-dind-enabled: "true"
  decorate: true
  extra_refs:
  - org: <org>
    repo: <repo>
    base_ref: <branch> # This should be the branch you want to fuzz.
  interval: 24h # Note: this can be a cron as well.
  spec:
    serviceAccountName: <K8s-SA>
    containers:
    - image: gcr.io/k8s-testimages/clusterfuzzlite:latest
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
        value: coverage
      - name: CLOUD_BUCKET
        value: <your-cloud-bucket>
      - name: CFL_PLATFORM
        value: prow
```
{% endraw %}

You can copy and paste this example, but make sure to set
`<your-prowjob-name>`, `<K8s-SA>`, `<your-cloud-bucket>`, `<org>`, `<repo>` and
`<branch>` to the appropriate values.
We recommend scheduling coverage reports to run once a day.

## Testing it Out

You can test each of the workflows by [Running the Prowjob Locally].

## Storage Layout

Artifacts produced by ClusterFuzzLite are stored in `<your-cloud-bucket>`.
The bucket has the following layout.
1. Coverage report: <your-cloud-bucket>/coverage/latest/report/linux/index.html
1. Builds: <your-cloud-bucket>/build/<sanitizer>-<commit>/
1. Corpora: <your-cloud-bucket>/corpus/<fuzz-target>/
1. Crashes: <your-cloud-bucket>/crashes/<fuzz-target>/<sanitizer>/<crash-file>
   and
   <your-cloud-bucket>/crashes/<fuzz-target>/<sanitizer>/<crash-file>.summary
   (output from crash).

## Viewing Crashes

You can view your crashes on Deck by clicking the Artifacts button on the top of
the page for your prowjob run.
Crashes will be located in `artifacts/out/artifacts/fuzzer/`.
<!--
TODO(mpherman): Get a screenshot of this.
-->

[bind-service-accounts.sh]: https://github.com/kubernetes/test-infra/tree/f7e21a3c18f4f4bbc7ee170675ed53e4544a0632/workload-identity
[create a Google Cloud Storage Bucket]: https://cloud.google.com/storage/docs/creating-buckets
[Prow]: https://github.com/kubernetes/test-infra/tree/master/prow#readme
[Running the Prowjob Locally]: https://github.com/kubernetes/test-infra/tree/master/prow#readme
