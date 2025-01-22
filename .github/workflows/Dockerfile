FROM ubuntu:latest

ARG TARGET
ARG MAC_SDK_VERSION
ARG MACOSX_PPC_DEPLOYMENT_TARGET

ENV ROOT_DIR=/home
ENV TARGET_PREFIX=/usr/local

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
    && apt-get autoremove -y --purge \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# dsymutil

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

# cctools

RUN mkdir -p $ROOT_DIR/cctools-port && \
    cd $ROOT_DIR/cctools-port && \
    git init && \
    git remote add origin https://github.com/Wohlstand/cctools-port.git && \
    git fetch --depth 1 origin f0ac160455ce815def93194a298c07d78e81f343 && \
    git checkout FETCH_HEAD

RUN mkdir -p $ROOT_DIR/build/target/cctools-port && \
    cd $ROOT_DIR/build/target/cctools-port && \
    $ROOT_DIR/cctools-port/cctools/configure CC=clang CXX=clang++ --prefix="$TARGET_PREFIX" --target=$TARGET && \
    make -j && \
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

RUN cd $TARGET_PREFIX/$MAC_SDK_VERSION.sdk/usr/lib && \
    for i in **/*.dylib **/*.a **/*.o; do \
      if file "$i" | grep -q "for architecture ppc)"; then \
        mv $i $i.universal && \
        $TARGET_PREFIX/bin/$TARGET-lipo -thin ppc $i.universal -output $i; \
      elif file "$i" | grep -q "for architecture ppc7400)"; then \
        mv $i $i.universal && \
        $TARGET_PREFIX/bin/$TARGET-lipo -thin ppc7400 $i.universal -output $i; \
      fi \
    done

# GCC & Apple's CRT

RUN mkdir -p $ROOT_DIR/gcc && \
    cd $ROOT_DIR/gcc && \
    git init && \
    git remote add origin https://github.com/VariantXYZ/gcc-13-branch.git && \
    git fetch --depth 1 origin 908dbc96f1271f995759c87fc9d32879d6f49756 && \
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
    $ROOT_DIR/gcc/configure CC=clang CXX=clang++ --disable-nls --disable-multilib --enable-languages=c,c++,lto --with-dwarf2 --prefix="$TARGET_PREFIX" --target=$TARGET --with-sysroot="$TARGET_PREFIX/$MAC_SDK_VERSION.sdk" CXXFLAGS_FOR_TARGET="-O2 -g -mmacosx-version-min=$MACOSX_PPC_DEPLOYMENT_TARGET" CFLAGS_FOR_TARGET="-O2 -g -mmacosx-version-min=$MACOSX_PPC_DEPLOYMENT_TARGET" LDFLAGS_FOR_TARGET="-O2 -g -mmacosx-version-min=$MACOSX_PPC_DEPLOYMENT_TARGET" --disable-bootstrap && \
    make all-gcc -j && \
    make install-gcc -j

## Rebuilding Apple's crt1.o

RUN cd $ROOT_DIR/csu && \
    make CC=$TARGET_PREFIX/bin/powerpc-apple-darwin8-gcc ARCH_CFLAGS="-arch ppc -D__private_extern__= -isysroot $TARGET_PREFIX/$MAC_SDK_VERSION.sdk" ./crt1.v1.o && \
    mv ./crt1.v1.o $TARGET_PREFIX/$MAC_SDK_VERSION.sdk/usr/lib/crt1.o && \
    cd $ROOT_DIR && \
    rm -rf $ROOT_DIR/csu

## Build the rest of GCC

RUN cd $ROOT_DIR/build/target/gcc && \
	make -j && \
	make install -j && \
	cd $ROOT_DIR && \
	rm -rf $ROOT_DIR/build/target/gcc && \
	rm -rf $ROOT_DIR/gcc






