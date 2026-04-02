# GROMACS Docker image with NVIDIA CUDA support
# Supports both single-precision (gmx) and double-precision (gmx_d)
#
# Build arguments:
#   GROMACS_VERSION  - GROMACS version to build (default: 2025.2)
#   CUDA_VERSION     - CUDA base image version (default: 12.8.1)
#   CUDNN_VERSION    - cuDNN variant (default: cudnn)
#   UBUNTU_VERSION   - Ubuntu base version (default: ubuntu22.04)
#
# Example build:
#   docker build --build-arg GROMACS_VERSION=2025.2 -t gromacs:2025.2-cuda12.8.1 .

ARG CUDA_VERSION=12.8.1
ARG CUDNN_VERSION=cudnn
ARG UBUNTU_VERSION=ubuntu22.04

FROM nvidia/cuda:${CUDA_VERSION}-${CUDNN_VERSION}-devel-${UBUNTU_VERSION}

ARG GROMACS_VERSION=2025.2

LABEL org.opencontainers.image.title="GROMACS with CUDA"
LABEL org.opencontainers.image.description="GROMACS molecular dynamics simulation software with NVIDIA CUDA GPU acceleration. Includes both single-precision (gmx) and double-precision (gmx_d) builds."
LABEL org.opencontainers.image.source="https://github.com/JinZhangLab/gmxPyTools"
LABEL org.opencontainers.image.licenses="LGPL-2.1"
LABEL gromacs.version="${GROMACS_VERSION}"
LABEL cuda.version="${CUDA_VERSION}"

ENV DEBIAN_FRONTEND=noninteractive
ENV OMPI_ALLOW_RUN_AS_ROOT=1
ENV OMPI_ALLOW_RUN_AS_ROOT_CONFIRM=1

# Install build dependencies and CMake from Kitware PPA (required for GROMACS 2024+)
RUN apt-get update && \
    apt-get install -y \
        software-properties-common \
        apt-transport-https \
        ca-certificates \
        gnupg \
        wget \
        python3 \
        python3-pip \
        make \
        gcc \
        g++ \
        libfftw3-dev \
        libopenmpi-dev && \
    wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc \
        | gpg --dearmor -o /usr/share/keyrings/kitware-archive-keyring.gpg && \
    echo 'deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ jammy main' \
        | tee /etc/apt/sources.list.d/kitware.list > /dev/null && \
    apt-get update && \
    apt-get install -y cmake && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /gromacs

# Download and unpack GROMACS source
RUN wget -q "http://ftp.gromacs.org/pub/gromacs/gromacs-${GROMACS_VERSION}.tar.gz" \
    && tar -xzf "gromacs-${GROMACS_VERSION}.tar.gz" \
    && rm "gromacs-${GROMACS_VERSION}.tar.gz"

# Build single-precision version (gmx) with GPU support
RUN mkdir build_single && cd build_single && \
    cmake "../gromacs-${GROMACS_VERSION}" \
        -DGMX_BUILD_OWN_FFTW=ON \
        -DREGRESSIONTEST_DOWNLOAD=ON \
        -DGMX_GPU=CUDA \
        -DGMX_DOUBLE=OFF \
        -DCMAKE_BUILD_TYPE=Release && \
    make -j "$(nproc)" && \
    make check && \
    make install && \
    cd / && rm -rf /gromacs/build_single

# Build double-precision version (gmx_d) — CPU only; CUDA does not support double precision MD
RUN mkdir /gromacs/build_double && cd /gromacs/build_double && \
    cmake "../gromacs-${GROMACS_VERSION}" \
        -DGMX_BUILD_OWN_FFTW=ON \
        -DREGRESSIONTEST_DOWNLOAD=ON \
        -DGMX_GPU=OFF \
        -DGMX_DOUBLE=ON \
        -DCMAKE_BUILD_TYPE=Release && \
    make -j "$(nproc)" && \
    make check && \
    make install && \
    cd / && rm -rf /gromacs/build_double /gromacs/gromacs-${GROMACS_VERSION}

# Source GROMACS environment in all new shells
RUN echo 'source /usr/local/gromacs/bin/GMXRC' >> /etc/bash.bashrc

ENV PATH="/usr/local/gromacs/bin:${PATH}"

WORKDIR /workspace

CMD ["/bin/bash"]
