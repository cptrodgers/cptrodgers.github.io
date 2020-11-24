---
layout: post
title: Use Dockerfile with Private Dependencies in Rust
published: true
---
# Using Docker to Build Rust Project with Publish and Private Dependencies (Git)

Rust is an awesome programming language. It has more selling points such as Fast, Concurrent, and Safe (Memory safe, concurrent safe, ...). But, as a young language, it also has some downsides; one of them is compiling time. The contributors of compiling time are Updating Index, Downloading, Checking, Optimizing, etc. Nowadays, almost companies want to add, test, and deploy with small lifecycle feature/fix development. This post will be focusing on improving the build flow of rust project by using Dockerfile, private dependencies, and multi-stage of Docker build.

# Private github repo and how to use it while docker build

First, we will come to a simple example. The project contains one private lib, it's stored by Github. In Rust project, you can use git source as a dependency; You can read at [here](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#specifying-dependencies-from-git-repositories). Git location in Cargo.toml is using the `master` branch as the default branch. You can specify your code by: `revision (commit)`, `tag`, or `branch`. Examples:

```rust
lib_1 = { git = "ssh://git@github.com/cptrodgers/rust-docker-example", branch = "deploy"  }
lib_2 = { git = "ssh://git@github.com/cptrodgers/rust-docker-example", revision = "abcxyz"  }
lib_3 = { git = "ssh://git@github.com/cptrodgers/rust-docker-example", tag = "a"  }

```

If it's a public repo, you can easily download it. If it's a private repo, Rust team is supporting `ssh-agent` to verify and download; You should use `ssh-add -K ~/.ssh/id_rsa` (or your private key path).

You will face some problems when using private repo in Docker build. Fortunately, the Docker team provided one solution for `ssh-key` in Docker build. It is DOCKER BUILDKIT. In short, with Docker Buildkit, you can make a socket to transfer your local `ssh-agent` to Docker build. Your Dockerfile will look like:

```docker
# syntax=docker/dockerfile:experimental
FROM rust:1.41
...
...
RUN --mount=type=ssh cargo build --all

```

We have 3 important things that need to remember. First, Adding `# syntax=docker/dockerfile:experimental` in the first line of your Dockerfile; this line will notify Docker using the new syntax (you need a new syntax for transferring `ssh`). Second, Adding `--mount=type=ssh` before the line that wants to use the ssh data. Final, to build from Dockerfile, you should add `--ssh default .` in build command and set DOCKER_BUILDKIT env is 1, e.g:

```bash
export DOCKER_BUILDKIT=1
docker build --ssh default . -t rust-base:test

```

You will receive output like this:

```bash
[+] Building 2.3s (10/34)
 => [internal] load build definition from Debug.Base.Dockerfile            0.0s
 => => transferring dockerfile: 48B                                        0.0s
 => [internal] load .dockerignore                                          0.0s
 => => transferring context: 34B                                           0.0s
 => resolve image config for docker.io/docker/dockerfile:experimental      1.3s
 => CACHED docker-image://docker.io/docker/dockerfile:experimental@sha256  0.0s
 => [internal] load metadata for docker.io/library/rust:1.42               0.0s
 => [internal] load metadata for docker.io/library/rust-base:test          0.0s
 => [next-build 1/25] FROM docker.io/library/rust:1.42                     0.0s
...

```

# Caching your build

According to the notice in the introduction, Rust has a downside. That is compiling time. Rust is very slow in the first time builder because of the tradeoff performance, safety between compiling time and running time. Rust needs to check ownership, check safety rules, optimize performance, update index of crates, download dependencies, build your source code. That's. When you use Docker to build Rust, If you don't do anything, you just copy source code and build, you could face the problem that all building steps will re-run. Each build will waste your time up to 30 minutes. It's so expensive in development time.

To solve the problem of the building above, we have one silver bullet - `Cached`. Caching everything that you can. Now, we will analyze the compile steps and what we can cache. You should remember that "Pay Only For What You Use" (C++).

- First, updating the crates index. Basically, Cargo will update the index of crates. You can see it in [here](https://github.com/rust-lang/crates.io-index). This step should be cached. It only updates when your dependencies change.
- Second, downloading dependencies, the same as updating the index.
- Third, compiling dependencies, the same as downloading.
- Forth, compile your source code, cannot cache because you change your code. You need that Rust compiles it.
- Fifth, optimizing code, checking borrowship, checking rules, gen code from macro. All of them are worth. You should pay for them.

So, we will cache the index of crates, dependencies and its binary (the result of building crate). Docker provides a cache mechanism that is: the current layer will be cached if the previous layer is cached and nothing changes in this current layer; very simple and straightforward. In this case, It seems hard to use the Docker cache mechanism because all of the actions that are included in a command `cargo build --all`. If we add, remove, or change dependencies, Docker will not cache. We need another mechanism to solve this problem. One solution is the cache mechanism of Cargo and the multi-stage build of Docker.

### - Why caching mechanism of Cargo

Rust team knows our situation. They know the compiling time is a downside of Rust Programming Language. They provide a cache system to help us. You only compile your change in the Rust project. It caches updating index, it cache source code of dependencies and its binary; there are all things that we need to cache. Cargo is convenient, useful, and automatic. Why not use caching mechanism of Cargo?

### - Why multi-stage

The issue of using cache mechanism of Cargo in Docker is when you build the new image:

- If you use `Rust:1.41` image from (Rust official in Docker hub), you will have a new image in each building. The image is don't have anything that we need to cache as expected from the previous build.
- You can use recursive tagged Docker Image (e.g You tag Rust:1.41 is rust-base:1.41, after completed docker build, you tag the new image is rust-base:1.41 and it re-do each time when you build the docker image). If you use this way, you will be facing another problem of Docker mechanism that is maximum layers of Docker (Read [here](https://github.com/docker/docker.github.io/issues/8230)). With multi-stage, you can avoid the problem.

As discussed above, we will have a Dockerfile look like:

```docker
# syntax=docker/dockerfile:experimental

FROM rust-base:test as pre-build
WORKDIR /app  # Assume that you are using /app as a location of source code

################################
# You will create new image here, so layer will re-count from 0
FROM rust:1.42 as current-build

WORKDIR /app

# This command will help us avoid re-build dependencies
COPY --from=pre-build /app /app

# This command will help us avoid re-udpate and re-download dependencies
COPY --from=pre-build /usr/local/cargo /usr/local/cargo

ADD . /app

RUN --mount=type=ssh cargo build --all

```

The steps should be:

1. Just do this once. This command will help us prepare first image base from official image of Rust.

```docker
docker tag rust:1.42 rust-base:1.42

```

1. Run it and recursive tag.

```docker
export DOCKER_BUILDKIT=1
docker build --ssh default . -t rust-base:1.42

```

# Takeaways:

- Use Docker build kit and ssh-agent to use private dependency in Cargo.
- Use the default mechanism of Cargo.
- Use multi-stage of Docker to recursive tag. It will avoid the layers limited to Docker.

This is my [example project](https://github.com/cptrodgers/docker-rust-example).