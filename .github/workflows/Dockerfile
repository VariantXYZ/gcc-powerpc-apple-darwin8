FROM ubuntu:latest

ARG TARGET
ARG MAC_SDK_VERSION
ARG MACOSX_PPC_DEPLOYMENT_TARGET

ENV ROOT_DIR=/home
ENV TARGET_PREFIX=/usr/local
ENV LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/local/lib"

RUN mkdir -p $ROOT_DIR

# Pre-requisites
RUN apt-get update -q && \
	apt-get install -y \
	cmake \
	ninja-build \
	build-essential \
	clang \
	llvm-dev \
	git \
	python3 \
	wget \
	flex \
	texinfo \
    file \
    autoconf \
    libssl-dev \
    libz-dev \
    libtool-bin \
    && apt-get autoremove -y --purge \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# libxar on host

RUN mkdir -p $ROOT_DIR/xar && \
    cd $ROOT_DIR/xar && \
    git init && \
    git remote add origin https://github.com/tpoechtrager/xar.git && \
    git fetch --depth 1 origin 5fa4675419cfec60ac19a9c7f7c2d0e7c831a497 && \
    git checkout FETCH_HEAD

RUN mkdir -p $ROOT_DIR/build/host/xar && \
    cd $ROOT_DIR/build/host/xar && \
    CC=clang CXX=clang++ $ROOT_DIR/xar/xar/configure && \
    make -j2 && \
    make -j install

# cctools

## dsymutil

RUN mkdir -p $ROOT_DIR/llvm-project && \
    cd $ROOT_DIR/llvm-project && \
    git init && \
    git remote add origin https://github.com/llvm/llvm-project.git && \
    git fetch --depth 1 origin 4856a9330ee01d30e9e11b6c2f991662b4c04b07 && \
    git checkout FETCH_HEAD

RUN mkdir -p $ROOT_DIR/build/target/llvm-project && \
    cd $ROOT_DIR/build/target/llvm-project && \
    CC=clang CXX=clang++ cmake -G Ninja $ROOT_DIR/llvm-project/llvm \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_TARGETS_TO_BUILD="PowerPC" \
      -DLLVM_ENABLE_ASSERTIONS=OFF && \
    ninja dsymutil && \
    mkdir -p $TARGET_PREFIX/bin && \
    cp bin/dsymutil $TARGET_PREFIX/bin/$TARGET-dsymutil && \
    cd $ROOT_DIR && \
    rm -rf $ROOT_DIR/build/target/llvm-project && \
    rm -rf $ROOT_DIR/llvm-project

## ld, as, ar, lipo, nm, ranlib

RUN mkdir -p $ROOT_DIR/cctools-port && \
    cd $ROOT_DIR/cctools-port && \
    git init && \
    git remote add origin https://github.com/tpoechtrager/cctools-port.git && \
    git fetch --depth 1 origin 6694f27d56923e64e6190c8d3eb149413768e9b7 && \
    git checkout FETCH_HEAD && \
    cd cctools && \
    ./autogen.sh

## Symlink for convenience

RUN mkdir -p $ROOT_DIR/build/target/cctools-port && \
    cd $ROOT_DIR/build/target/cctools-port && \
    $ROOT_DIR/cctools-port/cctools/configure CC=clang CXX=clang++ --prefix="$TARGET_PREFIX" --target=$TARGET && \
    make -j2 && \
    make install -j && \
    mkdir -p $TARGET_PREFIX/$TARGET/bin && \
    for i in ld ar as lipo nm ranlib strip dsymutil; do \
      ln -s "$TARGET_PREFIX/bin/$TARGET-$i" "$TARGET_PREFIX/$TARGET/bin/$i"; \
    done && \
    cd $ROOT_DIR && \
    rm -rf $ROOT_DIR/build/target/cctools-port && \
    rm -rf $ROOT_DIR/cctools-port

# MacOSX SDK
RUN wget https://github.com/phracker/MacOSX-SDKs/releases/download/11.3/$MAC_SDK_VERSION.sdk.tar.xz && \
    tar xf ./$MAC_SDK_VERSION.sdk.tar.xz -C $TARGET_PREFIX && \
    rm $MAC_SDK_VERSION.sdk.tar.xz

# Force only the ppc binaries to exist
RUN cd $TARGET_PREFIX/$MAC_SDK_VERSION.sdk/usr/lib && \
    for i in $(find . -type f \( -name "*.dylib" -o -name "*.a" -o -name "*.o" \)); do \
      if file "$i" | grep -q "ppc:"; then \
        mv $i $i.universal && \
        $TARGET_PREFIX/bin/$TARGET-lipo -thin ppc $i.universal -output $i; \
      elif file "$i" | grep -q "\ppc_7400:"; then \
        mv $i $i.universal && \
        $TARGET_PREFIX/bin/$TARGET-lipo -thin ppc7400 $i.universal -output $i; \
      fi \
    done

# GCC & Apple's CRT

RUN mkdir -p $ROOT_DIR/gcc && \
    cd $ROOT_DIR/gcc && \
    git init && \
    git remote add origin https://github.com/VariantXYZ/gcc-14-branch.git && \
    git fetch --depth 1 origin 67aa1b48dd9a0651a2c2e3b4a1bf6d12e2363fb8 && \
    git checkout FETCH_HEAD && \
    ./contrib/download_prerequisites

RUN mkdir -p $ROOT_DIR/csu && \
    cd $ROOT_DIR/csu && \
    git init && \
    git remote add origin https://github.com/VariantXYZ/Csu.git && \
    git fetch --depth 1 origin 06967ab403a7a04e30e2f32479285d68152cedd2 && \
    git checkout FETCH_HEAD

## Compiler-only, note we use clang to build it because gcc itself will cause the docker container to abort with OOM...

RUN mkdir -p $ROOT_DIR/build/target/gcc && \
    cd $ROOT_DIR/build/target/gcc && \
    echo "" > config.site-$TARGET && \
    echo "gcc_cv_as_mmacosx_version_min=no" >> config.site-$TARGET && \
    echo "gcc_cv_as_darwin_build_version=no" >> config.site-$TARGET && \
    echo "gcc_cv_ld64_demangle=0" >> config.site-$TARGET && \
    echo "gcc_cv_ld64_platform_version=0" >> config.site-$TARGET && \
    CONFIG_SITE=$ROOT_DIR/build/target/gcc/config.site-$TARGET $ROOT_DIR/gcc/configure CC=clang CXX=clang++ --disable-nls --disable-multilib --enable-languages=c,c++,lto --with-dwarf2 --prefix="$TARGET_PREFIX" --target=$TARGET --with-sysroot="$TARGET_PREFIX/$MAC_SDK_VERSION.sdk" CXXFLAGS_FOR_TARGET="-O2 -g -mmacosx-version-min=$MACOSX_PPC_DEPLOYMENT_TARGET" CFLAGS_FOR_TARGET="-O2 -g -mmacosx-version-min=$MACOSX_PPC_DEPLOYMENT_TARGET" LDFLAGS_FOR_TARGET="-O2 -g -mmacosx-version-min=$MACOSX_PPC_DEPLOYMENT_TARGET" --disable-bootstrap && \
    make all-gcc -j2 && \
    make install-gcc -j

## Rebuilding Apple's crt1.o
## This should also allow for static linking: https://developer.apple.com/library/archive/qa/qa1118/_index.html
RUN cd $ROOT_DIR/csu && \
    make CC=$TARGET_PREFIX/bin/powerpc-apple-darwin8-gcc ARCH_CFLAGS="-arch ppc -D__private_extern__= -isysroot $TARGET_PREFIX/$MAC_SDK_VERSION.sdk" ./crt1.v1.o && \
    mv ./crt1.v1.o $TARGET_PREFIX/$MAC_SDK_VERSION.sdk/usr/lib/crt1.o && \
    cd $ROOT_DIR && \
    rm -rf $ROOT_DIR/csu

## Build the rest of GCC

RUN cd $ROOT_DIR/build/target/gcc && \
	make -j2 && \
	make install -j && \
	cd $ROOT_DIR && \
	rm -rf $ROOT_DIR/build/target/gcc && \
	rm -rf $ROOT_DIR/gcc






