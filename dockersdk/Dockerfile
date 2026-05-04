FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive \
    TZ=Asia/Shanghai \
    CMAKE_VERSION=3.30.8 \
    CPPCHECK_VERSION=2.13.0 \
    LCOV_VERSION=2.3.1

COPY vendor/ /tmp/vendor/

RUN sed -i 's|http://archive.ubuntu.com/ubuntu|http://mirrors.tuna.tsinghua.edu.cn/ubuntu|g' /etc/apt/sources.list \
    && sed -i 's|http://security.ubuntu.com/ubuntu|http://mirrors.tuna.tsinghua.edu.cn/ubuntu|g' /etc/apt/sources.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    git \
    build-essential \
    python3 \
    xz-utils \
    unzip \
    cpio \
    bc \
    bison \
    flex \
    libncurses-dev \
    file \
    pkg-config \
    perl \
    libdatetime-perl \
    libtimedate-perl \
    libcapture-tiny-perl \
    checkpolicy \
    liblzo2-dev \
    clang-tidy-18 \
    clang-format-18 \
    && [ -f /usr/bin/clang-format-18 ] && ln -sf /usr/bin/clang-format-18 /usr/bin/clang-format \
    && [ -f /usr/bin/clang-tidy-18 ] && ln -sf /usr/bin/clang-tidy-18 /usr/bin/clang-tidy \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && ln -sf /bin/bash /bin/sh \
    && sh /tmp/vendor/cmake-${CMAKE_VERSION}-linux-x86_64.sh --skip-license --prefix=/usr/local \
    && unzip -q /tmp/vendor/ninja-linux.zip -d /usr/local/bin \
    && chmod +x /usr/local/bin/ninja \
    && tar -xzf /tmp/vendor/cppcheck-${CPPCHECK_VERSION}.tar.gz -C /tmp \
    && make -C /tmp/cppcheck-${CPPCHECK_VERSION} -j"$(nproc)" \
    && make -C /tmp/cppcheck-${CPPCHECK_VERSION} install FILESDIR=/usr/local/bin PREFIX=/usr/local \
    && tar -xzf /tmp/vendor/lcov-${LCOV_VERSION}.tar.gz -C /tmp \
    && make -C /tmp/lcov-${LCOV_VERSION} install PREFIX=/usr/local \
    && mkdir -p /opt/googletest \
    && cmake -S /tmp/vendor/googletest -B /tmp/gt -G Ninja \
       -DCMAKE_INSTALL_PREFIX=/opt/googletest \
       -DINSTALL_GTEST=ON \
       -DBUILD_GMOCK=ON \
       -DCMAKE_BUILD_TYPE=Release \
    && cmake --build /tmp/gt -j"$(nproc)" \
    && cmake --install /tmp/gt \
    && mkdir -p /opt/ql_crosstools /workspace \
    && tar -xzf /tmp/vendor/ql-ag35-le22-gcc640-v1-toolchain.tar.gz -C /opt/ql_crosstools \
    && tar -xzf /tmp/vendor/ql-ol-extsdk-ag35envgmr11a06m4g_ocpu.tar.gz -C /opt/ql_crosstools \
    && rm -rf /tmp/* /var/tmp/*

WORKDIR /workspace

CMD ["/bin/bash"]
