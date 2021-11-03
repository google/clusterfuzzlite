---
layout: default
title: Integrating a Go project
parent: Build Integration
grand_parent: ClusterFuzzLite
nav_order: 1
permalink: /build-integration/go-lang/
---

# Integrating a Go project
{: .no_toc}

- TOC
{:toc}
---

The process of integrating a project written in Go with ClusterFuzzLite is very similar
to the general
[Build integration]({{ site.baseurl }}/build-integration/)
process. The key specifics of integrating a Go project are outlined below.

## Go-fuzz support

ClusterFuzzLite supports **go-fuzz** in the
[libFuzzer compatible mode](https://github.com/mdempsky/go114-fuzz-build)
only. In that mode, fuzz targets for Go use the libFuzzer engine with native Go
coverage instrumentation. Binaries compiled in this mode provide the same
libFuzzer command line interface as non-Go fuzz targets.

## Project files

First, you need to write a Go fuzz target that accepts a stream of bytes and
calls the program API with that. This fuzz target should reside in your project
repository
([example](https://github.com/golang/go/blob/4ad13555184eb0697c2e92c64c1b0bdb287ccc10/src/html/fuzz.go#L13)).

The structure of the `.clusterfuzzlite` directory doesn't differ for projects
written in Go. The project files have the following Go specific aspects.

### project.yaml

The `language` attribute must be specified.

```yaml
language: go
```

The only supported sanitizer is `address`.

### Dockerfile

The Dockerfile should start by `FROM gcr.io/oss-fuzz-base/base-builder-go`

The OSS-Fuzz builder image has the latest stable release of Golang installed. In
order to install dependencies of your project, add `RUN git clone ...` command to
your Dockerfile.

```dockerfile
# Dependency for one of the fuzz targets.
RUN git clone --depth 1 https://github.com/ianlancetaylor/demangle
```

go-fuzz will then automatically download the dependencies based on the go.mod file

### build.sh

In order to build a Go fuzz target, you need to call `go-fuzz`
command first, and then link the resulting `.a` file against
`$LIB_FUZZING_ENGINE` using the `$CXX $CXXFLAGS ...` command.

The best way to do this is by using a `compile_go_fuzzer` script,
as it also supports coverage builds.

A usage example from go-dns project is

```sh
compile_go_fuzzer github.com/miekg/dns FuzzNewRR fuzz_newrr fuzz
```

Arguments are :
* path of the package with the fuzz target
* name of the fuzz function
* name of the fuzzer to be built
* optional tag to be used by `go build` and such
