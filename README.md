# ClusterFuzzLite
ClusterFuzzLite is a continuous [fuzzing](https://en.wikipedia.org/wiki/Fuzzing)
solution that runs as part of
[Continuous Integration (CI)](https://en.wikipedia.org/wiki/Continuous_integration)
workflows to find vulnerabilities faster than ever before.
With just a few lines of code, GitHub users can integrate ClusterFuzzLite into
their workflow and fuzz pull requests to catch bugs before they are committed.

ClusterFuzzLite is based on [ClusterFuzz](https://google.github.io/clusterfuzz/).

## Features

- Quick code change (pull request) fuzzing to find bugs before they land
- Downloads of crashing testcases
- Continuous longer running fuzzing (batch fuzzing) to asynchronously find
   deeper bugs missed during code change fuzzing and build a corpus for
   use in code change fuzzing
- Coverage reports showing which parts of your code are fuzzed
- Modular functionality, so you can decide which features you want to use

![demo](https://raw.githubusercontent.com/google/clusterfuzzlite/refs/heads/bucket/images/demo.gif)

## Supported Languages
- C
- C++
- Java (and other JVM-based languages)
- Go
- Python
- Rust
- Swift

## Supported CI Systems
- GitHub Actions
- GitLab
- Google Cloud Build
- Prow
- Support for more CI systems is in-progess, and extending support to other CI
  systems is easy

## Documentation

Read our [detailed documentation](https://google.github.io/clusterfuzzlite) to learn how
to use ClusterFuzzLite.

## Staying in touch
Join our [mailing list](https://groups.google.com/g/clusterfuzzlite-users) for
announcements and discussions.

If you use ClusterFuzzLite, please fill out [this form](https://docs.google.com/forms/d/e/1FAIpQLSdAKB03YM4HjMwNe1K4T6Yr16OE4lCMj-VzThuUOrZUc3ytWw/viewform?usp=sf_link)
so we know who is using it.
This gives us an idea of the impact of ClusterFuzzLite and allows us to
justify future work.

Feel free to
[file an issue](https://github.com/google/clusterfuzzlite/issues/new)
if you experience any trouble or have feature requests.
