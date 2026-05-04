# AG35 SDK Docker image

## Overview

This directory builds a single **AG35 SDK** development image (`ubuntu:20.04`) for:

- **Host (x86_64) build/test**: CMake 3.30.8, Ninja 1.11, `/opt/googletest`, lcov 2.3.1, cppcheck 2.13.0, clang-format-18, clang-tidy-18
- **AG35 cross builds**: Quectel toolchain and extended SDK unpacked from `vendor/` at build time into `/opt/ql_crosstools/`
- **rootfs/kernel-style deps**: packages such as `checkpolicy`, `liblzo2-dev`, etc.

Default working directory inside the image is `/workspace`.

## Prerequisites (Linux)

The image is often loaded from the devops file site **`https://files.devops.local`**. On the machine that runs `curl` or `docker build`:

1. Resolve **`files.devops.local`** (and other `*.devops.local` names if needed) to the server that serves HTTPS on 443, e.g. append to **`/etc/hosts`**:

```bash
echo "<SERVER_IP> git.devops.local registry.devops.local cloud.devops.local files.devops.local" | sudo tee -a /etc/hosts
```

2. That host uses **Caddy `tls internal`**. Examples below use **`curl -k`** to skip TLS verification—acceptable only on a **trusted LAN**; do not rely on `-k` on the public internet.

## Build the image

### 1) Vendor bundles

Place these under `vendor/` at the **repository root** (names must match the `Dockerfile`):

- [cmake-3.30.8-linux-x86_64.sh](https://github.com/Kitware/CMake/releases/download/v3.30.8/cmake-3.30.8-linux-x86_64.sh)
- [ninja-linux.zip](https://github.com/ninja-build/ninja/releases/download/v1.11.1/ninja-linux.zip)
- [googletest](https://github.com/google/googletest.git)
- [cppcheck-2.13.0.tar.gz](https://github.com/danmar/cppcheck/archive/2.13.0.tar.gz)
- [lcov-2.3.1.tar.gz](https://github.com/linux-test-project/lcov/releases/download/v2.3.1/lcov-2.3.1.tar.gz)
- `ql-ag35-le22-gcc640-v1-toolchain.tar.gz`
- `ql-ol-extsdk-ag35envgmr11a06m4g_ocpu.tar.gz`

### 2) Build

```bash
docker build -t ag35sdk:latest .
```

### 3) Export (optional)

```bash
docker save ag35sdk:latest | gzip > ag35sdk-latest.tar.gz
```

Publish the tarball under the devops file tree so it is visible at a stable URL, e.g. **`/opt/data/files/srv/docker_images/`** on the host maps to **`https://files.devops.local/docker_images/...`**.

## Load a published image

```bash
curl -kfSL "https://files.devops.local/docker_images/ag35sdk-1.0.tar.gz" | gunzip | docker load
```

Adjust the path/filename to match what you published under `/opt/data/files/srv/`.

## Run example

```bash
docker run --rm -it \
  --user "$(id -u):$(id -g)" \
  -v /etc/passwd:/etc/passwd:ro \
  -v /etc/group:/etc/group:ro \
  -v "$PWD:/workspace" \
  -w /workspace \
  ag35sdk:1.0 \
  cmake --preset host-debug --fresh
```

Stack layout and hosts for **`*.devops.local`**: see the main [README.md](../README.md) in the devops repository.
