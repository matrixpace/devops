FROM alpinelinux/docker-cli:latest
# CI/release helpers: bash, curl (GitLab API), python3 (JSON), git, gzip
RUN apk add --no-cache \
    bash \
    curl \
    python3 \
    git \
    gzip
# Use bash as default shell for CI scripts.
SHELL ["/bin/bash", "-lc"]
CMD ["/bin/bash"]
