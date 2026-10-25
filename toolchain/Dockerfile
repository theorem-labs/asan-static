# syntax=docker/dockerfile:1.7
#
# Reproducible musl + compiler-rt(ASan) toolchain, patched so that
# AddressSanitizer works in a fully static link.
#
# One architecture per image, cross-built.  The image runs on the machine doing the
# build and compiles for whatever the three arguments below name, so the target is
# a property of the arguments and not of the base image.  Where the two agree the
# build is the native one it always was.  Build (from the repo root):
#   docker build \
#     -f toolchain/Dockerfile \
#     --build-arg TRIPLE=aarch64-unknown-linux-musl \
#     --build-arg MUSL_TARGET=aarch64-linux-musl \
#     --build-arg APK_ARCH=aarch64 \
#     --build-context musl-src=$PWD/musl \
#     --build-context llvm-src=$PWD/llvm-project \
#     --build-context patches=$PWD/patches \
#     -t tc-asan-musl toolchain
#
# The three name the same architecture three ways, because three things spell it
# differently: TRIPLE is what clang is told, MUSL_TARGET is what musl's configure
# selects its arch directory by, and APK_ARCH is what Alpine calls it.  They
# default to aarch64.  For x86_64 they are x86_64-unknown-linux-musl,
# x86_64-linux-musl and x86_64; for riscv64, riscv64-unknown-linux-musl,
# riscv64-linux-musl and riscv64; for 32-bit Arm,
# armv7-unknown-linux-musleabihf, armv7-linux-musleabihf and armv7.  No --platform
# is needed and passing one does nothing here: the base image is pinned to
# $BUILDPLATFORM.  --build-arg TOOLCHAIN_PLATFORM=<platform> overrides that, which
# is how the emulated build this replaced is still reachable -- see
# "Cross-building" in README.md, where the two are compared.
#
# Nothing else in here is written per architecture.  What each architecture needs
# that is not the compiler -- kernel uapi headers, libstdc++ headers, a GNU ld, the
# musl loader, GCC's builtins -- is fetched from Alpine's repository for APK_ARCH in
# section 0a, at the same pinned versions the target's own image installs, so a
# cross-built image reads the same files a native one does.  armv7 is the one
# architecture that changed anything else, in one place: the CAN_TARGET_* settings
# in section 3, which name architectures rather than being chosen per architecture,
# and are explained there.  The one other thing that is not portable is not in this
# file: ASan on riscv64 takes the dynamic-shadow path and its allocator reserves
# 128GB, so the one verification step below that runs an ASan binary needs a
# container whose RLIMIT_AS is not small.  The other two do not: the plain static
# musl link runs and the link that must fail still fails.  A 32-bit target needs
# none of that -- its shadow is about 512MiB.  See the "Running an emulated image"
# section of README.md.
#
# The three steps at the end run binaries for the target.  On a native build that
# is nothing to arrange; on a cross build the machine has to be able to execute
# them, which on Linux means binfmt_misc with qemu-user registered -- `docker run
# --privileged tonistiigi/binfmt --install <arch>` once per boot, or
# docker/setup-qemu-action in CI.  Only these three steps need it.  Nothing the
# image is for does.
#
# Sources are taken from the mounted checkouts at their pinned base commits with
# git archive, unpacked into the image and patched there, so the patch series is
# what is actually built and the host checkouts are never written to (nor is
# whatever state they happen to be in ever used).  The pinned bases are musl f21a9653
# (v1.2.6-20-gf21a9653) and llvm-project 87f0227cb (llvmorg-20.1.8); the patch series in
# patches/ applies to exactly those and the build fails if it does not.
#
# Produces:
#   /opt/musl-sysroot                  musl 1.2.6 sysroot (headers, crt*, libc.a/so)
#                                      + kernel uapi headers + LLVM libunwind.a
#   /opt/asan-rt/lib/clang/20          stock compiler-rt: dlsym interception,
#                                      what a dynamic link uses
#   /opt/asan-static-rt/lib/clang/20   compiler-rt with COMPILER_RT_STATIC_INTERCEPTION:
#                                      the static ASan runtime is attached by the
#                                      linker, plus libclang_rt.asan.a.wrap
#   /opt/asan-static/musl.cfg          clang config: plain static musl
#   /opt/asan-static/asan-musl.cfg     clang config: static musl + ASan
#   /opt/asan-static/asan-musl-dynamic.cfg
#                                      clang config: dynamic ASan, stock runtime
#   /opt/asan-static/asan-musl-nowrap.cfg
#   /opt/asan-static/asan-musl-bfd.cfg  the two ways of getting the static link
#                                      wrong, as configurations, so the suite can
#                                      build them on purpose and require them to
#                                      fail
#   /opt/asan-static/asan-musl-sealed.cfg
#                                      clang config: the same static ASan link
#                                      with sealed init turned on.  Same runtime
#                                      and same wrap list as asan-musl.cfg; the
#                                      whole difference is one --defsym
#   /opt/asan-static/asan-musl-silent.cfg
#                                      clang config: the same static ASan link
#                                      with silent reports turned on: the runtime
#                                      still detects and still dies, and writes
#                                      nothing.  The same one-line difference,
#                                      with a different symbol
#
# Usage:
#   clang --config=/opt/asan-static/asan-musl.cfg prog.c -o prog
#
# The image the compiler runs on, which is not the architecture it compiles for.
# $BUILDPLATFORM is the machine running the build, so the default is a cross build
# with a native compiler: one clang binary targets every architecture, and the two
# stages that are not clang -- musl's configure and compiler-rt's CMake -- are told
# the target rather than allowed to detect it.  Passing
# --build-arg TOOLCHAIN_PLATFORM=linux/riscv64 --platform linux/riscv64 gets the old
# arrangement back, an image of the target's own architecture built under qemu, which
# is what the two are compared against in "Cross-building" in README.md.
ARG TOOLCHAIN_PLATFORM=$BUILDPLATFORM
# The base is pinned by the digest of its multi-platform index -- alpine:3.22
# as published 2026-06-22 (3.22.5) -- rather than by tag, so that the image
# comes out the same from one day to the next: a moved tag moved the host
# packages under an image whose own inputs had not changed (libexpat, liblzma),
# and everything downstream that keys on the image's contents rebuilt.  The
# digest is the index's, so one line pins every platform this file builds.  It
# does not pin what `apk add` fetches below beyond the versions written there:
# a dependency those pull in unnamed still follows the repository.  To move the
# base on purpose, `docker buildx imagetools inspect alpine:3.22` prints the
# current index digest.
FROM --platform=$TOOLCHAIN_PLATFORM alpine:3.22@sha256:14358309a308569c32bdc37e2e0e9694be33a9d99e68afb0f5ff33cc1f695dce

ARG TRIPLE=aarch64-unknown-linux-musl
ARG MUSL_TARGET=aarch64-linux-musl
# Alpine's name for the target architecture -- the third spelling of it, and the
# one its repository and package names use.  It is what selects the target's own
# packages below, and the whole block is declared here rather than after the apk
# steps because those steps read it.
ARG APK_ARCH=aarch64
ARG SYSROOT=/opt/musl-sysroot
ARG RESOURCE_DIR=/opt/asan-rt/lib/clang/20
ARG STATIC_RESOURCE_DIR=/opt/asan-static-rt/lib/clang/20

# Extra target flags threaded through every component that ends up inside a link
# this image makes: musl, libunwind, both compiler-rt builds, and the six config
# files.  Empty by default, so an image built without it is the stock one.
#
# It exists for a target whose ISA subset is part of what the produced binary
# claims, and riscv64 is that target: a caller who wants -march=rv64g cannot get
# it by putting -march on their own command line, because the linker ORs the
# RISC-V attributes of every input.  One rv64gc object anywhere in the static
# link -- libc.a, the builtins, the ASan runtime -- sets EF_RISCV_RVC in the
# output whatever the caller compiled their own code as.  So the flag has to be
# in this image's own build, which is what this is for; threading it into the
# config files as well means a caller who forgets it still gets a consistent
# link.
#
# Cross-building does not make this unnecessary and the reason is worth writing
# down, because it looks as if it should: an x86_64 clang targeting riscv64 never
# sees Alpine's rv64gc musl, so one might expect no compressed instructions to
# get in.  They get in anyway.  The default ISA string for the riscv64 triple is
# LLVM's, not the sysroot's -- `clang --target=riscv64-unknown-linux-musl -dM -E`
# defines __riscv_compressed on an x86_64 host exactly as it does on a riscv64
# one -- so every object this image compiles is rv64gc until told otherwise, and
# e_flags comes out 0x5 on both arrangements.
ARG TARGET_CFLAGS=

# clang20/lld20/llvm20 are all 20.1.8 - the same version as the llvmorg-20.1.8
# checkout we build compiler-rt from, so the resource dir stays consistent.
# ninja-build installs to /usr/lib/ninja-build/bin/ninja, not onto PATH.
# alpine-keys carries the signing key of every architecture Alpine publishes, which
# is what lets the next step fetch the target's packages from this image.
# The pins below are `~series` pins, not exact `=x.y.z-rN` pins. Alpine keeps only the newest build of each
# package in a branch, so an exact pin stops resolving the day Alpine ships an update (measured 2026-10-01:
# python3=3.12.14-r0 gone, 3.12.15-r0 current, `apk add` -> "unable to select packages"). A `~series` pin
# can therefore resolve to a newer build within the series than the one this file was last built with.
#
RUN apk add --no-cache \
        clang20~20.1.8 \
        lld20~20.1.8 \
        llvm20~20.1.8 \
        cmake~3.31 \
        ninja-build~1.12 \
        build-base~0.5 \
        linux-headers~6.14 \
        alpine-keys~2.5 \
        git~2.49 \
        python3~3.12 \
        patch~2.8 \
        file~5.46 \
        docker-cli~28.3 \
 && ln -sf /usr/lib/ninja-build/bin/ninja /usr/bin/ninja

# A GNU ld that targets the target.  Two of the six configurations in section 5 use
# a linker that is not lld, and one of them exists to be a link that must fail: what
# it has to fail on is what the link asks of the linker, so the ld it reaches has to
# be able to target this architecture at all.  It is also what musl's configure
# probes with, since a probe does not see the flags the build is given.
#
# clang looks for it at /usr/<triple>/bin, four directories up from the GCC
# installation it detects, and Alpine's binutils puts one there on the target's own
# image and on no other.  So the cross package goes to the same place, and only when
# it is not there already: adding it unconditionally is an apk file conflict, since
# the two packages own those paths between them.  A native build therefore keeps the
# ld it always had and a cross build gets one that can do the same job.
RUN set -eu; \
    set -- /usr/"$APK_ARCH"-alpine-linux-*/bin/ld.bfd; \
    [ -x "$1" ] || apk add --no-cache binutils-"$APK_ARCH"~2.42

# ---------------------------------------------------------------------------
# 0a. The target's own Alpine packages
#
# Five things this image needs are per-architecture and are not the compiler, so a
# cross build cannot take them from the image it runs in: the kernel uapi headers
# copied into the sysroot in section 1, the libstdc++ headers section 3 puts after
# it, the clang configuration file Alpine installs for its own target, GCC's
# builtins library, and the musl dynamic loader.  Each is fetched from Alpine's
# repository for $APK_ARCH and unpacked -- never installed -- so what lands here is
# the same file the target's own image would have had, at the same pinned version,
# and the native build reads exactly what the cross build reads.
#
# Two of them go to absolute paths rather than under $TARGET_PKGS, because an
# absolute path is what something else looks them up by, and both are put there
# only if they are not there already -- on the target's own image they are, from
# the base image and from build-base, and the whole point is that this is the same
# file rather than a second copy of it.
#
#   /lib/ld-musl-<arch>.so.1     asan-musl-dynamic.cfg's output asks for it by that
#                                name.  Nothing in this image loads it: the name has
#                                the architecture in it and this image's own libc is
#                                a different file.
#   /usr/lib/gcc/<triple>/       musl is the one thing here that links against the
#                                compiler's builtins rather than compiler-rt's,
#                                which do not exist until section 3.  Its configure
#                                asks the compiler for them, the answer is
#                                libgcc.a, and without one the libc.so link ends in
#                                a screenful of undefined __subtf3.  clang finds it
#                                by the target triple, so it does not collide with
#                                this image's own.
#
# The clang config is small and load-bearing.  Alpine ships
# /etc/clang20/<triple>.cfg containing -fstack-clash-protection, and clang loads it
# whenever the target triple is that one.  On the target's own image every `clang`
# invocation gets it; on this one, only the invocations that name the target do --
# and only if the file is here.  musl is the one thing built with the target's
# default configuration rather than an explicit -target (section 1), so without this
# file its objects would quietly differ from the native build's by one hardening
# flag.
#
# That file is also where the fourth spelling of the architecture comes from.
# Alpine's own triple is not composable from APK_ARCH: it is armv7-alpine-linux-
# musleabihf where APK_ARCH is armv7, and the eabihf is not something to guess at.
# The clang20 package contains exactly one such config and its name is the triple,
# so it is read off the file and written to $TARGET_PKGS/alpine-triple for the two
# later steps that need it -- musl's CC and the libstdc++ include directory -- since
# a value computed in one RUN cannot become an ARG for the next.
#
# --keys-dir is needed because the index for another architecture is signed with
# that architecture's key, which lives in a per-arch subdirectory rather than in
# /etc/apk/keys.  A missing arch directory therefore fails here rather than
# silently falling back to this image's own packages.
#
# The versions are pinned by assertion rather than by constraint: `apk fetch
# pkg=1.2-r3` does not resolve at all when --arch names another architecture, so
# each package is fetched by name and the file it produced is then required to be
# the expected upstream version, with any Alpine build number (-rN).  The reason
# is the one the `~series` pins of the apk add above give: Alpine keeps only the
# newest build, so an exact -rN stops resolving the day Alpine rebuilds the
# package (measured 2026-10-25: musl-1.2.5-r12 gone, r13 current).  This check
# is narrower than those pins: a new upstream version, not only a new series,
# stops the build here.
# ---------------------------------------------------------------------------
ARG TARGET_PKGS=/opt/target-pkgs
RUN set -eu; \
    keys=/usr/share/apk/keys/"$APK_ARCH"; \
    [ -d "$keys" ] || { echo "no Alpine signing keys for arch $APK_ARCH in $keys"; exit 1; }; \
    apk="apk --arch $APK_ARCH --keys-dir $keys --no-cache"; \
    mkdir -p "$TARGET_PKGS" /tmp/apks; \
    $apk fetch -o /tmp/apks linux-headers libstdc++-dev clang20 musl gcc; \
    for p in linux-headers-6.14.2 libstdc++-dev-14.2.0 clang20-20.1.8 \
             musl-1.2.5 gcc-14.2.0; do \
      set -- /tmp/apks/"$p"-r*.apk; \
      [ -f "$1" ] || { echo "expected $p-r*.apk for $APK_ARCH, got:"; ls /tmp/apks; exit 1; }; \
    done; \
    tar -C "$TARGET_PKGS" -xzf /tmp/apks/linux-headers-*.apk usr; \
    tar -C "$TARGET_PKGS" -xzf /tmp/apks/libstdc++-dev-*.apk usr; \
    tar -C "$TARGET_PKGS" -xzf /tmp/apks/clang20-*.apk etc; \
    tar -C "$TARGET_PKGS" -xzf /tmp/apks/musl-*.apk lib; \
    tar -C "$TARGET_PKGS" -xzf /tmp/apks/gcc-*.apk usr/lib/gcc; \
    for f in "$TARGET_PKGS"/lib/ld-musl-*.so.1; do \
      [ -e "/lib/${f##*/}" ] || cp -a "$f" /lib/; \
    done; \
    for d in "$TARGET_PKGS"/usr/lib/gcc/*; do \
      [ -e "/usr/lib/gcc/${d##*/}" ] || { mkdir -p /usr/lib/gcc; cp -a "$d" /usr/lib/gcc/; }; \
    done; \
    rm -rf /tmp/apks; \
    set -- "$TARGET_PKGS"/etc/clang20/*.cfg; \
    [ $# -eq 1 ] && [ -f "$1" ] || { echo "expected exactly one clang config for $APK_ARCH, found: $*"; exit 1; }; \
    mkdir -p /etc/clang20; \
    cp "$1" /etc/clang20/; \
    at=${1##*/}; printf '%s' "${at%.cfg}" > "$TARGET_PKGS/alpine-triple"

# ---------------------------------------------------------------------------
# 0. Sources
#
# Copied out of the read-only mounts so they can be patched, to the same paths
# the unpatched build used, so debug info in the runtime still points at
# /llvm-project/compiler-rt/... .  Only the parts of llvm-project the two builds
# below read are copied; .git is left behind (it is 2GB for llvm-project).
#
# Only the musl series is applied here, and only patches/musl is mounted.  The
# compiler-rt series is applied in its own step just before section 3, once
# libunwind and libc++ have been built from the unpatched tree, because the
# layer cache is keyed on what a step mounts as well as on what it runs, and the
# musl, libunwind and libc++ builds read no compiler-rt patch.  Applying both
# series in this step made an edit to any compiler-rt patch -- the usual change
# in this repository -- rebuild all three for nothing; mounting the whole
# patches/ tree here would do the same, since the key is the checksum of the
# mount and not of the files the step happened to read.
# ---------------------------------------------------------------------------
ARG MUSL_BASE=f21a9653
ARG LLVM_BASE=87f0227cb

RUN --mount=type=bind,from=musl-src,target=/src/musl \
    --mount=type=bind,from=llvm-src,target=/src/llvm \
    --mount=type=bind,from=patches,source=musl,target=/src/patches/musl \
    set -eu; \
    git config --global --add safe.directory '*'; \
    mkdir -p /musl-src /llvm-project; \
    git -C /src/musl archive "$MUSL_BASE" | tar -C /musl-src -xf -; \
    git -C /src/llvm archive "$LLVM_BASE" cmake llvm/cmake llvm/utils/llvm-lit third-party \
      compiler-rt libunwind runtimes libcxx libcxxabi libc \
      | tar -C /llvm-project -xf -; \
    cd /musl-src; for p in /src/patches/musl/*.patch; do echo "musl: $p"; patch -p1 --fuzz=0 --no-backup-if-mismatch -i "$p"; done

# ---------------------------------------------------------------------------
# 1. musl -> sysroot
#
# --prefix=/usr --syslibdir=/usr/lib with DESTDIR gives the layout clang's
# Linux driver expects directly: headers in <sysroot>/usr/include, crt*.o and
# libc.a/libc.so in <sysroot>/usr/lib. No post-install file shuffling.
#
# AR/RANLIB must be set explicitly: --target=<triple> makes musl's configure
# derive CROSS_COMPILE=<triple>- and look for aarch64-linux-musl-ar, which does
# not exist in a native container.
#
# musl's --target is not the compiler's.  configure uses it to pick the arch
# directory and to derive CROSS_COMPILE, and never passes it to $CC -- so on a
# host that is not the target, plain `clang` would compile x86_64 objects out of
# the riscv64 sources and the mismatch would surface as an assembler error deep in
# the build.  CC therefore carries the target itself, and it is spelled with
# Alpine's vendor rather than $TRIPLE's on purpose: that is the spelling clang
# loads /etc/clang20/<triple>.cfg for, so musl is built with the same
# -fstack-clash-protection here as on the target's own image.  Section 0a puts
# that file where this can find it and writes the triple down beside it;
# everything after this stage names $TRIPLE instead, as it did before, and gets no
# config file on either arrangement.
#
# LDFLAGS names lld because the only thing musl links rather than compiles is
# libc.so, and the default linker is /usr/bin/ld -- this image's binutils, which
# knows one architecture and answers "unrecognised emulation mode: elf64lriscv"
# when that architecture is not the target's.  lld knows them all.  It is not
# conditional on cross-building: a native build passes it too, so both
# arrangements produce the same libc.so.  Nothing the static link is about is
# affected either way -- libc.a and the crt objects are compiler output, and
# asan-musl-dynamic.cfg is the only configuration that links against libc.so.
# ---------------------------------------------------------------------------
RUN mkdir -p /build/musl && cd /build/musl \
 && /musl-src/configure \
      --target="$MUSL_TARGET" \
      --prefix=/usr \
      --exec-prefix=/usr \
      --syslibdir=/usr/lib \
      --disable-gcc-wrapper \
      CC="clang --target=$(cat $TARGET_PKGS/alpine-triple) -fuse-ld=lld" AR=llvm-ar RANLIB=llvm-ranlib \
      CFLAGS="$TARGET_CFLAGS" \
 && make -j"$(nproc)" \
 && make DESTDIR="$SYSROOT" install \
 && rm -rf /build/musl

# Alpine patches clang to pass -lssp_nonshared unconditionally for musl targets.
# Upstream musl's libc.a already defines __stack_chk_fail_local (weak), so an
# empty archive is all that is needed to satisfy the driver.
RUN llvm-ar rcs "$SYSROOT/usr/lib/libssp_nonshared.a"

# musl ships no kernel uapi headers; sanitizer_common needs linux/* and asm/*.
# This is every uapi directory Alpine's linux-headers package installs (it has no
# drm/), so a missing one means the package moved and the build has to stop here
# rather than surface later as a confusing compiler-rt error.
#
# They come from the target's copy of that package rather than this image's.  asm/
# is the reason: it is the one uapi directory whose contents are per-architecture,
# so an x86_64 image's asm/ in a riscv64 sysroot would give sanitizer_common the
# wrong syscall numbers and the wrong ptrace layout.  The rest are identical
# across architectures and are taken from the same place for consistency.
RUN set -eu; \
    for d in linux asm asm-generic mtd rdma sound video misc xen; do \
      cp -a "$TARGET_PKGS/usr/include/$d" "$SYSROOT/usr/include/"; \
    done

# ---------------------------------------------------------------------------
# 2. LLVM libunwind -> sysroot
#
# musl has no libgcc_eh/libgcc_s, and sanitizer_common's slow unwinder needs
# _Unwind_Backtrace / _Unwind_GetIP. Built from the same llvmorg-20.1.8 tree.
# CMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY is required because no builtins
# library exists yet, so CMake's default link-an-executable probe would fail.
# ---------------------------------------------------------------------------
RUN cmake -S /llvm-project/runtimes -B /build/libunwind -G Ninja \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_ENABLE_RUNTIMES=libunwind \
      -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_ASM_COMPILER=clang \
      -DCMAKE_C_COMPILER_TARGET="$TRIPLE" \
      -DCMAKE_CXX_COMPILER_TARGET="$TRIPLE" \
      -DCMAKE_ASM_COMPILER_TARGET="$TRIPLE" \
      -DCMAKE_SYSROOT="$SYSROOT" \
      -DCMAKE_C_FLAGS="-ffunction-sections -fdata-sections $TARGET_CFLAGS" \
      -DCMAKE_CXX_FLAGS="-ffunction-sections -fdata-sections $TARGET_CFLAGS" \
      -DCMAKE_ASM_FLAGS="$TARGET_CFLAGS" \
      -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY \
      -DLIBUNWIND_ENABLE_SHARED=OFF \
      -DLIBUNWIND_ENABLE_STATIC=ON \
      -DLIBUNWIND_USE_COMPILER_RT=ON \
      -DLIBUNWIND_INCLUDE_TESTS=OFF \
      -DCMAKE_INSTALL_PREFIX="$SYSROOT/usr" \
      -DLIBUNWIND_INSTALL_LIBRARY_DIR="$SYSROOT/usr/lib" \
      -DLIBUNWIND_INSTALL_INCLUDE_DIR="$SYSROOT/usr/include" \
 && ninja -C /build/libunwind -j"$(nproc)" \
 && ninja -C /build/libunwind install \
 && rm -rf /build/libunwind

# ---------------------------------------------------------------------------
# 2b. LLVM libc++ / libc++abi -> sysroot
#
# So that a C++ program can be linked static, RVC-clear and musl by this
# toolchain.  It is 2b rather than 3 because it belongs with the libunwind build
# above -- same tree, same flags, same CMake driver -- and because comments in
# sections 0a and 5 refer to "section 3" by number.
#
# WHY NOT THE libstdc++ THAT IS ALREADY HERE.  $TARGET_PKGS carries Alpine's
# libstdc++ for the target, and section 3 uses its HEADERS to build compiler-rt.
# Its ARCHIVES cannot be linked into anything this toolchain is for: Alpine
# builds riscv64 for the rv64gc baseline, so every member carries EF_RISCV_RVC
# and the linker ORs that bit into whatever is linked against them, whatever
# -march the caller passed for its own code.  Measured on the pinned image:
# 420 of 420 members across libstdc++.a (189), libsupc++.a (65), libstdc++exp.a
# (22), libstdc++fs.a (6), libgcc.a (133) and libgcc_eh.a (5) have e_flags 0x5.
# This is the same reason musl and libunwind are built here rather than taken
# from Alpine.  Rebuilding libstdc++ instead is not a lighter option and not
# really an option at all: it is not a standalone project, it falls out of a
# full cross-GCC build, and mixing GCC-built objects into a clang link gives up
# the one-compiler property callers of this image depend on.
#
# NOTHING ABOUT AN EXISTING BUILD CHANGES.  libstdc++ stays the driver's default
# C++ library for this target; libc++ is reached only by passing -stdlib=libc++.
# Section 3's compiler-rt still compiles against Alpine's libstdc++ headers,
# because it does not pass that flag.
#
# Three settings are load-bearing and each cost a build to find:
#
#   LIBCXXABI_ENABLE_THREADS=OFF, with LIBCXX_ENABLE_THREADS left ON.
#     THIS LOOKS LIKE A MISTAKE AND IS NOT.  libc++abi's __cxa_guard_acquire
#     calls syscall(SYS_gettid) -- cxa_guard_impl.h's PlatformThreadID(), used
#     by the GlobalMutexGuard that is selected when libc++abi is built with
#     threads -- once per function-local static with a non-constant initializer.
#     That is an unmodeled syscall for a caller gating on a modeled set, in the
#     PLAIN build, not only under a sanitizer.  Measured: a program with one such
#     static traces 1 gettid built this way against a threads-ON libc++abi and 0
#     against this one; an iostream program traces 4 and 0.
#     DO NOT "FINISH" THIS BY ALSO SETTING LIBCXX_ENABLE_THREADS=OFF.  That
#     removes std::mutex from the library outright -- <mutex> then fails with
#     "no type named 'mutex' in namespace 'std'" -- and it is not needed: with
#     only the ABI library's threading off, std::mutex still compiles, links and
#     runs, and the gettid is still gone.
#
#   LIBCXX_ENABLE_STATIC_ABI_LIBRARY=ON merges libc++abi's objects into
#     libc++.a, so a link needs -stdlib=libc++ and nothing else.  Without it the
#     driver adds -lc++ but not -lc++abi and the link ends in undefined
#     __cxa_guard_abort, __gxx_personality_v0 and typeinfo for std::exception.
#
#   libunwind is named in LLVM_ENABLE_RUNTIMES again, even though section 2 just
#     built and installed it, because LIBCXXABI_USE_LLVM_UNWINDER=ON is a hard
#     CMake error without it ("libunwind is not specified in
#     LLVM_ENABLE_RUNTIMES").  It is the same source built with the same flags,
#     so what it reinstalls is the same library.
#
# One more thing this needs is not here but in what is checked out: libcxx,
# libcxxabi AND libc must be in LLVM_PATHS.  libc is not a typo -- libc++ 20's
# charconv.cpp includes llvm-libc's "shared/fp_bits.h", and without that path
# the build fails at edge 2296 of 2302 with a missing-header error that names
# nothing to do with libc++.
# ---------------------------------------------------------------------------
RUN cmake -S /llvm-project/runtimes -B /build/libcxx -G Ninja \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_ENABLE_RUNTIMES="libunwind;libcxxabi;libcxx" \
      -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_ASM_COMPILER=clang \
      -DCMAKE_C_COMPILER_TARGET="$TRIPLE" \
      -DCMAKE_CXX_COMPILER_TARGET="$TRIPLE" \
      -DCMAKE_ASM_COMPILER_TARGET="$TRIPLE" \
      -DCMAKE_SYSROOT="$SYSROOT" \
      -DCMAKE_C_FLAGS="-ffunction-sections -fdata-sections $TARGET_CFLAGS" \
      -DCMAKE_CXX_FLAGS="-ffunction-sections -fdata-sections $TARGET_CFLAGS" \
      -DCMAKE_ASM_FLAGS="$TARGET_CFLAGS" \
      -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY \
      -DLIBUNWIND_ENABLE_SHARED=OFF \
      -DLIBUNWIND_ENABLE_STATIC=ON \
      -DLIBUNWIND_USE_COMPILER_RT=ON \
      -DLIBUNWIND_INCLUDE_TESTS=OFF \
      -DLIBUNWIND_INSTALL_LIBRARY_DIR="$SYSROOT/usr/lib" \
      -DLIBUNWIND_INSTALL_INCLUDE_DIR="$SYSROOT/usr/include" \
      -DLIBCXX_ENABLE_SHARED=OFF \
      -DLIBCXX_ENABLE_STATIC=ON \
      -DLIBCXXABI_ENABLE_SHARED=OFF \
      -DLIBCXXABI_ENABLE_STATIC=ON \
      -DLIBCXX_HAS_MUSL_LIBC=ON \
      -DLIBCXX_USE_COMPILER_RT=ON \
      -DLIBCXXABI_USE_COMPILER_RT=ON \
      -DLIBCXXABI_USE_LLVM_UNWINDER=ON \
      -DLIBCXX_CXX_ABI=libcxxabi \
      -DLIBCXX_ENABLE_STATIC_ABI_LIBRARY=ON \
      -DLIBCXX_ENABLE_THREADS=ON \
      -DLIBCXXABI_ENABLE_THREADS=OFF \
      -DLIBCXX_INCLUDE_BENCHMARKS=OFF \
      -DLIBCXX_INCLUDE_TESTS=OFF \
      -DLIBCXXABI_INCLUDE_TESTS=OFF \
      -DCMAKE_INSTALL_PREFIX="$SYSROOT/usr" \
      -DLIBCXX_INSTALL_LIBRARY_DIR="$SYSROOT/usr/lib" \
      -DLIBCXXABI_INSTALL_LIBRARY_DIR="$SYSROOT/usr/lib" \
      -DLIBCXX_INSTALL_INCLUDE_DIR="$SYSROOT/usr/include/c++/v1" \
      -DLIBCXX_INSTALL_INCLUDE_TARGET_DIR="$SYSROOT/usr/include/$TRIPLE/c++/v1" \
 && ninja -C /build/libcxx -j"$(nproc)" \
 && ninja -C /build/libcxx install \
 && rm -rf /build/libcxx

# ---------------------------------------------------------------------------
# The compiler-rt patch series, applied to the tree section 0 copied, here and
# not there so that a change to a patch invalidates the layer cache from this
# step on and not from section 0: musl, libunwind and libc++ above read nothing
# under compiler-rt/ and every patch in the series touches nothing outside it,
# so they are the same build either way and rebuilding them was the price of
# every patch edit -- about three minutes of each CI job.  Only patches/compiler-rt
# is mounted, for the reason section 0 gives.  The tree is still patched in
# place at /llvm-project, so the debug-info property section 0 describes is
# unchanged.
# ---------------------------------------------------------------------------
RUN --mount=type=bind,from=patches,source=compiler-rt,target=/src/patches/compiler-rt \
    set -eu; \
    cd /llvm-project; for p in /src/patches/compiler-rt/*.patch; do echo "compiler-rt: $p"; patch -p1 --fuzz=0 --no-backup-if-mismatch -i "$p"; done

# ---------------------------------------------------------------------------
# 3. compiler-rt (standalone) -> resource dir
#
# Non-obvious settings, all load-bearing:
#   COMPILER_RT_BUILD_CRT=ON        clang's Linux driver uses clang_rt.crtbegin.o
#                                   instead of GCC's crtbeginT.o only when that
#                                   file exists under the resource dir.
#   LLVM_ENABLE_PER_TARGET_RUNTIME_DIR=ON
#                                   lib/<triple>/ layout, which is what Alpine's
#                                   clang 20 looks for (-print-runtime-dir).
#   CMAKE_TRY_COMPILE_TARGET_TYPE   bootstrap: builtins do not exist yet.
#   SANITIZER_CXX_ABI=none          otherwise shared runtimes link -lstdc++.
#   CMAKE_SHARED_LINKER_FLAGS       shared runtimes need compiler-rt's own
#                                   crtbeginS.o (found via -resource-dir at the
#                                   build tree) and an unwinder. There is no
#                                   CMake dependency edge from the .so targets to
#                                   clang_rt.crtbegin.o, so it is a race under
#                                   -j16 -- hence `ninja crt builtins` first.
#                                   -fuse-ld=lld is there because the default
#                                   linker is this image's binutils, which knows
#                                   one architecture; lld knows them all, and it
#                                   is the linker every configuration this
#                                   toolchain writes already names.  It is not
#                                   conditional on cross-building: a native build
#                                   passes it too, so both arrangements produce
#                                   the same shared runtimes rather than the same
#                                   archives and different .so files.
#   CMAKE_CXX_FLAGS -idirafter ...  rtsan includes <new>; sanitizer flags force
#                                   -nostdinc++. -idirafter (NOT -isystem) puts
#                                   libstdc++ headers *after* the sysroot so
#                                   musl's <stdlib.h>/<string.h> still win --
#                                   -isystem here breaks dfsan. Both come from
#                                   the target's own libstdc++-dev (section 0a)
#                                   rather than from this image's, and the
#                                   second one is why: it is per-architecture,
#                                   and its c++config.h differs in more than
#                                   decoration -- _GLIBCXX_HAVE_ATOMIC_LOCK_POLICY
#                                   is defined for x86_64 and not for riscv64.
#   CAN_TARGET_<arch>=OFF           compiler-rt does not build for the target it
#                                   was given.  base-config-ix.cmake probes a
#                                   whole family of architectures around it, with
#                                   test_target_arch, and builds every one the
#                                   probe accepts -- each with its own -march and
#                                   -mfloat-abi, and all of them against the one
#                                   sysroot this image has.  The probe is a
#                                   compile and not a link, since
#                                   CMAKE_TRY_COMPILE_TARGET_TYPE is
#                                   STATIC_LIBRARY here, so it accepts an
#                                   architecture whose ABI the sysroot could not
#                                   possibly satisfy.
#                                   On an x86_64 host it finds a working -m32 and
#                                   configures x86_64;i386, and the 32-bit half
#                                   built against the 64-bit musl sysroot fails
#                                   sixteen static_asserts about struct layouts,
#                                   musl's own msghdr among them -- a loud
#                                   failure.  On a 32-bit Arm host it probes four
#                                   (armv4t, armv6m, arm, armhf) and the failure
#                                   is silent instead: all four have the same
#                                   struct layouts, so nothing static_asserts,
#                                   and three of them would be compiled
#                                   -mfloat-abi=soft against a sysroot whose libc
#                                   passes floats in VFP registers.  armhf is the
#                                   one that matches, so it is the one left on --
#                                   which is a statement about the sysroot and not
#                                   about Arm.  A soft-float 32-bit Arm target
#                                   (a triple ending eabi rather than eabihf)
#                                   would need armhf off and arm on instead, and
#                                   is not a target this has been built for.
#                                   Each name below is off on every architecture
#                                   except the one that probes for it, where the
#                                   name is not mentioned at all.
#                                   armhf is also not the architecture the triple
#                                   names.  ALL_ASAN_SUPPORTED_ARCH is `arm
#                                   armhf` on Linux, so there is no "armv7" to
#                                   ask for; asking anyway, by naming arm rather
#                                   than armv7 in the triple, makes
#                                   COMPILER_RT_DEFAULT_TARGET_ARCH equal to a
#                                   probed name, and then turning that name off
#                                   is a fatal error rather than a skip.  What
#                                   that costs is in section 5.
#   COMPILER_RT_USE_LLVM_UNWINDER=OFF
#                                   ON requires the in-tree unwind_static CMake
#                                   target, which does not exist standalone.
#   COMPILER_RT_BUILD_GWP_ASAN=OFF  the ONLY component disabled. gwp_asan (linked
#                                   into scudo) hard-errors "No suitable unwinder
#                                   library" without an in-tree unwinder target.
#                                   scudo_standalone itself still builds.
#
# Both resource dirs are built from these flags by one shell function, called
# twice with the settings that differ.  A shell function does not survive into
# the next RUN, so both builds are in a single RUN instead of in a script file
# sourced by two RUNs; that is the simpler arrangement and costs nothing, since
# an edit to the shared flags invalidates both builds either way.
#
# -resource-dir also relocates clang's own builtin headers (stddef.h, stdarg.h,
# arm_neon.h, ...). compiler-rt installs only sanitizer/ and fuzzer/, so each
# install ends by merging the 237 headers from Alpine's clang 20 resource dir in.
# ---------------------------------------------------------------------------

# ---------------------------------------------------------------------------
# 4. compiler-rt again, with link-time interception -> second resource dir
#
# That is the second call of build_compiler_rt below; the arguments it is called
# with are the whole difference between the two builds.  It has to be a second
# resource dir rather than a second archive in the first one: clang's Linux
# driver picks libclang_rt.asan.a out of the resource dir by name, and for a musl
# target it picks the *static* runtime even when the executable is dynamic.
# Overwriting it in place would therefore change every ASan link this toolchain
# does, including the dynamic ones, which cannot work without the --wrap flags.
# Two resource dirs keep the stock behaviour reachable and unchanged, and the
# config files below decide which one a given kind of link gets.
#
# The wrap list is produced by the build (lib/asan/CMakeLists.txt) from the
# archive it just linked, intersected with what the sysroot's libraries actually
# define, and installed next to it as libclang_rt.asan.a.wrap.
# ---------------------------------------------------------------------------
# -ffunction-sections -fdata-sections is written into every LLVM runtime build
# here (libunwind and libc++ above, both compiler-rt builds below) rather than
# left to TARGET_CFLAGS, because the static configs pass --gc-sections and the
# interceptors are collected by section: a runtime compiled without it has every
# interceptor body in one .text, and patch 0018's one-section-per-trampoline
# then keeps them all through the one that is live.  Only the riscv64 family the
# consuming project ships happened to carry the flags in TARGET_CFLAGS; the
# property has to hold on every architecture CI builds.  musl's configure adds
# the same two flags itself.
RUN set -eu; \
    cxx=$(echo "$TARGET_PKGS"/usr/include/c++/*/); \
    cxxarch="$cxx$(cat $TARGET_PKGS/alpine-triple)"; \
    [ -d "$cxxarch" ] || { echo "no libstdc++ headers for $APK_ARCH under $cxx"; exit 1; }; \
    build_compiler_rt() { \
      build=$1; prefix=$2; shift 2; \
      cmake -S /llvm-project/compiler-rt -B "$build" -G Ninja \
        -DCMAKE_BUILD_TYPE=Release \
        -DCOMPILER_RT_STANDALONE_BUILD=ON \
        -DLLVM_CMAKE_DIR=/llvm-project/llvm/cmake/modules \
        -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_ASM_COMPILER=clang \
        -DCMAKE_C_COMPILER_TARGET="$TRIPLE" \
        -DCMAKE_CXX_COMPILER_TARGET="$TRIPLE" \
        -DCMAKE_ASM_COMPILER_TARGET="$TRIPLE" \
        -DCMAKE_SYSROOT="$SYSROOT" \
        -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY \
        -DCOMPILER_RT_DEFAULT_TARGET_TRIPLE="$TRIPLE" \
        -DCAN_TARGET_i386=OFF \
        -DCAN_TARGET_arm=OFF \
        -DCAN_TARGET_armv4t=OFF \
        -DCAN_TARGET_armv6m=OFF \
        -DLLVM_ENABLE_PER_TARGET_RUNTIME_DIR=ON \
        -DCOMPILER_RT_BUILD_BUILTINS=ON \
        -DCOMPILER_RT_BUILD_CRT=ON \
        -DCOMPILER_RT_BUILD_SANITIZERS=ON \
        -DCOMPILER_RT_BUILD_LIBFUZZER=OFF \
        -DCOMPILER_RT_BUILD_MEMPROF=OFF \
        -DCOMPILER_RT_BUILD_PROFILE=OFF \
        -DCOMPILER_RT_BUILD_XRAY=OFF \
        -DCOMPILER_RT_BUILD_ORC=OFF \
        -DCOMPILER_RT_BUILD_CTX_PROFILE=OFF \
        -DCOMPILER_RT_BUILD_GWP_ASAN=OFF \
        -DCOMPILER_RT_CXX_LIBRARY=none \
        -DSANITIZER_CXX_ABI=none \
        -DCOMPILER_RT_USE_LLVM_UNWINDER=OFF \
        -DCOMPILER_RT_HAS_GCC_S_LIB=OFF \
        -DCOMPILER_RT_HAS_GCC_LIB=OFF \
        -DCOMPILER_RT_INCLUDE_TESTS=OFF \
        -DCMAKE_C_FLAGS="-ffunction-sections -fdata-sections $TARGET_CFLAGS" \
        -DCMAKE_ASM_FLAGS="$TARGET_CFLAGS" \
        -DCMAKE_CXX_FLAGS="-ffunction-sections -fdata-sections $TARGET_CFLAGS -idirafter $cxx -idirafter $cxxarch" \
        -DCMAKE_SHARED_LINKER_FLAGS="-fuse-ld=lld --rtlib=compiler-rt --unwindlib=none -resource-dir=$build -lunwind" \
        -DCMAKE_INSTALL_PREFIX="$prefix" \
        "$@"; \
      ninja -C "$build" -j"$(nproc)" crt builtins; \
      ninja -C "$build" -j"$(nproc)"; \
      ninja -C "$build" install; \
      rm -rf "$build"; \
      cp -a /usr/lib/llvm20/lib/clang/20/include/. "$prefix/include/"; \
    }; \
    build_compiler_rt /build/compiler-rt "$RESOURCE_DIR" \
      -DCOMPILER_RT_STATIC_INTERCEPTION=OFF; \
    build_compiler_rt /build/compiler-rt-static "$STATIC_RESOURCE_DIR" \
      -DCOMPILER_RT_STATIC_INTERCEPTION=ON \
      -DCOMPILER_RT_STATIC_INTERCEPTION_LIBS="$SYSROOT/usr/lib/libc.a;$SYSROOT/usr/lib/libunwind.a" \
      -DCMAKE_NM=/usr/bin/llvm-nm

# The second resource dir ships no libclang_rt.*.syms.  Each is the list of
# symbols a runtime exports, and clang hands the ASan one to the link as
# --dynamic-list so that an executable's dynamic symbol table exports the
# runtime to the instrumented shared objects it will load.  A static, non-PIE
# executable has no dynamic symbol table, so the list can do nothing it is for.
# What it still does, in lld, is make every name in it a root that --gc-sections
# may not collect -- all 1173 interceptor names among them -- and extract the
# archive members that define those names.  That, together with every
# trampoline sharing one section (patches/compiler-rt/0018-trampoline-sections
# .patch), is why a hello world used to carry all 391 interceptors and 543
# members of libc.a that nothing in it called.  The driver adds the option only
# when the file exists, so the file goes rather than the option being
# countermanded from the config, and the first resource dir, whose dynamic
# link is the case the list exists for, keeps its copies.  What this leaves in a
# binary is checked by conformance/checks/dead-interceptors.py.
RUN set -eu; \
    rtdir=$(clang --target="$TRIPLE" -resource-dir="$STATIC_RESOURCE_DIR" -print-runtime-dir); \
    [ -f "$rtdir/libclang_rt.asan.a.syms" ] || { echo "no libclang_rt.asan.a.syms under $rtdir to remove"; exit 1; }; \
    rm -v "$rtdir"/libclang_rt.*.syms

# ---------------------------------------------------------------------------
# 5. clang config files
#
# Seven of them, each explicitly named on the command line; none is a default
# config.  A default config is selected by target triple and loads on every
# invocation, so the wrap response file in it would break every link that does
# not pull in the ASan runtime (undefined symbol: __wrap_memset).  Each of these
# is a closed configuration instead: musl.cfg is a plain static musl link and
# asan-musl.cfg is a static musl link with ASan, the two the toolchain is for;
# asan-musl-nowrap.cfg and asan-musl-bfd.cfg are the two ways of getting the
# static link wrong, written down so the suite can build them and require them
# to fail; asan-musl-dynamic.cfg is the ordinary dynamic ASan link against the
# stock runtime.  Nothing is shared between any of them at run time.
#
# Every static one passes --gc-sections.  The objects are already built with
# -ffunction-sections -fdata-sections, so this is the half of that arrangement
# that was missing, and it is what lets the two changes beside the runtime
# build above -- no .syms in the second resource dir, one section per
# trampoline -- take a hello world from 1500KB to 551KB stripped.  It removes
# what nothing references; the wrap list is the same file and attaches the
# same interceptors to every call the program or the libc makes.  The README's
# "Binary size" section has the numbers and what remains.
#
# asan-musl-sealed.cfg and asan-musl-silent.cfg are the last two and are the odd
# ones: each is asan-musl.cfg plus one --defsym, and neither has a runtime of its
# own.  The runtime carries every behaviour and the link picks, which is what
# keeps each mode from being a second compiler-rt build, a second resource dir
# and a second wrap list to keep in step -- and what lets a test build one
# program both ways inside one image and compare the two.  What sealed init
# removes, and why each removal is safe, is in
# patches/compiler-rt/0012-sealed-init.patch and in sanitizer_sealed.h; the short
# version is that the runtime then reads no environment and no /proc, installs no
# signal handler, makes no prlimit64 or getpid and no madvise but MADV_DONTNEED
# (0020-sealed-madvise-dontneed.patch: the shadow of freed memory is handed back,
# which the caller's model charges for), and on riscv64 places its shadow at a
# constant offset while reading the size of the address space off the frame it
# started on; that a detection dies by SIGABRT rather than by an exit status
# the program could also have produced (abort_on_error is pinned to 1, after the
# program's own __asan_default_options() is parsed, so a hook cannot unpin it);
# and, from patches/compiler-rt/0015-sealed-allocation-failure.patch,
# an allocation the runtime cannot get for itself is a spin making no syscall
# rather than a diagnostic or an exit, so a run the kernel refuses memory to
# contributes nothing to the trace that a run it does not would lack -- unless
# an error report is in progress, in which case it aborts, so that a detection
# cannot vanish -- and, from 0016-sealed-refusal-sites.patch, the four places
# that rule did not reach, one of which was not a death but an arena based at
# address 0 from an unchecked mmap result.  0017-sealed-raw-mmap-returns.patch
# holds the two remaining unchecked mmap returns to the same rule; no sealed
# ASan binary reaches either, and no binary's behaviour changes.
# 0018-trampoline-sections.patch is not about the seal: it gives each
# interceptor's trampoline a section of its own so that --gc-sections can
# collect the interceptors a program never calls.
# 0019-sealed-shadow-first.patch reserves the shadow and its gap before any
# other mapping, with MAP_FIXED_NOREPLACE, so a mapping the kernel placed
# earlier is refused rather than covered, and on riscv64 reads the size of the
# address space off the initial frame so one runtime serves Sv39 and Sv48
# with the same constant offset.  0020-sealed-madvise-dontneed.patch issues
# madvise(MADV_DONTNEED) for real again, that advice alone.
# 0021-static-interception-extract-real.patch has every interceptor carry a
# parse-time reference to the function it replaces, so lld extracts the body
# from libc.a before wrapping (a program's weak reference to it, OpenSSL's
# getentropy, used to leave the body out and the interceptor jumping to 0).
# The reference sits in a section marked SHF_EXCLUDE, which the linker drops
# before it looks at relocations, so a link without --gc-sections -- every
# autoconf probe -- does not trip over the eleven names this libc lacks.
# What silent
# reports removes is in patches/compiler-rt/0014-silent-reports.patch and in
# sanitizer_silent.h: the
# runtime still detects, still dies, and dies with the same exit status, but
# writes nothing on the way down -- no report, no log file, no write syscalls to
# carry either.  The two symbols are independent, so a link may pass both
# --defsyms and get a sealed, silent binary; the configs each turn on one so the
# suite measures each mode against its own control.
#
# -fuse-ld=lld is not a preference.  GNU ld applies --wrap only to references
# that cross a translation unit boundary, and does not resolve a weak
# __real_<func> against the archive member defining <func> at all; a link with
# it succeeds and then dies on the first interceptor with a null REAL().  So the
# ASan config also passes --no-warn-backrefs, which lld ignores (it is already
# the default) and every other linker rejects outright, to turn "used the wrong
# linker" into a link error instead of a runtime crash.
#
# Six of the seven therefore name lld, and asan-musl-dynamic.cfg is not an
# exception being made: it needs a linker that can target
# this architecture and clang's default is not one on a cross-built image, and
# between the two that can -- lld, and the GNU ld asan-musl-bfd.cfg uses -- only
# lld is the same file on both arrangements.  Its subject is the stock runtime
# and its dlsym interception, not the linker.  asan-musl-bfd.cfg is the odd one
# out and is
# where the -B below goes: it must reach GNU ld, and -fuse-ld=bfd alone finds
# /usr/bin/ld.bfd, which on a cross-built image targets the machine underneath.
#
# The wrap response file is named by an absolute path, and that path is asked for
# rather than composed.  <resource dir>/lib/<TRIPLE> is where it lands on the two
# 64-bit targets and it is not where it lands on 32-bit Arm: compiler-rt names
# the per-target directory after the architecture it built, run back through
# get_compiler_rt_target(), and section 3 explains why the architecture it built
# is armhf while the triple says armv7.  So the runtime installed for
# TRIPLE=armv7-unknown-linux-musleabihf is in lib/arm-unknown-linux-musleabihf,
# and clang finds it there because its own lookup falls back to the same
# normalisation (ToolChain::getTargetSubDirPath, "various ways of naming the Arm
# architecture may have been normalised to simply arm").  -print-runtime-dir is
# that lookup, so asking it is the one spelling that is right on every target,
# and the existence check below turns a directory that moved into a failure here
# instead of a config file quietly naming a file that is not there.
# ---------------------------------------------------------------------------
RUN set -eu; \
    wrap=$(clang --target="$TRIPLE" -resource-dir="$STATIC_RESOURCE_DIR" \
             -print-runtime-dir)/libclang_rt.asan.a.wrap; \
    [ -f "$wrap" ] || { echo "no wrap list at $wrap"; exit 1; }; \
    set -- /usr/"$APK_ARCH"-alpine-linux-*/bin/ld.bfd; \
    [ $# -eq 1 ] && [ -x "$1" ] || { echo "expected exactly one target ld.bfd, found: $*"; exit 1; }; \
    bfd=${1%/ld.bfd}; \
    mkdir -p /opt/asan-static \
 && printf '%s\n' \
      "# Static musl, no sanitizer.  clang --config=/opt/asan-static/musl.cfg prog.c -o prog" \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$RESOURCE_DIR" \
      "--rtlib=compiler-rt" \
      "--unwindlib=libunwind" \
      "-fuse-ld=lld" \
      "-static" \
      "-Wl,--gc-sections" \
      > /opt/asan-static/musl.cfg \
 && printf '%s\n' \
      "# Static musl with AddressSanitizer." \
      "# clang --config=/opt/asan-static/asan-musl.cfg prog.c -o prog" \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$STATIC_RESOURCE_DIR" \
      "--rtlib=compiler-rt" \
      "--unwindlib=libunwind" \
      "-fuse-ld=lld" \
      "-fsanitize=address" \
      "-static" \
      "-Wl,--gc-sections" \
      "-Wl,--no-warn-backrefs" \
      "-Wl,@$wrap" \
      > /opt/asan-static/asan-musl.cfg \
 && printf '%s\n' \
      "# The same link WITHOUT the wrap response file: the runtime is in the" \
      "# binary and every interceptor is detached.  It exists so the test suite" \
      "# can build that configuration on purpose and assert that it does not" \
      "# work; it is not a way to build anything.  Since the runtime refers to" \
      "# __asan_wrap_list_present, which that file defines, the link fails." \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$STATIC_RESOURCE_DIR" \
      "--rtlib=compiler-rt" \
      "--unwindlib=libunwind" \
      "-fuse-ld=lld" \
      "-fsanitize=address" \
      "-static" \
      "-Wl,--gc-sections" \
      "-Wl,--no-warn-backrefs" \
      > /opt/asan-static/asan-musl-nowrap.cfg \
 && printf '%s\n' \
      "# The same link with GNU ld instead of lld, which cannot do what this" \
      "# needs: it applies --wrap only across translation unit boundaries and" \
      "# does not resolve a weak __real_<func> against the archive member that" \
      "# defines <func>.  It exists so the test suite can assert that using the" \
      "# wrong linker fails; --no-warn-backrefs is what it fails on first." \
      "# -B is where that ld is.  -fuse-ld=bfd on its own finds /usr/bin/ld.bfd," \
      "# which on a cross-built image targets the machine underneath and answers" \
      "# \"unrecognised emulation mode\" before it has looked at a single flag --" \
      "# a link that fails for the wrong reason, and a case that passes on it is" \
      "# testing nothing.  The one named here targets the target on both" \
      "# arrangements: the image's own binutils where it can, and Alpine's" \
      "# binutils-<arch> in the same place where it cannot." \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$STATIC_RESOURCE_DIR" \
      "--rtlib=compiler-rt" \
      "--unwindlib=libunwind" \
      "-B$bfd" \
      "-fuse-ld=bfd" \
      "-fsanitize=address" \
      "-static" \
      "-Wl,--gc-sections" \
      "-Wl,--no-warn-backrefs" \
      "-Wl,@$wrap" \
      > /opt/asan-static/asan-musl-bfd.cfg \
 && printf '%s\n' \
      "# An ordinary dynamic ASan link against the stock runtime, which resolves" \
      "# its interceptors with dlsym the way upstream compiler-rt does.  This is" \
      "# the configuration the other resource dir exists to leave alone, and the" \
      "# suite builds it to show that it still works." \
      "# It names lld, which it did not have to before, and the linker is not what" \
      "# this configuration is about: the stock runtime and its dlsym interception" \
      "# are.  What it had before was whatever clang defaults to, and that is a" \
      "# different GNU ld on the two arrangements -- the image's own 2.44 where the" \
      "# image is the target's architecture, binutils-<arch>'s 2.42 where it is not" \
      "# -- which is the one thing that made this configuration's output differ" \
      "# between them.  Naming a linker makes it the same file either way, and lld" \
      "# is the one every other configuration here already names." \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$RESOURCE_DIR" \
      "--rtlib=compiler-rt" \
      "--unwindlib=libunwind" \
      "-fuse-ld=lld" \
      "-fsanitize=address" \
      > /opt/asan-static/asan-musl-dynamic.cfg \
 && printf '%s\n' \
      "# The static ASan link with sealed init.  Same resource dir and same wrap" \
      "# list as asan-musl.cfg: the seal is the --defsym below, which the runtime" \
      "# reads as an undefined weak symbol, so this config differs from that one" \
      "# in exactly one line and the runtime is the same file." \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$STATIC_RESOURCE_DIR" \
      "--rtlib=compiler-rt" \
      "--unwindlib=libunwind" \
      "-fuse-ld=lld" \
      "-fsanitize=address" \
      "-static" \
      "-Wl,--gc-sections" \
      "-Wl,--no-warn-backrefs" \
      "-Wl,@$wrap" \
      "-Wl,--defsym=__sanitizer_sealed_init=1" \
      > /opt/asan-static/asan-musl-sealed.cfg \
 && printf '%s\n' \
      "# The static ASan link with silent reports.  Same resource dir and same" \
      "# wrap list as asan-musl.cfg: the silence is the --defsym below, which the" \
      "# runtime reads as an undefined weak symbol, so this config differs from" \
      "# that one in exactly one line and the runtime is the same file.  The" \
      "# runtime still detects and still dies with the same exit status; what is" \
      "# gone is the report, the log file, and the write syscalls that would have" \
      "# carried them.  Composes with the seal: add the sealed --defsym too." \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$STATIC_RESOURCE_DIR" \
      "--rtlib=compiler-rt" \
      "--unwindlib=libunwind" \
      "-fuse-ld=lld" \
      "-fsanitize=address" \
      "-static" \
      "-Wl,--gc-sections" \
      "-Wl,--no-warn-backrefs" \
      "-Wl,@$wrap" \
      "-Wl,--defsym=__sanitizer_silent_reports=1" \
      > /opt/asan-static/asan-musl-silent.cfg

# ---------------------------------------------------------------------------
# 6. Build-time verification
#
# What the wrap list contains is not checked here.  It used to be counted, with
# anything under 300 of the 380 entries treated as a generator that went wrong,
# and a count with eighty entries of slack is not a property worth having: the
# same file with seventy interceptors deleted passes it, and so does everything
# else in this image, because a name that is not in the file is simply not
# attached and nothing anywhere says so.  conformance/checks/wrap-list.py
# recomputes the whole file instead -- the __wrap_ symbols of the installed
# archive, intersected with what the sysroot's libc.a and libunwind.a define --
# and requires the installed one to be exactly that, which is the count and every
# name in it, and holds after the build as well as during it.  CI runs it against
# this image on both architectures.
# ---------------------------------------------------------------------------
RUN printf '#include <stdio.h>\nint main(void){printf("hello musl\\n");return 0;}\n' > /tmp/hello.c \
 && clang --config=/opt/asan-static/musl.cfg /tmp/hello.c -o /tmp/hello \
 && /tmp/hello

# The interceptors are attached by the link, so "the runtime is in the binary"
# says nothing about whether they are live: drop the response file and every one
# of them detaches while the binary still looks perfectly sanitized.  Plant an
# overflow that only an interceptor can report and require the report.
#
# The copy goes through a volatile function pointer, and that is the whole point
# of the case.  A source-level memcpy is lowered to llvm.memcpy and rewritten by
# the instrumentation pass into a call to __asan_memcpy, which is compiled into
# the program and reports the overflow whether anything is intercepted or not --
# so a gate written with a plain memcpy passes with every interceptor detached.
# An indirect call cannot be rewritten: it is a real call to whatever the symbol
# memcpy resolved to, which is __wrap_memcpy only if the link wrapped it.  A
# report here therefore means interception is live.
#
# The pointer's third parameter is size_t and has to be spelled that way.  It was
# `unsigned long`, which is size_t on the two 64-bit targets and is not size_t on
# a 32-bit one -- there it is `unsigned int`, the same width and a different
# type, and the initialization is an error rather than a conversion.
#
# The program aborts on the report, so its exit status is discarded explicitly
# and the greps are the whole assertion; the compile before it still has to
# succeed.
#
# Running it is the one step here that asks anything of the machine.  A 64-bit
# ASan process reserves ~14TiB of shadow before main, so a build whose RLIMIT_AS
# is smaller than that cannot start the binary at all -- and a container gets its
# RLIMIT_AS from dockerd, so `systemctl show docker -p LimitAS` is one setting
# away from every build on the host dying here on an allocator error that names
# ulimit and nothing else.  That is a fact about the machine; the compile and the
# link above have already succeeded, and no claim about this toolchain has been
# tested either way.
#
# So the limit is consulted, but only after the program has run and its report
# has not matched, and only ASan's own "could not reserve the shadow" is accepted
# as the reason.  That message has two spellings and the second is riscv64's.
# There the shadow goes through MapDynamicShadow(), whose MmapNoAccess() returns
# (uptr)-errno on failure while the CHECK_NE(map_start, ~(uptr)0) below it rejects
# only exactly -1, so an ENOMEM passes the guard, the arithmetic wraps, and what
# comes out is UnmapFromTo's
#
#   AddressSanitizer: CHECK failed: sanitizer_linux_libcdep.cpp:NNNN "((to >= from)) != (0)"
#
# with an empty stack and no mention of memory at all.  It means the same thing,
# and without it here VERIFY_RUNTIME=defer does not work on the one architecture
# whose reservation is the largest: the build fails at this step saying the report
# is not there and that a constrained address space is not why, when it is.  A host that can run the binary at all therefore takes exactly
# the branch it takes today, and a check that runs and fails is still a build
# failure whatever the limit is.  What the constrained host does instead is
# VERIFY_RUNTIME: "require", the default, is a build failure that says what the
# cause and the fix are rather than an opaque one; "defer" is for a host that
# needs the image anyway, and leaves this property to tests/run.sh, which asserts
# it in more detail than this does -- memcpy_overflow is this program, and its
# expectation is the resolved name of frame #0 -- from a docker run, where the
# limit can be lifted.  A deferral is recorded in the image, because a build log
# is not where anyone looks to find out what an image was not checked for.
ARG VERIFY_RUNTIME=require
RUN set -eu; \
    case "$VERIFY_RUNTIME" in \
      require|defer) ;; \
      *) echo "VERIFY_RUNTIME must be require or defer, not $VERIFY_RUNTIME" >&2; exit 2 ;; \
    esac; \
    printf '#include <stdlib.h>\n#include <string.h>\nstatic void *(*volatile copy)(void *, const void *, size_t) = memcpy;\nint main(void){volatile int n=100;char s[128];char*d=malloc(64);memset(s,65,sizeof s);copy(d,s,n);return 0;}\n' > /tmp/attach.c; \
    clang --config=/opt/asan-static/asan-musl.cfg -g /tmp/attach.c -o /tmp/attach; \
    ulimit -v unlimited 2>/dev/null || true; \
    limit=$(ulimit -v); \
    { ASAN_OPTIONS=detect_leaks=0 /tmp/attach 2>&1 || true; } | tee /tmp/attach.log; \
    if grep -q "WRITE of size 100" /tmp/attach.log \
       && grep -qE "#0 .*memcpy .*sanitizer_common_interceptors_memintrinsics" /tmp/attach.log; then \
      echo "interceptors are attached"; \
    elif [ "$limit" != unlimited ] \
         && grep -qE "ReserveShadowMemoryRange failed|CHECK failed: sanitizer_linux_libcdep.cpp.*to >= from" /tmp/attach.log; then \
      printf '%s\n' \
        "" \
        "  This build cannot run an ASan binary, so whether the interceptors are" \
        "  attached has not been tested either way.  AddressSanitizer reserves" \
        "  ~14TiB of shadow before main and this build's address space limit is" \
        "  ${limit}KB.  The compile and the link both succeeded, so nothing here is" \
        "  a statement about the toolchain." \
        "" \
        "  A container inherits RLIMIT_AS from the daemon.  Show it with" \
        "    systemctl show docker -p LimitAS" \
        "  and clear it (LimitAS=infinity) if the machine is yours to change.  A RUN" \
        "  step cannot raise the limit itself: that needs CAP_SYS_RESOURCE, so" \
        "  RUN --security=insecure on a docker-container builder started with" \
        "  --allow security.insecure." \
        "" >&2; \
      if [ "$VERIFY_RUNTIME" != defer ]; then \
        printf '%s\n' \
          "  To build the image anyway and leave this property to tests/run.sh," \
          "  which checks it more closely than this does and runs where the limit" \
          "  can be lifted:  --build-arg VERIFY_RUNTIME=defer" \
          "" >&2; \
        exit 1; \
      fi; \
      printf '%s\n' \
        "The interceptors in this image were not verified when it was built." \
        "" \
        "The build ran under an RLIMIT_AS of ${limit}KB, too small for the ~14TiB of" \
        "shadow an ASan process reserves, so the check could not start its binary." \
        "It was built with --build-arg VERIFY_RUNTIME=defer, which asks for that" \
        "deliberately and for this file to be here saying so." \
        "" \
        "Nothing has shown that this toolchain attaches its interceptors.  Run" \
        "tests/run.sh against this image before trusting it: memcpy_overflow is the" \
        "case the build-time check would have been, and asserts more than it did." \
        > /opt/asan-static/RUNTIME-UNVERIFIED; \
      echo "interceptors NOT verified: see /opt/asan-static/RUNTIME-UNVERIFIED"; \
    else \
      echo "the report is not there, and a constrained address space is not why:" >&2; \
      cat /tmp/attach.log >&2; \
      exit 1; \
    fi

# ...and that the same link without the response file is not silently allowed.
# The wrap list defines __asan_wrap_list_present and the runtime refers to it, so
# a command line that uses the statically intercepting resource dir and forgets
# -Wl,@<list> -- which is every hand-written one, since the config file is what
# usually carries it -- fails here instead of producing that detached binary.
RUN set -eu; \
    if clang --target="$TRIPLE" --sysroot="$SYSROOT" \
         -resource-dir="$STATIC_RESOURCE_DIR" --rtlib=compiler-rt \
         --unwindlib=libunwind -fuse-ld=lld -fsanitize=address -static \
         /tmp/attach.c -o /tmp/detached 2>/tmp/detached.log; then \
      echo "a link without the wrap list succeeded; the interceptors are detached"; \
      exit 1; \
    fi; \
    grep -q "__asan_wrap_list_present" /tmp/detached.log; \
    echo "a link without the wrap list fails"

ENV MUSL_SYSROOT=$SYSROOT \
    ASAN_RESOURCE_DIR=$RESOURCE_DIR
