# AG35 SDK Docker 镜像说明

## 概述

本项目用于构建统一的 AG35 SDK 开发镜像（基础镜像：`ubuntu:20.04`），覆盖以下场景：

- **Host（x86_64）构建/测试**：CMake 3.30.8、Ninja 1.11、`/opt/googletest`、`lcov-2.3.1`、`cppcheck-2.13.0`、`clang-format-18`、`clang-tidy-18`
- **AG35 交叉构建**：Quectel 工具链和扩展 SDK 在构建阶段从 `vendor/` 离线包解压到 `/opt/ql_crosstools/`
- **rootfs/kernel 构建依赖**：镜像内预装 `checkpolicy`、`liblzo2-dev` 等常用依赖

镜像默认工作目录为 `/workspace`。

当前镜像版本信息：
```
AG35SDK tag:        1.0
Docker image:       ag35sdk:1.0
Image ID (sha256):  sha256:749ba31893849d1796c56b7d1ff56eb9997b16f063759321a43f0334bb2acf79
Image created:      2026-04-22T17:32:12.384766631+08:00
```

## 构建镜像

### 1) 准备离线依赖包

将以下文件放在工程根目录的 `vendor/` 下（文件名需与 `Dockerfile` 一致）：

- [cmake-3.30.8-linux-x86_64.sh](https://github.com/Kitware/CMake/releases/download/v3.30.8/cmake-3.30.8-linux-x86_64.sh)
- [ninja-linux.zip](https://github.com/ninja-build/ninja/releases/download/v1.11.1/ninja-linux.zip)
- [googletest](https://github.com/google/googletest.git)
- [cppcheck-2.13.0.tar.gz](https://github.com/danmar/cppcheck/archive/2.13.0.tar.gz)
- [lcov-2.3.1.tar.gz](https://github.com/linux-test-project/lcov/releases/download/v2.3.1/lcov-2.3.1.tar.gz)
- `ql-ag35-le22-gcc640-v1-toolchain.tar.gz`
- `ql-ol-extsdk-ag35envgmr11a06m4g_ocpu.tar.gz`

### 2) 构建镜像

```bash
docker build -t ag35sdk:1.0 .
```

### 3) 导出镜像（可选）

```bash
docker save ag35sdk:1.0 | gzip > ag35sdk-1.0.tar.gz
```

## 加载已导出的镜像

```bash
curl -fSL "http://192.168.24.55:8081/docker_images/ag35sdk-1.0.tar.gz" | gunzip | docker load
```

## 运行示例

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

