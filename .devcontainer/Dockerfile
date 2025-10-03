FROM registry.access.redhat.com/ubi9/ubi:latest

# Set working directory to workspace
WORKDIR /workspace

# Enable EPEL repository for additional packages
RUN dnf install -y \
    https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm \
    && dnf clean all

# Install development dependencies and tools using dnf (RHEL package manager)
# Note: curl-minimal is already installed in UBI, so we skip curl
RUN dnf install -y \
    gcc \
    gcc-c++ \
    make \
    openssl-devel \
    pkg-config \
    wget \
    unzip \
    && dnf clean all

RUN dnf install -y procps-ng selinux-policy-targeted

# Keep container running
CMD ["sleep", "infinity"]