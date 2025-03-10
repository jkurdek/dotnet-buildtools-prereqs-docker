ARG ROOTFS_DIR=/crossrootfs/x64

FROM mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net10.0-crossdeps-builder-amd64 AS builder
ARG ROOTFS_DIR

# Use Ubuntu Bionic as the base image for the rootfs
# to get a new enough libstdc++ for the sanitizer runtimes and instrumentation.
RUN /scripts/eng/common/cross/build-rootfs.sh x64 bionic --skipunmount

RUN TARGET_TRIPLE="x86_64-linux-gnu" && \
    CLANG_MAJOR_VERSION=$(clang --version | grep -oP "(?<=version )\d+") && \
    cmake -S llvm-project.src/runtimes -B runtimes \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_ASM_COMPILER=clang \
        -DCMAKE_C_COMPILER=clang \
        -DCMAKE_CXX_COMPILER=clang++ \
        -DCMAKE_ASM_COMPILER_TARGET="$TARGET_TRIPLE" \
        -DCMAKE_C_COMPILER_TARGET="$TARGET_TRIPLE" \
        -DCMAKE_CXX_COMPILER_TARGET="$TARGET_TRIPLE" \
        # We're going to link the static libraries we build here into PIC images, so build the static libraries as PIC.
        -DCMAKE_POSITION_INDEPENDENT_CODE=ON \
        -DCMAKE_SYSROOT="$ROOTFS_DIR" \
        # Specify linker to use for exes directly for CMake toolchain detection
        -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=lld" \
        # Don't search for tools in the sysroot as we're cross-compiling
        -DCMAKE_FIND_ROOT_PATH_MODE_PROGRAM="NEVER" \
        -DLLVM_USE_LINKER=lld \
        -DLLVM_ENABLE_RUNTIMES="compiler-rt;libcxx" \
        -DLIBCXX_ENABLE_SHARED=OFF \
        -DLIBCXX_CXX_ABI="libstdc++" \
        -DLIBCXX_CXX_ABI_INCLUDE_PATHS="$ROOTFS_DIR/usr/include/c++/7.5.0/;$ROOTFS_DIR/usr/include/$TARGET_TRIPLE" \
        -DCOMPILER_RT_CXX_LIBRARY="libcxx" \
        -DCOMPILER_RT_STATIC_CXX_LIBRARY=ON \
        -DSANITIZER_CXX_ABI_LIBNAME="libstdc++" \ 
        -DCOMPILER_RT_BUILD_MEMPROF=OFF \
        # The libfuzzer build in LLVM doesn't correctly forward the required CMake properties to the "fuzzed libc++" build
        # so skip it here.
        -DCOMPILER_RT_BUILD_LIBFUZZER=OFF \
        -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON \
        # Work around error building rtsan (this enables all sanitizers but rtsan)
        -DCOMPILER_RT_SANITIZERS_TO_BUILD="asan;dfsan;msan;hwasan;tsan;safestack;cfi;scudo_standalone;ubsan_minimal;gwp_asan;nsan;asan_abi" \
        -DCOMPILER_RT_INSTALL_PATH="/usr/local/lib/clang/$CLANG_MAJOR_VERSION" && \
    cmake --build runtimes -j && \
    cmake --install runtimes --prefix "$ROOTFS_DIR/usr"

FROM mcr.microsoft.com/dotnet-buildtools/prereqs:azurelinux-3.0-net10.0-crossdeps-llvm-amd64
ARG ROOTFS_DIR

COPY --from=builder /usr/local/lib/clang /usr/local/lib/clang
COPY --from=builder $ROOTFS_DIR $ROOTFS_DIR
