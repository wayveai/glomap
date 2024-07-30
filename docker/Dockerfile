ARG UBUNTU_VERSION=22.04
ARG NVIDIA_CUDA_VERSION=12.3.1

#
# Docker builder stage.
#
FROM nvidia/cuda:${NVIDIA_CUDA_VERSION}-devel-ubuntu${UBUNTU_VERSION} as builder

ARG GLOMAP_GIT_COMMIT=main
ARG CUDA_ARCHITECTURES=native
ENV QT_XCB_GL_INTEGRATION=xcb_egl

# Prevent stop building ubuntu at time zone selection.
ENV DEBIAN_FRONTEND=noninteractive

# Prepare and empty machine for building.
RUN apt-get update && \
    apt-get install -y --no-install-recommends --no-install-suggests \
    git \
    build-essential \
    cmake \
    ninja-build \
    wget \
    unzip \
    libboost-program-options-dev \
    libboost-filesystem-dev \
    libboost-graph-dev \
    libboost-system-dev \
    libeigen3-dev \
    libsuitesparse-dev \
    libceres-dev \
    libflann-dev \
    libfreeimage-dev \
    libmetis-dev \
    libgoogle-glog-dev \
    libgtest-dev \
    libsqlite3-dev \
    libglew-dev \
    qtbase5-dev \
    libqt5opengl5-dev \
    libcgal-dev \
    libcgal-qt5-dev \
    libgl1-mesa-dri \
    libunwind-dev \
    xvfb \
    clang-format-14 \
    python3 \
    python3-pip && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Install newer version of CMake
RUN apt-get install -y wget && \
wget https://github.com/Kitware/CMake/releases/download/v3.30.1/cmake-3.30.1-linux-x86_64.sh && \
chmod +x cmake-3.30.1-linux-x86_64.sh && \
./cmake-3.30.1-linux-x86_64.sh --skip-license --prefix=/usr/local

# Install compiler cache tools
RUN mkdir -p /usr/local/compiler-cache/bin && \
    wget https://github.com/ccache/ccache/releases/download/v4.8.2/ccache-4.8.2-linux-x86_64.tar.xz && \
    echo "0b33f39766fe9db67f40418aed6a5b3d7b2f4f7fab025a8213264b77a2d0e1b1  ccache-4.8.2-linux-x86_64.tar.xz" | sha256sum --check && \
    tar xfv ccache-4.8.2-linux-x86_64.tar.xz && \
    mv ccache-4.8.2-linux-x86_64/ccache /usr/local/compiler-cache/bin && \
    rm -rf ccache-4.8.2-linux-x86_64* && \
    ctcache_commit_id="66c3614175fc650591488519333c411b2eac15a3" && \
    wget https://github.com/matus-chochlik/ctcache/archive/${ctcache_commit_id}.zip && \
    echo "108b087f156a9fe7da0c796de1ef73f5855d2a33a27983769ea39061359a40fc  ${ctcache_commit_id}.zip" | sha256sum --check && \
    unzip "${ctcache_commit_id}.zip" && \
    mv ctcache-${ctcache_commit_id}/clang-tidy* /usr/local/compiler-cache/bin && \
    rm -rf ctcache-${ctcache_commit_id}*

# Add the compiler cache to PATH
ENV PATH="/usr/local/compiler-cache/bin:${PATH}"

# Set up compiler environment
RUN apt-get update && \
    apt-get install -y \
    clang-15 \
    libomp-15-dev \
    gcc-10 \
    g++-10 \
    nvidia-cuda-toolkit \
    nvidia-cuda-toolkit-gcc && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

ENV CC=/usr/bin/gcc-10
ENV CXX=/usr/bin/g++-10
ENV CUDAHOSTCXX=/usr/bin/g++-10

# Build and install GLOMAP.
RUN git clone https://github.com/wayveai/glomap.git
RUN cd glomap && \
    git fetch https://github.com/wayveai/glomap.git ${GLOMAP_GIT_COMMIT} && \
    git checkout FETCH_HEAD && \
    mkdir build && \
    cd build && \
    cmake .. \
        -GNinja \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX=/install \
        -DCMAKE_CUDA_ARCHITECTURES=${CUDA_ARCHITECTURES} \
        -DSuiteSparse_CHOLMOD_LIBRARY="/usr/lib/x86_64-linux-gnu/libcholmod.so" \
        -DSuiteSparse_CHOLMOD_INCLUDE_DIR="/usr/include/suitesparse" \
        -DTESTS_ENABLED=ON \
        -DASAN_ENABLED=false && \
    ninja install

# Build and install COLMAP.
RUN git clone https://github.com/colmap/colmap.git
RUN cd colmap && \
    git fetch https://github.com/colmap/colmap.git ${COLMAP_GIT_COMMIT} && \
    git checkout FETCH_HEAD && \
    mkdir build && \
    cd build && \
    cmake .. -GNinja -DCMAKE_CUDA_ARCHITECTURES=${CUDA_ARCHITECTURES} \
        -DCMAKE_INSTALL_PREFIX=/colmap_installed && \
    ninja install
##
## Docker runtime stage.
##
#FROM nvidia/cuda:${NVIDIA_CUDA_VERSION}-runtime-ubuntu${UBUNTU_VERSION} as runtime
#
## Minimal dependencies to run GLOMAP binary compiled in the builder stage.
## Note: this reduces the size of the final image considerably, since all the
## build dependencies are not needed.
#RUN apt-get update && \
#    apt-get install -y --no-install-recommends --no-install-suggests \
#        libboost-filesystem1.74.0 \
#        libboost-program-options1.74.0 \
#        libc6 \
#        libceres2 \
#        libfreeimage3 \
#        libgcc-s1 \
#        libgl1 \
#        libglew2.2 \
#        libgoogle-glog0v5 \
#        libqt5core5a \
#        libqt5gui5 \
#        libqt5widgets5
#
## Copy all files from /glomap_installed/ in the builder stage to /usr/local/ in
## the runtime stage. This simulates installing COLMAP in the default location
## (/usr/local/), which simplifies environment variables. It also allows the user
## of this Docker image to use it as a base image for compiling against COLMAP as
## a library. For instance, CMake will be able to find COLMAP easily with the
## command: find_package(COLMAP REQUIRED).

RUN cp -r /install/* /usr/local/
RUN cp -r /colmap_installed/* /usr/local/
#COPY /install/ /usr/local/