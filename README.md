# buildx

[![PkgGoDev](https://img.shields.io/badge/go.dev-docs-007d9c?logo=go&logoColor=white)](https://pkg.go.dev/github.com/docker/buildx)
[![Build Status](https://github.com/docker/buildx/workflows/build/badge.svg)](https://github.com/docker/buildx/actions?query=workflow%3Abuild)
[![Go Report Card](https://goreportcard.com/badge/github.com/docker/buildx)](https://goreportcard.com/report/github.com/docker/buildx)
[![codecov](https://codecov.io/gh/docker/buildx/branch/master/graph/badge.svg)](https://codecov.io/gh/docker/buildx)

`buildx` is a Docker CLI plugin for extended build capabilities with
[BuildKit](https://github.com/moby/buildkit).

Key features:

- Familiar UI from `docker build`
- Full BuildKit capabilities with container driver
- Multiple builder instance support
- Multi-node builds for cross-platform images
- Compose build support
- High-level build constructs (`bake`)
- In-container driver support (both Docker and Kubernetes)

# Table of Contents

- [Installing](#installing)
  - [Docker](#docker)
  - [Binary release](#binary-release)
  - [From `Dockerfile`](#from-dockerfile)
- [Set buildx as the default builder](#set-buildx-as-the-default-builder)
- [Building](#building)
- [Getting started](#getting-started)
  - [Building with buildx](#building-with-buildx)
  - [Working with builder instances](#working-with-builder-instances)
  - [Building multi-platform images](#building-multi-platform-images)
  - [High-level build options](#high-level-build-options)
- [Documentation](docs/reference/buildx.md)
  - [`buildx bake`](docs/reference/buildx_bake.md)  
  - [`buildx build`](docs/reference/buildx_build.md)
  - [`buildx create`](docs/reference/buildx_create.md)
  - [`buildx du`](docs/reference/buildx_du.md)
  - [`buildx imagetools`](docs/reference/buildx_imagetools.md)
    - [`buildx imagetools create`](docs/reference/buildx_imagetools_create.md)
    - [`buildx imagetools inspect`](docs/reference/buildx_imagetools_inspect.md)  
  - [`buildx inspect`](docs/reference/buildx_inspect.md)
  - [`buildx install`](docs/reference/buildx_install.md)
  - [`buildx ls`](docs/reference/buildx_ls.md)  
  - [`buildx prune`](docs/reference/buildx_prune.md)  
  - [`buildx rm`](docs/reference/buildx_rm.md)
  - [`buildx stop`](docs/reference/buildx_stop.md)
  - [`buildx uninstall`](docs/reference/buildx_uninstall.md)
  - [`buildx use`](docs/reference/buildx_use.md)
  - [`buildx version`](docs/reference/buildx_version.md)
- [Contributing](#contributing)

# Installing

Using `buildx` as a docker CLI plugin requires using Docker 19.03 or newer.
A limited set of functionality works with older versions of Docker when
invoking the binary directly.

## Docker

`buildx` comes bundled with Docker Desktop and in latest
[Docker CE packages](https://docs.docker.com/engine/install/), but may not be
included in third-party software components (in which case follow the
[binary release](#binary-release) instructions).

## Binary release

You can also download the latest `buildx` binary from the
[GitHub releases](https://github.com/docker/buildx/releases/latest) page, copy
it to `~/.docker/cli-plugins` folder with name `docker-buildx` and change the
permission to execute:

```console
$ chmod a+x ~/.docker/cli-plugins/docker-buildx
```

## From `Dockerfile`

Here is how to use buildx inside a Dockerfile through the
[`docker/buildx-bin`](https://hub.docker.com/r/docker/buildx-bin) image:

```Dockerfile
FROM docker
COPY --from=docker/buildx-bin /buildx /usr/libexec/docker/cli-plugins/docker-buildx
RUN docker buildx version
```

# Set buildx as the default builder

Running the command [`docker buildx install`](docs/reference/buildx_install.md)
sets up docker builder command as an alias to `docker buildx build`. This
results in the ability to have `docker build` use the current buildx builder.

To remove this alias, run [`docker buildx uninstall`](docs/reference/buildx_uninstall.md).

# Building

```console
# Buildx 0.6+
$ docker buildx bake "git://github.com/docker/buildx"
$ mkdir -p ~/.docker/cli-plugins
$ mv ./bin/buildx ~/.docker/cli-plugins/docker-buildx

# Docker 19.03+
$ DOCKER_BUILDKIT=1 docker build --platform=local -o . "git://github.com/docker/buildx"
$ mkdir -p ~/.docker/cli-plugins
$ mv buildx ~/.docker/cli-plugins/docker-buildx

# Local 
$ git clone git://github.com/docker/buildx && cd buildx
$ make install
```

# Getting started

## Building with buildx

Buildx is a Docker CLI plugin that extends the `docker build` command with the
full support of the features provided by [Moby BuildKit](https://github.com/moby/buildkit)
builder toolkit. It provides the same user experience as `docker build` with
many new features like creating scoped builder instances and building against
multiple nodes concurrently.

After installation, buildx can be accessed through the `docker buildx` command
with Docker 19.03.  `docker buildx build` is the command for starting a new
build. With Docker versions older than 19.03 buildx binary can be called
directly to access the `docker buildx` subcommands.

```console
$ docker buildx build .
[+] Building 8.4s (23/32)
 => ...
```

Buildx will always build using the BuildKit engine and does not require
`DOCKER_BUILDKIT=1` environment variable for starting builds.

The `docker buildx build` command supports features available for `docker build`,
including features such as outputs configuration, inline build caching, and
specifying target platform. In addition, Buildx also supports new features that
are not yet available for regular `docker build` like building manifest lists,
distributed caching, and exporting build results to OCI image tarballs.

Buildx is supposed to be flexible and can be run in different configurations
that are exposed through a driver concept. Currently, we support a
[`docker` driver](docs/reference/buildx_create.md#docker-driver) that uses
the BuildKit library bundled into the Docker daemon binary, a
[`docker-container` driver](docs/reference/buildx_create.md#docker-container-driver)
that automatically launches BuildKit inside a Docker container and a
[`kubernetes` driver](docs/reference/buildx_create.md#kubernetes-driver) to
spin up pods with defined BuildKit container image to build your images. We
plan to add more drivers in the future.

The user experience of using buildx is very similar across drivers, but there
are some features that are not currently supported by the `docker` driver,
because the BuildKit library bundled into docker daemon currently uses a
different storage component. In contrast, all images built with `docker` driver
are automatically added to the `docker images` view by default, whereas when
using other drivers the method for outputting an image needs to be selected
with `--output`.

## Working with builder instances

By default, buildx will initially use the `docker` driver if it is supported,
providing a very similar user experience to the native `docker build`. Note tha
you must use a local shared daemon to build your applications.

Buildx allows you to create new instances of isolated builders. This can be
used for getting a scoped environment for your CI builds that does not change
the state of the shared daemon or for isolating the builds for different
projects. You can create a new instance for a set of remote nodes, forming a
build farm, and quickly switch between them.

You can create new instances using the [`docker buildx create`](docs/reference/buildx_create.md)
command. This creates a new builder instance with a single node based on your
current configuration.

To use a remote node you can specify the `DOCKER_HOST` or the remote context name
while creating the new builder. After creating a new instance, you can manage its
lifecycle using the [`docker buildx inspect`](docs/reference/buildx_inspect.md),
[`docker buildx stop`](docs/reference/buildx_stop.md), and
[`docker buildx rm`](docs/reference/buildx_rm.md) commands. To list all
available builders, use [`buildx ls`](docs/reference/buildx_ls.md). After
creating a new builder you can also append new nodes to it.

To switch between different builders, use [`docker buildx use <name>`](docs/reference/buildx_use.md).
After running this command, the build commands will automatically use this
builder.

Docker also features a [`docker context`](https://docs.docker.com/engine/reference/commandline/context/)
command that can be used for giving names for remote Docker API endpoints.
Buildx integrates with `docker context` so that all of your contexts
automatically get a default builder instance. While creating a new builder
instance or when adding a node to it you can also set the context name as the
target.

## Building multi-platform images

BuildKit is designed to work well for building for multiple platforms and not
only for the architecture and operating system that the user invoking the build
happens to run.

When you invoke a build, you can set the `--platform` flag to specify the target
platform for the build output, (for example, `linux/amd64`, `linux/arm64`, or
`darwin/amd64`).

When the current builder instance is backed by the `docker-container` or 
`kubernetes` driver, you can specify multiple platforms together. In this case,
it builds a manifest list which contains images for all specified architectures.
When you use this image in [`docker run`](https://docs.docker.com/engine/reference/commandline/run/)
or [`docker service`](https://docs.docker.com/engine/reference/commandline/service/),
Docker picks the correct image based on the node's platform.

You can build multi-platform images using three different strategies that are
supported by Buildx and Dockerfiles:

1. Using the QEMU emulation support in the kernel
2. Building on multiple native nodes using the same builder instance
3. Using a stage in Dockerfile to cross-compile to different architectures

QEMU is the easiest way to get started if your node already supports it (for
example. if you are using Docker Desktop). It requires no changes to your
Dockerfile and BuildKit automatically detects the secondary architectures that
are available. When BuildKit needs to run a binary for a different architecture,
it automatically loads it through a binary registered in the `binfmt_misc`
handler.

For QEMU binaries registered with `binfmt_misc` on the host OS to work
transparently inside containers they must be registered with the `fix_binary`
flag. This requires a kernel >= 4.8 and binfmt-support >= 2.1.7. You can check
for proper registration by checking if `F` is among the flags in
`/proc/sys/fs/binfmt_misc/qemu-*`. While Docker Desktop comes preconfigured
with `binfmt_misc` support for additional platforms, for other installations
it likely needs to be installed using [`tonistiigi/binfmt`](https://github.com/tonistiigi/binfmt)
image.

```console
$ docker run --privileged --rm tonistiigi/binfmt --install all
```

Using multiple native nodes provide better support for more complicated cases
that are not handled by QEMU and generally have better performance. You can
add additional nodes to the builder instance using the `--append` flag.

Assuming contexts `node-amd64` and `node-arm64` exist in `docker context ls`;

```console
$ docker buildx create --use --name mybuild node-amd64
mybuild
$ docker buildx create --append --name mybuild node-arm64
$ docker buildx build --platform linux/amd64,linux/arm64 .
```

Finally, depending on your project, the language that you use may have good
support for cross-compilation. In that case, multi-stage builds in Dockerfiles
can be effectively used to build binaries for the platform specified with
`--platform` using the native architecture of the build node. A list of build
arguments like `BUILDPLATFORM` and `TARGETPLATFORM` is available automatically
inside your Dockerfile and can be leveraged by the processes running as part
of your build.

```dockerfile
FROM --platform=$BUILDPLATFORM golang:alpine AS build
ARG TARGETPLATFORM
ARG BUILDPLATFORM
RUN echo "I am running on $BUILDPLATFORM, building for $TARGETPLATFORM" > /log
FROM alpine
COPY --from=build /log /log
```

You can also use [`tonistiigi/xx`](https://github.com/tonistiigi/xx) Dockerfile
cross-compilation helpers for more advanced use-cases.

## High-level build options

Buildx also aims to provide support for high-level build concepts that go beyond
invoking a single build command. We want to support building all the images in
your application together and let the users define project specific reusable
build flows that can then be easily invoked by anyone.

BuildKit efficiently handles multiple concurrent build requests and
de-duplicating work. The build commands can be combined with general-purpose
command runners (for example, `make`). However, these tools generally invoke
builds in sequence  and therefore cannot leverage the full potential of BuildKit
parallelization, or combine BuildKit’s output for the user. For this use case,
we have added a command called [`docker buildx bake`](docs/reference/buildx_bake.md).

The `bake` command supports building images from compose files, similar to
[`docker-compose build`](https://docs.docker.com/compose/reference/build/),
but allowing all the services to be built concurrently as part of a single
request.

There is also support for custom build rules from HCL/JSON files allowing
better code reuse and different target groups. The design of bake is in very
early stages and we are looking for feedback from users.

# Contributing

Want to contribute to Buildx? Awesome! You can find information about
contributing to this project in the [CONTRIBUTING.md](/.github/CONTRIBUTING.md)
