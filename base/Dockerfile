# Stage 1: Clone llvm-mos
FROM debian:bookworm-slim as llvm-clone

# Install Git
RUN apt-get update && \
    apt-get install -y \
        git && \
    rm -rf /var/lib/apt/lists/*

# Clone llvm-mos repository at the specific commit
WORKDIR /
RUN git clone https://github.com/mrk-its/llvm-mos

# Stage 2: Build llvm-mos
FROM llvm-clone as llvm-builder

# Install build dependencies for llvm-mos
RUN apt-get update && \
    apt-get install -y \
        cmake \
        clang \
        ninja-build \
        libxml2-dev && \
    rm -rf /var/lib/apt/lists/*

# Build llvm-mos with the appropriate configuration
WORKDIR /llvm-mos/
RUN echo "DO NOT RUN ME WITHOUT SETTING -DLIBXML2_LIBRARY" && \
    cmake -C clang/cmake/caches/MOS.cmake -G "Ninja" -S llvm -B build \
       -DLLVM_INSTALL_TOOLCHAIN_ONLY=OFF \
       -DLLVM_BUILD_LLVM_DYLIB=ON -DLLVM_LINK_LLVM_DYLIB=ON \
       -DLLVM_INSTALL_UTILS=ON -DLLVM_BUILD_UTILS=ON -DLLVM_TOOLCHAIN_UTILITIES=FileCheck \
       -DLLVM_TOOLCHAIN_TOOLS="llvm-addr2line;llvm-ar;llvm-cxxfilt;llvm-dwarfdump;llvm-mc;llvm-nm;llvm-objcopy;llvm-objdump;llvm-ranlib;llvm-readelf;llvm-readobj;llvm-size;llvm-strings;llvm-strip;llvm-symbolizer;llvm-config;llc" \
       -DLIBXML2_LIBRARY=/usr/lib/x86_64-linux-gnu/libxml2.so \
       -DLLVM_TARGETS_TO_BUILD="MOS;X86" \
       -DLLVM_ENABLE_PROJECTS="clang;lld;lldb"
RUN cmake --build build -t install


# Stage 3: Clone rust-mos
FROM llvm-builder as rust-clone

# Install Git (if not already installed)
RUN apt-get update && \
    apt-get install -y \
        git && \
    rm -rf /var/lib/apt/lists/*

# Clone the rust-mos repository
WORKDIR /
RUN git clone https://github.com/mrk-its/rust-mos

# Stage 4: Build rust-mos
FROM rust-clone as rust-builder

# Install Python (if not already installed)
RUN apt-get update && \
    apt-get install -y \
        python3 curl pkg-config libssl-dev && \
    rm -rf /var/lib/apt/lists/*

RUN curl https://sh.rustup.rs -sSf | sh -s -- -y

ENV PATH="/root/.cargo/bin:${PATH}"

# Build the rust-mos toolchain
WORKDIR /rust-mos/
COPY config.toml /rust-mos/config.toml

RUN export RUST_TARGET_PATH=/usr/local/rust-mos-targets/ && \
    ./x.py build -i --stage 0 src/tools/cargo && \
    ./x.py build -i && \
    ln -s ../../stage0-tools-bin/cargo build/x86_64-unknown-linux-gnu/stage1/bin/cargo && \
    rustup toolchain link mos build/x86_64-unknown-linux-gnu/stage1 && \
    rustup default mos && \
    mkdir -p $RUST_TARGET_PATH && \
    python3 create_mos_targets.py $RUST_TARGET_PATH

# Stage 5: Final image
FROM rust-builder as rust-mos

WORKDIR /

RUN rm -rf /llvm-mos

# Create a script to remove directories
RUN echo '#!/bin/bash' > /cleanup.sh && \
    echo 'set -e' >> /cleanup.sh && \
    echo 'find /rust-mos/build/x86_64-unknown-linux-gnu/ -type d \( -name "deps" -o -name "incremental" \) -exec rm -rf {} +' >> /cleanup.sh && \
    chmod +x /cleanup.sh

# Run the script to remove directories
RUN /cleanup.sh

