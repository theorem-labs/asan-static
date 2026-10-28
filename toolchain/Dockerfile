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
# 128GB, so the two verification steps below that run an ASan binary (the
# interceptor check and the std::string check) need a container whose RLIMIT_AS
# is not small, and so does the MSan step.  That step also needs riscv64's MSan
# layout (patch 0029) to be free: Sv48 hardware, or a qemu-user that maps the
# program above MSan's shadow (9.2 and later do, 8.1 and 8.2 do not).  The
# others do not: the plain static musl link runs and the link that must fail
# still fails.  A 32-bit target needs none of that -- its shadow is about 512MiB.
# See the "Running an emulated image" section of README.md.
#
# Six of the steps at the end run binaries for the target; one of them, the
# MemorySanitizer test, only on x86_64, aarch64 and riscv64, the targets with an
# msan runtime.  On a native build that is nothing to arrange; on a cross build the
# machine has to be able to execute them, which on Linux means binfmt_misc with
# qemu-user registered -- `docker run --privileged tonistiigi/binfmt --install
# <arch>` once per boot, or docker/setup-qemu-action in CI.  Only these six
# steps need it.  Nothing the image is for does.  x86_64 and aarch64 MSan cannot
# run under qemu-user at all (qemu places the guest stack inside MSan's fixed
# ranges), so on those two the MSan step detects emulation and does not run its
# binaries; it treats that as a deferral, through VERIFY_RUNTIME like the ASan
# check: the default, "require", fails the build, and "defer" records the MSan
# runs as not verified.  Build aarch64 natively to have MSan checked; see that
# step.  riscv64 MSan runs under qemu-user 9.2 or later; see that step too.
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
#   /opt/musl-sysroot/usr/lib/asan-libcxx  a second libc++ / libc++abi, built
#                                      with ASan, for the static ASan configs
#   /opt/musl-sysroot/usr/lib/hardened-libcxx
#                                      a third libc++ / libc++abi, built with
#                                      the extensive hardening mode, for
#                                      musl-hardened.cfg
#   /opt/ubsan-vptr-trap/libclang_rt.ubsan_vptr_trap.a
#                                      a trap-only -fsanitize=vptr handler, for
#                                      ubsan-vptr-musl.cfg
#   /opt/asan-rt/lib/clang/20          stock compiler-rt: dlsym interception,
#                                      what a dynamic link uses
#   /opt/asan-static-rt/lib/clang/20   compiler-rt with COMPILER_RT_STATIC_INTERCEPTION:
#                                      the static ASan runtime is attached by the
#                                      linker, plus libclang_rt.asan.a.wrap
#   /opt/asan-static/musl.cfg          clang config: plain static musl
#   /opt/asan-static/musl-hardened.cfg clang config: musl.cfg, with C++ against
#                                      the hardened libc++
#   /opt/asan-static/ubsan-vptr-musl.cfg
#                                      clang config: musl.cfg with
#                                      -fsanitize=vptr checks that trap, through
#                                      the handler above and no UBSan runtime
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
#   /opt/asan-static/msan-musl.cfg     clang config: static musl + MemorySanitizer,
#                                      with libclang_rt.msan.a.wrap.  Emitted only
#                                      on the targets whose msan runtime the build
#                                      produced (x86_64, aarch64, riscv64); the
#                                      other target has none -- see section 5b
#   /opt/asan-static/msan-musl-sealed.cfg
#                                      clang config: the same static MSan link
#                                      with sealed init turned on.  Same runtime
#                                      and wrap list as msan-musl.cfg; one --defsym
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
ARG ASAN_LIBCXX=$SYSROOT/usr/lib/asan-libcxx
ARG HARDENED_LIBCXX=$SYSROOT/usr/lib/hardened-libcxx
ARG VPTR_TRAP=/opt/ubsan-vptr-trap

# Extra target flags threaded through every component that ends up inside a link
# this image makes: musl, libunwind, the three libc++ builds, both compiler-rt
# builds, the vptr handler, and the config files (nine, and two more for MSan on
# x86_64, aarch64 and riscv64).  Empty by default, so an image built without it is the
# stock one.
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
        bash~5.2 \
        cmake~3.31 \
        meson~1.8 \
        ninja-build~1.12 \
        pkgconf~2.4 \
        build-base~0.5 \
        linux-headers~6.14 \
        alpine-keys~2.5 \
        git~2.49 \
        python3~3.12 \
        openjdk21-jre-headless~21.0 \
        patch~2.8 \
        file~5.46 \
        docker-cli~28.3 \
 && ln -sf /usr/lib/ninja-build/bin/ninja /usr/bin/ninja

# A GNU ld that targets the target.  One of the configurations in section 5 uses a
# linker that is not lld, and it exists to be a link that must fail: what
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
# 4b. LLVM libc++ / libc++abi again, built with ASan -> $ASAN_LIBCXX
#
# libc++ annotates std::string for AddressSanitizer only when libc++ itself was
# built with ASan.  Every annotation in <string> is guarded by
# _LIBCPP_INSTRUMENTED_WITH_ASAN, and libcxx/CMakeLists.txt defines that macro to
# 1 in __config_site only when LLVM_USE_SANITIZER contains "Address".  Section
# 2b builds without it, so with 2b's copy an ASan build reports nothing for a
# read or write past size() and within capacity() of a std::string, while it
# does report the same access on a std::vector, whose annotations need no
# instrumented library.
#
# Section 2b's copy is the one every configuration links, the plain and UBSan
# ones included, so it stays as it is.  This is a second copy, built with
# -DLLVM_USE_SANITIZER=Address and installed under its own prefix with its own
# __config_site.  Only asan-musl.cfg, asan-musl-sealed.cfg and
# asan-musl-silent.cfg use it (section 5): its headers replace the default C++
# headers (-stdlib++-isystem), and its library directory comes before
# $SYSROOT/usr/lib.  A C compile does not use the C++ headers and a C link takes
# nothing from that directory, so C ASan binaries and every non-ASan build are
# unchanged.
#
# The CMake flags are section 2b's, with three differences: the sanitizer, the
# install prefix, and -isystem /llvm-project/compiler-rt/include.  libunwind is
# still named in LLVM_ENABLE_RUNTIMES, because LIBCXXABI_USE_LLVM_UNWINDER=ON is
# a hard CMake error without it (2b), and keeping that setting gives libc++abi
# the same <unwind.h> as 2b's copy.  The build then compiles libunwind too, with
# ASan, and libunwind.cpp includes <sanitizer/asan_interface.h>; the in-tree
# compiler-rt headers are what it finds.  Only install-cxx and install-cxxabi
# run, so that instrumented libunwind is never installed, and an ASan link keeps
# section 2's libunwind.a, which is also what ASan's own unwinding goes through.
#
# Every translation unit in an ASan link that uses std::string, std::vector or
# std::deque must then be built with -fsanitize=address: one built without it
# does not update the annotations, and correct code can report
# container-overflow (README.md, "Using the image").
# ---------------------------------------------------------------------------
RUN cmake -S /llvm-project/runtimes -B /build/asan-libcxx -G Ninja \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_ENABLE_RUNTIMES="libunwind;libcxxabi;libcxx" \
      -DLLVM_USE_SANITIZER=Address \
      -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_ASM_COMPILER=clang \
      -DCMAKE_C_COMPILER_TARGET="$TRIPLE" \
      -DCMAKE_CXX_COMPILER_TARGET="$TRIPLE" \
      -DCMAKE_ASM_COMPILER_TARGET="$TRIPLE" \
      -DCMAKE_SYSROOT="$SYSROOT" \
      -DCMAKE_C_FLAGS="-ffunction-sections -fdata-sections -isystem /llvm-project/compiler-rt/include $TARGET_CFLAGS" \
      -DCMAKE_CXX_FLAGS="-ffunction-sections -fdata-sections -isystem /llvm-project/compiler-rt/include $TARGET_CFLAGS" \
      -DCMAKE_ASM_FLAGS="$TARGET_CFLAGS" \
      -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY \
      -DLIBUNWIND_ENABLE_SHARED=OFF \
      -DLIBUNWIND_ENABLE_STATIC=ON \
      -DLIBUNWIND_USE_COMPILER_RT=ON \
      -DLIBUNWIND_INCLUDE_TESTS=OFF \
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
      -DCMAKE_INSTALL_PREFIX="$ASAN_LIBCXX" \
      -DLIBCXX_INSTALL_LIBRARY_DIR="$ASAN_LIBCXX/lib" \
      -DLIBCXXABI_INSTALL_LIBRARY_DIR="$ASAN_LIBCXX/lib" \
      -DLIBCXX_INSTALL_INCLUDE_DIR="$ASAN_LIBCXX/include/c++/v1" \
      -DLIBCXX_INSTALL_INCLUDE_TARGET_DIR="$ASAN_LIBCXX/include/$TRIPLE/c++/v1" \
 && ninja -C /build/asan-libcxx -j"$(nproc)" cxx cxxabi \
 && ninja -C /build/asan-libcxx install-cxx install-cxxabi \
 && rm -rf /build/asan-libcxx

# ---------------------------------------------------------------------------
# 4c. LLVM libc++ / libc++abi a third time, hardened -> $HARDENED_LIBCXX
#
# libc++'s hardening mode turns the library's preconditions into checks: an
# index past the end of a std::span, std::string_view, std::array or
# std::vector, front() or back() of an empty container, operator* of an empty
# std::optional, an iterator range whose ends are swapped, a null pointer with
# a nonzero length.  In every mode but debug a failed check runs the default
# assertion handler, which is __builtin_verbose_trap: the program dies on the
# trap instruction and prints nothing, as a -fsanitize-trap check does.
#
# Defining _LIBCPP_HARDENING_MODE when the program is compiled reaches only the
# code that the program's own translation units compile: the inline functions
# and templates of the headers.  The members of std::string and of the streams
# that libc++ instantiates in libc++.a are compiled once, when the library is,
# and the program calls that copy.  So the library is built here with
# LIBCXX_HARDENING_MODE=extensive.  Its __config_site then makes extensive the
# default for every program compiled against its headers, and its archive has
# the checks too.  Measured, riscv64: section 2b's libc++.a has 6 trap
# instructions and this one has 319.  A program that calls
# std::string::find(nullptr, pos, n) with pos past the end and n nonzero, or
# std::string::rfind(nullptr, 0, n), traps with this copy, and does not with
# section 2b's copy and -D_LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_EXTENSIVE.
#
# As with 4b, section 2b's copy stays as it is, and this one has its own prefix
# and its own __config_site.  Only musl-hardened.cfg (section 5) uses it.  The
# CMake flags are section 2b's, with two differences: the hardening mode and
# the install prefix.  As in 4b, only install-cxx and install-cxxabi run.
# ---------------------------------------------------------------------------
RUN cmake -S /llvm-project/runtimes -B /build/hardened-libcxx -G Ninja \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_ENABLE_RUNTIMES="libunwind;libcxxabi;libcxx" \
      -DLIBCXX_HARDENING_MODE=extensive \
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
      -DCMAKE_INSTALL_PREFIX="$HARDENED_LIBCXX" \
      -DLIBCXX_INSTALL_LIBRARY_DIR="$HARDENED_LIBCXX/lib" \
      -DLIBCXXABI_INSTALL_LIBRARY_DIR="$HARDENED_LIBCXX/lib" \
      -DLIBCXX_INSTALL_INCLUDE_DIR="$HARDENED_LIBCXX/include/c++/v1" \
      -DLIBCXX_INSTALL_INCLUDE_TARGET_DIR="$HARDENED_LIBCXX/include/$TRIPLE/c++/v1" \
 && ninja -C /build/hardened-libcxx -j"$(nproc)" cxx cxxabi \
 && ninja -C /build/hardened-libcxx install-cxx install-cxxabi \
 && rm -rf /build/hardened-libcxx

# ---------------------------------------------------------------------------
# 4d. A trap-only -fsanitize=vptr handler -> $VPTR_TRAP
#
# A vptr check (a downcast, member call or member access whose object's dynamic
# type is not the static type) cannot be a trap: -fsanitize-trap=vptr is an
# error, because the decision needs the object's RTTI, which the inline check
# cannot read.  So -fsanitize-trap=all leaves vptr out, and the check otherwise
# calls __ubsan_handle_dynamic_type_cache_miss_abort in the UBSan runtime.
# toolchain/ubsan_vptr_trap.cpp defines that handler and the cache it fills,
# and nothing else: the runtime's decision, then __builtin_trap() on a
# mismatch.  A link that passes -fno-sanitize-link-runtime and this archive gets
# vptr checks with the same outcome as the other trap-mode checks and no UBSan
# runtime (ubsan-vptr-musl.cfg, section 5).
#
# It is compiled with the target's flags and no sanitizer, against section 2b's
# libc++ headers.  It reads the type_info layout of libc++abi, which every
# libc++ copy in this image shares.  The archive's name matches
# libclang_rt.*.a, as the other sanitizer runtimes' names do.
# ---------------------------------------------------------------------------
COPY ubsan_vptr_trap.cpp /src/ubsan_vptr_trap.cpp
RUN set -eu; \
    mkdir -p "$VPTR_TRAP"; \
    clang++ --target="$TRIPLE" --sysroot="$SYSROOT" $TARGET_CFLAGS -stdlib=libc++ \
      -O2 -ffunction-sections -fdata-sections -fno-exceptions \
      -c /src/ubsan_vptr_trap.cpp -o /tmp/ubsan_vptr_trap.o; \
    llvm-ar rcsD "$VPTR_TRAP/libclang_rt.ubsan_vptr_trap.a" /tmp/ubsan_vptr_trap.o; \
    rm /tmp/ubsan_vptr_trap.o

# ---------------------------------------------------------------------------
# 5. clang config files
#
# Nine of them, each explicitly named on the command line; none is a default
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
# musl-hardened.cfg is musl.cfg with section 4c's libc++ headers and library in
# front of the default ones, as the ASan configs put section 4b's.
# ubsan-vptr-musl.cfg is musl.cfg plus the vptr check and section 4d's handler:
# -fno-sanitize-trap=vptr, because vptr cannot trap; -fno-sanitize-recover=vptr,
# so the check calls the _abort handler; -fno-sanitize-link-runtime, so the
# driver links no UBSan runtime; and the archive, as a -Wl, argument, so a
# compile with -c does not see a linker input.  clang reads a config's options
# before the command line, so a command line that adds -fsanitize-trap=all
# must add -fno-sanitize-trap=vptr after it: -fsanitize-trap=all would
# otherwise name vptr again, and clang refuses vptr in trap mode.  Neither
# config can be combined with the ASan ones: each picks its own libc++, and
# -fno-sanitize-link-runtime would drop the ASan runtime.
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
# Eight of the nine therefore name lld, and asan-musl-dynamic.cfg is not an
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
    [ -f "$VPTR_TRAP/libclang_rt.ubsan_vptr_trap.a" ] || { echo "no vptr handler in $VPTR_TRAP"; exit 1; }; \
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
      "# Static musl, no sanitizer, C++ against the hardened libc++ (section 4c)." \
      "# clang++ --config=/opt/asan-static/musl-hardened.cfg -stdlib=libc++ prog.cpp -o prog" \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$RESOURCE_DIR" \
      "-stdlib++-isystem$HARDENED_LIBCXX/include/$TRIPLE/c++/v1" \
      "-stdlib++-isystem$HARDENED_LIBCXX/include/c++/v1" \
      "-L$HARDENED_LIBCXX/lib" \
      "--rtlib=compiler-rt" \
      "--unwindlib=libunwind" \
      "-fuse-ld=lld" \
      "-static" \
      "-Wl,--gc-sections" \
      > /opt/asan-static/musl-hardened.cfg \
 && printf '%s\n' \
      "# Static musl with -fsanitize=vptr checks that trap, through the handler" \
      "# in $VPTR_TRAP and no UBSan runtime (section 4d)." \
      "# clang++ --config=/opt/asan-static/ubsan-vptr-musl.cfg -stdlib=libc++ prog.cpp -o prog" \
      "# With -fsanitize-trap=all on the command line, add -fno-sanitize-trap=vptr after it." \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$RESOURCE_DIR" \
      "--rtlib=compiler-rt" \
      "--unwindlib=libunwind" \
      "-fuse-ld=lld" \
      "-static" \
      "-Wl,--gc-sections" \
      "-fsanitize=vptr" \
      "-fno-sanitize-trap=vptr" \
      "-fno-sanitize-recover=vptr" \
      "-fno-sanitize-link-runtime" \
      "-Wl,$VPTR_TRAP/libclang_rt.ubsan_vptr_trap.a" \
      > /opt/asan-static/ubsan-vptr-musl.cfg \
 && printf '%s\n' \
      "# Static musl with AddressSanitizer." \
      "# clang --config=/opt/asan-static/asan-musl.cfg prog.c -o prog" \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$STATIC_RESOURCE_DIR" \
      "-stdlib++-isystem$ASAN_LIBCXX/include/$TRIPLE/c++/v1" \
      "-stdlib++-isystem$ASAN_LIBCXX/include/c++/v1" \
      "-L$ASAN_LIBCXX/lib" \
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
      "-stdlib++-isystem$ASAN_LIBCXX/include/$TRIPLE/c++/v1" \
      "-stdlib++-isystem$ASAN_LIBCXX/include/c++/v1" \
      "-L$ASAN_LIBCXX/lib" \
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
      "-stdlib++-isystem$ASAN_LIBCXX/include/$TRIPLE/c++/v1" \
      "-stdlib++-isystem$ASAN_LIBCXX/include/c++/v1" \
      "-L$ASAN_LIBCXX/lib" \
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
# 5b. MemorySanitizer configs
#
# The same static-musl link as asan-musl.cfg / asan-musl-sealed.cfg, with
# -fsanitize=memory and msan's own wrap list.  Emitted only where the build
# produced an msan runtime: compiler-rt builds msan for the architectures in
# ALL_MSAN_SUPPORTED_ARCH, which here are x86_64, aarch64 and, with patch 0029,
# riscv64.  armv7 has no msan runtime in compiler-rt 20.1.8, so there is nothing
# to point a config at and this step skips it.
#
# riscv64 gets one more line, and it is the compiler half of the change whose
# runtime half is patch 0029 (llvm/llvm-project#235906).  The clang in this
# image is Alpine's 20.1.8, which has no MemorySanitizer mapping for riscv64:
# MemorySanitizer::initializeModule reaches report_fatal_error("unsupported
# architecture") on every riscv64 module.  Before it looks at the triple,
# though, it checks whether -msan-shadow-base or -msan-origin-base was given,
# and if either was it takes all four mapping constants from the -msan-*
# options instead.  The two options below are #235906's riscv64 constants,
# which are LoongArch64's -- and-mask 0, xor-mask 0x500000000000, shadow-base
# 0, origin-base 0x100000000000, the two zeros being the options' defaults --
# and they are the layout patch 0029 gives the runtime.  Measured on this
# clang: with the two options, -O1 IR and objects, with and without
# -fsanitize-memory-track-origins=2, are byte-identical to those of a clang
# 20.1.8 built with #235906's MemorySanitizer.cpp hunk and given no options,
# for zstd's zstd_decompress.c and zstd_decompress_block.c, cgif.c, and three
# small files (an uninitialized read, varargs, struct copies).  With
# -msan-xor-mask alone the override is not taken and the compile still fails.
# The options are written into the configs rather than left to the caller
# because a riscv64 -fsanitize=memory compile without them does not compile.
# On x86_64 and aarch64 the line is not written, so their configs are as they
# were.  When a clang with #235906 replaces this one, the line and patch 0029
# can both go.
RUN set -eu; \
    rtdir=$(clang --target="$TRIPLE" -resource-dir="$STATIC_RESOURCE_DIR" \
              -print-runtime-dir); \
    msan="$rtdir/libclang_rt.msan.a"; \
    mwrap="$msan.wrap"; \
    if [ ! -f "$msan" ]; then \
      echo "no msan runtime under $rtdir for $TRIPLE; skipping msan configs"; \
      exit 0; \
    fi; \
    [ -f "$mwrap" ] || { echo "msan runtime without a wrap list at $mwrap"; exit 1; }; \
    case "$TRIPLE" in \
      riscv64-*) mmap="-mllvm -msan-xor-mask=0x500000000000 -mllvm -msan-origin-base=0x100000000000" ;; \
      *)         mmap= ;; \
    esac; \
    mkdir -p /opt/asan-static \
 && printf '%s\n' \
      "# Static musl with MemorySanitizer." \
      "# clang --config=/opt/asan-static/msan-musl.cfg prog.c -o prog" \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$STATIC_RESOURCE_DIR" \
      "--rtlib=compiler-rt" \
      "--unwindlib=libunwind" \
      "-fuse-ld=lld" \
      "-fsanitize=memory" \
      ${mmap:+"$mmap"} \
      "-static" \
      "-Wl,--gc-sections" \
      "-Wl,--no-warn-backrefs" \
      "-Wl,@$mwrap" \
      > /opt/asan-static/msan-musl.cfg \
 && printf '%s\n' \
      "# The static MSan link with sealed init.  Same resource dir and same wrap" \
      "# list as msan-musl.cfg: the seal is the --defsym below, which the runtime" \
      "# reads as an undefined weak symbol, so this config differs from that one" \
      "# in exactly one line and the runtime is the same file." \
      "--target=$TRIPLE" \
      "--sysroot=$SYSROOT" \
      "$TARGET_CFLAGS" \
      "-resource-dir=$STATIC_RESOURCE_DIR" \
      "--rtlib=compiler-rt" \
      "--unwindlib=libunwind" \
      "-fuse-ld=lld" \
      "-fsanitize=memory" \
      ${mmap:+"$mmap"} \
      "-static" \
      "-Wl,--gc-sections" \
      "-Wl,--no-warn-backrefs" \
      "-Wl,@$mwrap" \
      "-Wl,--defsym=__sanitizer_sealed_init=1" \
      > /opt/asan-static/msan-musl-sealed.cfg

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

# ...and that std::string reports container-overflow in the ASan builds, through
# the copy in section 4b, and nowhere else.  The macro is 1 in that copy and 0
# in the default one; a plain C++ program still links against the default copy
# and runs; a sealed ASan program that reads past size() and within capacity()
# reports (exit 134); and one that grows, shrinks and copies a string correctly
# does not.  The two ASan programs run only when the interceptor check above
# could run its binary.  The sealed runtime does not lower RLIMIT_CORE, as
# disable_coredump does in the other configs, so the program that reports turns
# off its own core dump: a core of its address space, shadow included, takes
# minutes to write and can fill the build host's disk.
RUN set -eu; \
    grep -q "^#define _LIBCPP_INSTRUMENTED_WITH_ASAN 1$" "$ASAN_LIBCXX/include/$TRIPLE/c++/v1/__config_site"; \
    grep -q "^#define _LIBCPP_INSTRUMENTED_WITH_ASAN 0$" "$SYSROOT/usr/include/$TRIPLE/c++/v1/__config_site"; \
    printf '#include <cstdio>\n#include <string>\nint main(){std::string s("plain");std::printf("%%s\\n",s.c_str());return 0;}\n' > /tmp/plain.cpp; \
    clang++ --config=/opt/asan-static/musl.cfg -stdlib=libc++ /tmp/plain.cpp -o /tmp/plain-cxx; \
    [ "$(/tmp/plain-cxx)" = plain ]; \
    printf '#include <cstdio>\n#include <string>\n#include <sys/prctl.h>\n#include <sys/resource.h>\nint main(){struct rlimit r={0,0};setrlimit(RLIMIT_CORE,&r);prctl(PR_SET_DUMPABLE,0);std::string s;s.reserve(64);s.resize(4,(char)97);volatile char c=s.data()[20];std::printf("%%d\\n",(int)c);return 0;}\n' > /tmp/string-over.cpp; \
    printf '#include <cstdio>\n#include <string>\nint main(){std::string s("hello");for(int i=0;i<200;i++)s+=(char)(97+i%%26);s.resize(10);s.shrink_to_fit();std::string t=s+s;std::printf("%%zu\\n",t.size());return 0;}\n' > /tmp/string-ok.cpp; \
    clang++ --config=/opt/asan-static/asan-musl-sealed.cfg -stdlib=libc++ -O1 /tmp/string-over.cpp -o /tmp/string-over; \
    clang++ --config=/opt/asan-static/asan-musl-sealed.cfg -stdlib=libc++ -O1 /tmp/string-ok.cpp -o /tmp/string-ok; \
    if [ -e /opt/asan-static/RUNTIME-UNVERIFIED ]; then \
      echo "std::string container-overflow: built, not run (RUNTIME-UNVERIFIED)"; \
    else \
      st=0; /tmp/string-over >/dev/null 2>&1 || st=$?; \
      [ "$st" = 134 ] || { echo "a std::string read past size() did not report (exit $st)" >&2; exit 1; }; \
      [ "$(/tmp/string-ok)" = 20 ]; \
      echo "std::string container-overflow is reported; a correct program does not report"; \
    fi

# MemorySanitizer test, on the targets that have an msan runtime (x86_64,
# aarch64 and riscv64; on armv7 section 5b wrote no msan config and this step
# says so and does nothing).
#
# Two programs, each linked with msan-musl.cfg (plain) and msan-musl-sealed.cfg
# (sealed), so four binaries:
#
#   clean -- a fully initialized heap block, a wrapped fopen/fread/fclose and a
#            printf.  A pass is exit 0 with its line "msan ok <n>".  The FILE
#            calls are what reach unpoison_file (patch 0028), and the sealed
#            binary's start-up is what reads the recorded init frame (0026).
#   umr   -- branches on an uninitialized heap value.  A pass is MSan's
#            use-of-uninitialized-value report and death: by exit status 1 (the
#            common exitcode flag, which MSan leaves at its default) when plain,
#            by SIGABRT (134) when sealed (abort_on_error is pinned, patch
#            0027).  The report proves interception and
#            instrumentation are live.
#
# Both programs first turn core dumps off (RLIMIT_CORE 0 and PR_SET_DUMPABLE
# 0).  A sealed detection dies by SIGABRT and the runtime leaves RLIMIT_CORE
# alone, so without that the kernel dumps a core; where core_pattern pipes to a
# handler, as on the CI runners, writing it outlasts the timeout below and a
# run that reported is killed (137) instead of ending by 134.
#
# Each binary is run MSAN_RUNS times, under `timeout -s KILL`, and every run is
# classified by one function, msan_check.  The image's timeout is busybox's: it
# kills the program itself, so the shell sees 128+signal -- 137 for KILL -- and
# never 124.  A run's outcome is one of:
#
#   pass      as above.
#   spin      sealed only: killed by the timeout (137) having printed nothing
#             (the log holds at most the shell's own "Killed").
#             MSan's x86_64 and aarch64 layouts have fixed shadow, origin and
#             allocator ranges, and a sealed runtime that finds one of the
#             kernel's start-up mappings inside them spins rather than re-exec
#             (README.md, "Known limitation").  With full ASLR that is a share
#             of runs, not all of them.
#   re-exec   plain only: the plain runtime answers the same conflict by
#             re-execing with ASLR off, and in a RUN step the default seccomp
#             profile refuses its personality(ADDR_NO_RANDOMIZE), which MSan
#             reports as one CHECK, matched here by its full text (the
#             condition as printed, with musl's ADDR_NO_RANDOMIZE expanded):
#               MemorySanitizer: CHECK failed: msan_linux.cpp:<line>
#               "((personality(old_personality | 0x0040000))) != ((-1))"
#   as-limit  MSan's own "can not mmap the shadow" / "Cannot protect memory
#             range", accepted only when this build's RLIMIT_AS is not
#             unlimited -- the shadow is TBs -- or on riscv64, whose layout
#             (patch 0029) does not fit Sv39 hardware, and which qemu-user
#             8.1 and 8.2 take by mapping the program inside MSan's shadow.  On
#             riscv64 a plain re-exec counts as that kind of miss too.
#
# Anything else -- another signal, another exit status, a report missing or
# where none belongs, a CHECK other than that one -- fails the build at once.
# And at least one run of each binary must pass: with a spin or re-exec share
# of about one in four, twelve runs that all miss happen about once in 17
# million builds.  Only when no run passed AND the build's RLIMIT_AS is set (or
# the target is riscv64) AND the misses were the address-space kind (as-limit,
# or a sealed spin, which is what a refused reservation looks like) is the miss
# a property of the machine rather than the toolchain; that goes through
# VERIFY_RUNTIME exactly as the ASan check above does: "require" fails the
# build saying so, "defer" appends what was not verified to
# /opt/asan-static/RUNTIME-UNVERIFIED.
#
# x86_64 and aarch64 MSan does not run under qemu-user at all: on aarch64
# qemu-user puts the guest stack inside MSan's origin range, and both the plain
# and the sealed runs fail.
# So the step first runs a static probe and reads /proc/<pid>/exe from outside
# it: natively that is the probe itself, under binfmt_misc it is the emulator.
# If that link cannot be read at all, the step cannot tell which, and says so
# rather than guess.  In either case nothing is run and nothing is reported as
# passing, and the miss goes through VERIFY_RUNTIME like the others, by one
# helper, deferral(): "require" fails the build saying so, "defer" appends to
# RUNTIME-UNVERIFIED that MSan was not verified.
# riscv64 is the exception: there is no riscv64 runner to build on natively,
# and its layout holds under qemu-user 9.2 and later, which map the program
# above MSan's shadow.  So on riscv64 the step names the emulator and runs the
# binaries under it, and a qemu that does not leave the layout free shows up as
# the address-space misses above.  Measured: qemu 8.1.5 and 8.2.2 (Ubuntu
# 24.04's qemu-user-static) answer the program's first mmap(NULL) with
# 0x2aaaab2ad000, in shadow-3, and under 8.2.2 in a RUN step every plain run
# re-execs and is refused and every sealed run spins; 9.2.0, 9.2.2, 10.0.4,
# 10.0.11, 10.1.3 and 10.2.3 answer at 0x7f..., in app-3, and the binaries run.
# 9.0 and 9.1 were not tried.  armv7, which has no msan runtime, skips this
# step before the probe.
# CI builds aarch64 on a native arm64 runner and riscv64 under the qemu-user
# that docker/setup-qemu-action registers, so CI runs these binaries for real.
#
# Last, a negative link: the clean program against the statically intercepting
# resource dir, with --gc-sections as both MSan configs pass it, and WITHOUT the
# wrap list.  It must fail on __msan_wrap_list_present (patch 0024).  It is
# given the configs' -mllvm line (riscv64's mapping, section 5b; there is none
# elsewhere), or on riscv64 it would stop at the compile and never reach the
# link.
ARG MSAN_RUNS=12
RUN set -eu; \
    [ -f /opt/asan-static/msan-musl.cfg ] || { echo "no msan config for $TRIPLE; no msan test"; exit 0; }; \
    printf '%s\n' \
      '#include <stdio.h>' \
      '#include <stdlib.h>' \
      '#include <string.h>' \
      '#include <sys/prctl.h>' \
      '#include <sys/resource.h>' \
      'int main(void){' \
      '  struct rlimit nocore = {0, 0}; setrlimit(RLIMIT_CORE, &nocore); prctl(PR_SET_DUMPABLE, 0);  /* no core */' \
      '  char *d = malloc(64); memset(d, 0x41, 64);' \
      '  unsigned long s = 0; for (int i = 0; i < 64; i++) s += (unsigned char)d[i];' \
      '  FILE *f = fopen("/tmp/msan_clean.c", "rb");' \
      '  if (f) { char b[32]; size_t n = fread(b, 1, sizeof b, f); for (size_t i = 0; i < n; i++) s += (unsigned char)b[i]; fclose(f); }' \
      '  printf("msan ok %lu\n", s); free(d); return 0;' \
      '}' \
      > /tmp/msan_clean.c; \
    printf '%s\n' \
      '#include <stdio.h>' \
      '#include <stdlib.h>' \
      '#include <sys/prctl.h>' \
      '#include <sys/resource.h>' \
      'int main(void){' \
      '  struct rlimit nocore = {0, 0}; setrlimit(RLIMIT_CORE, &nocore); prctl(PR_SET_DUMPABLE, 0);  /* no core */' \
      '  volatile int *p = (volatile int *)malloc(sizeof(int));  /* uninitialized */' \
      '  int r = *p ? 1 : 2;  /* branch on an uninitialized value: MSan must report */' \
      '  printf("%d\n", r); free((void *)p); return 0;' \
      '}' \
      > /tmp/msan_umr.c; \
    clang --config=/opt/asan-static/msan-musl.cfg        -g /tmp/msan_clean.c -o /tmp/msan_clean; \
    clang --config=/opt/asan-static/msan-musl-sealed.cfg -g /tmp/msan_clean.c -o /tmp/msan_clean_sealed; \
    clang --config=/opt/asan-static/msan-musl.cfg        -g /tmp/msan_umr.c   -o /tmp/msan_umr; \
    clang --config=/opt/asan-static/msan-musl-sealed.cfg -g /tmp/msan_umr.c   -o /tmp/msan_umr_sealed; \
    echo "msan configs link"; \
    mmap=$(grep '^-mllvm ' /opt/asan-static/msan-musl.cfg || true); \
    if clang --target="$TRIPLE" --sysroot="$SYSROOT" -resource-dir="$STATIC_RESOURCE_DIR" \
         --rtlib=compiler-rt --unwindlib=libunwind -fuse-ld=lld -fsanitize=memory $mmap -static \
         -Wl,--gc-sections /tmp/msan_clean.c -o /tmp/msan_detached 2>/tmp/msan_detached.log; then \
      echo "an msan link without the wrap list succeeded; the interceptors are detached" >&2; exit 1; \
    fi; \
    grep -q "__msan_wrap_list_present" /tmp/msan_detached.log \
      || { echo "the msan link without the wrap list failed for another reason:" >&2; cat /tmp/msan_detached.log >&2; exit 1; }; \
    echo "an msan link without the wrap list fails, with --gc-sections"; \
    deferral() { \
      what=$1 why=$2; shift 2; \
      printf '%s\n' "" "$@" "" >&2; \
      if [ "$VERIFY_RUNTIME" != defer ]; then \
        printf '%s\n' "  To build the image anyway:  --build-arg VERIFY_RUNTIME=defer" "" >&2; \
        exit 1; \
      fi; \
      printf '%s\n' "" "MemorySanitizer: $what was not verified when this image was built." "$why" \
        "Nothing has shown that msan-musl.cfg and msan-musl-sealed.cfg work in this" \
        "image.  Run binaries linked with them natively, under an RLIMIT_AS that is" \
        "not small, before trusting them." >> /opt/asan-static/RUNTIME-UNVERIFIED; \
      echo "msan: $what NOT verified: see /opt/asan-static/RUNTIME-UNVERIFIED"; \
    }; \
    printf 'int main(void){for(;;);}\n' > /tmp/msan_probe.c; \
    clang --config=/opt/asan-static/musl.cfg /tmp/msan_probe.c -o /tmp/msan_probe; \
    /tmp/msan_probe & probe=$!; sleep 1; \
    probe_exe=$(readlink "/proc/$probe/exe" || true); kill -9 "$probe"; wait "$probe" 2>/dev/null || true; \
    if [ "$probe_exe" != /tmp/msan_probe ] && [ "${TRIPLE%%-*}" = riscv64 ]; then \
      echo "msan: riscv64 binaries run through ${probe_exe:-a runner this build cannot name}; riscv64's layout needs qemu-user 9.2 or later"; \
    elif [ -z "$probe_exe" ]; then \
      deferral "every run (clean and umr, plain and sealed)" \
        "The build could not read /proc/<pid>/exe of a probe, so it could not tell whether the target's binaries run natively, and none was run." \
        "  This build could not read /proc/$probe/exe of a static probe, so it" \
        "  cannot tell whether the target's binaries run natively or under" \
        "  emulation.  MemorySanitizer's fixed layout does not hold under" \
        "  qemu-user, so none of the MSan binaries was run and whether MSan works" \
        "  in this image has not been tested either way.  The compiles and links" \
        "  succeeded."; \
      exit 0; \
    elif [ "$probe_exe" != /tmp/msan_probe ]; then \
      deferral "every run (clean and umr, plain and sealed)" \
        "The build ran the target's binaries through $probe_exe (binfmt_misc emulation), under which MSan's fixed layout does not hold, so none was run." \
        "  This build runs the target's binaries through $probe_exe" \
        "  (binfmt_misc emulation), not natively.  MemorySanitizer's fixed layout" \
        "  does not hold under qemu-user, so none of the MSan binaries was run and" \
        "  whether MSan works in this image has not been tested either way.  The" \
        "  compiles and links succeeded.  Build on a machine of the target's own" \
        "  architecture to have it checked."; \
      exit 0; \
    fi; \
    ulimit -v unlimited 2>/dev/null || true; \
    vlimit=$(ulimit -v); \
    case "$vlimit:$TRIPLE" in unlimited:riscv64-*) asmiss=va ;; unlimited:*) asmiss= ;; *) asmiss=rlimit ;; esac; \
    reexec_re='MemorySanitizer: CHECK failed: msan_linux\.cpp:[0-9]+ "\(\(personality\(old_personality \| 0x0040000\)\)\) != \(\(-1\)\)"'; \
    aslimit_re='FATAL: MemorySanitizer can not mmap the shadow memory|FATAL: MemorySanitizer: Cannot protect memory range'; \
    msan_check() { \
      label=$1 bin=$2 prog=$3 mode=$4; \
      n=0 pass=0 spin=0 reexec=0 aslim=0; \
      while [ "$n" -lt "$MSAN_RUNS" ]; do \
        n=$((n + 1)); rc=0; \
        timeout -s KILL 10 "$bin" > /tmp/msan_run.log 2>&1 || rc=$?; \
        if [ "$prog" = clean ] && [ "$rc" -eq 0 ] && grep -qx "msan ok [0-9]*" /tmp/msan_run.log; then \
          pass=$((pass + 1)); \
        elif [ "$prog" = umr ] && [ "$rc" -eq "$([ "$mode" = sealed ] && echo 134 || echo 1)" ] \
             && grep -q "WARNING: MemorySanitizer: use-of-uninitialized-value" /tmp/msan_run.log; then \
          pass=$((pass + 1)); \
        elif [ "$mode" = sealed ] && [ "$rc" -eq 137 ] && ! grep -qvx Killed /tmp/msan_run.log; then \
          spin=$((spin + 1)); \
        elif [ "$mode" = plain ] && grep -qE "$reexec_re" /tmp/msan_run.log; then \
          reexec=$((reexec + 1)); \
        elif [ -n "$asmiss" ] && grep -qE "$aslimit_re" /tmp/msan_run.log; then \
          aslim=$((aslim + 1)); \
        else \
          echo "$label: run $n of $MSAN_RUNS ended with status $rc, which is neither a pass nor a known miss:" >&2; \
          cat /tmp/msan_run.log >&2; exit 1; \
        fi; \
      done; \
      echo "$label: $n runs: $pass passed, $spin spun (sealed), $reexec re-exec refused (plain), $aslim address-space limit"; \
      [ "$pass" -gt 0 ] && return 0; \
      miss=$((spin + aslim)); [ "$asmiss" = va ] && miss=$((miss + reexec)); \
      if [ -z "$asmiss" ] || [ "$miss" -eq 0 ]; then \
        echo "$label: no run of $MSAN_RUNS passed, and a constrained address space is not why" >&2; exit 1; \
      fi; \
      if [ "$asmiss" = va ]; then \
        deferral "$label" \
          "The build ran on riscv64 with no RLIMIT_AS and no run of it passed: $spin spun, $aslim could not map the shadow, $reexec re-exec refused." \
          "  No run of $label passed, so whether MemorySanitizer works in it has not" \
          "  been tested either way.  riscv64's MSan layout (patch 0029) needs Sv48" \
          "  hardware, or qemu-user 9.2 or later." \
          "  On Sv39 hardware, or under an older qemu-user, a plain binary finds" \
          "  the layout taken and re-execs or cannot map its shadow, and a sealed" \
          "  one spins on its refused reservation."; \
        return 0; \
      fi; \
      deferral "$label" \
        "The build ran under an RLIMIT_AS of ${vlimit}KB and no run of it passed: $spin spun, $aslim could not map the shadow." \
        "  No run of $label passed, so whether MemorySanitizer works in it has not" \
        "  been tested either way.  This build's address space limit is ${vlimit}KB;" \
        "  MSan's shadow is TBs, a sealed MSan binary spins when its reservation is" \
        "  refused, and a plain one cannot map its shadow.  A container inherits" \
        "  RLIMIT_AS from the daemon: see the ASan check's message above for how to" \
        "  lift it."; \
    }; \
    msan_check clean/plain  /tmp/msan_clean        clean plain; \
    msan_check umr/plain    /tmp/msan_umr          umr   plain; \
    msan_check clean/sealed /tmp/msan_clean_sealed clean sealed; \
    msan_check umr/sealed   /tmp/msan_umr_sealed   umr   sealed; \
    echo "msan: every run passed or missed in a known way"

# The last two steps check sections 4c and 4d, with programs that are static,
# not position-independent and not ASan, like /tmp/hello in the first step, so
# they run wherever that one did, natively or under qemu-user, and they ask
# nothing more of the machine.  A check that fails dies on the target's trap
# instruction, so the status a failed check must give is measured on this
# target first, from a program that calls __builtin_trap(): SIGILL (132) on
# x86_64, riscv64 and 32-bit Arm, and SIGTRAP (133) on aarch64, whose trap is
# brk.  Each program that traps turns off its own core dump first, as the
# std::string step's does.  `expect` runs one program under busybox timeout
# -s KILL (a program stopped there gives 137, which no case expects) and
# requires its exit status and its whole standard output; standard error is
# not read, because qemu-user writes its own line there when a program dies
# by a signal.  Each program reads no input and depends on no address the
# kernel chooses, so one run decides.
#
# The hardened libc++ (4c).  Its __config_site makes extensive the default and
# section 2b's keeps none.  With musl-hardened.cfg, an index past the end of a
# std::span (a check inline in the program) and std::string::find with a null
# pointer, a start past the end and a nonzero length (a check in libc++.a)
# trap after "before", and a correct program that uses strings, a sort, a span,
# a string_view, an optional, a map and a stringstream does not trap.  The
# same two bad programs built with musl.cfg reach "after": the default copy
# checks neither.
RUN set -eu; \
    ulimit -c 0; \
    grep -q "^#define _LIBCPP_HARDENING_MODE_DEFAULT 16$" "$HARDENED_LIBCXX/include/$TRIPLE/c++/v1/__config_site"; \
    grep -q "^#define _LIBCPP_HARDENING_MODE_DEFAULT 2$" "$SYSROOT/usr/include/$TRIPLE/c++/v1/__config_site"; \
    nocore='#include <sys/prctl.h>\n#include <sys/resource.h>\nstatic void nocore(void){struct rlimit r={0,0};setrlimit(RLIMIT_CORE,&r);prctl(PR_SET_DUMPABLE,0);}\n'; \
    printf "$nocore"'#include <stdio.h>\nint main(void){nocore();puts("before");fflush(stdout);__builtin_trap();}\n' > /tmp/trap.c; \
    printf "$nocore"'#include <cstdio>\n#include <span>\nstatic volatile int idx=5;static volatile int sink;\nint main(){nocore();int a[4]={1,2,3,4};std::span<int> s(a,4);std::puts("before");std::fflush(stdout);sink=s[idx];std::puts("after");return 0;}\n' > /tmp/hard-span.cpp; \
    printf "$nocore"'#include <cstdio>\n#include <string>\nstatic const char *volatile p;static volatile unsigned long pos=100,n=5;\nint main(){nocore();std::string s("abc");std::puts("before");std::fflush(stdout);std::printf("after %%ld\\n",(long)s.find(p,pos,n));return 0;}\n' > /tmp/hard-find.cpp; \
    printf '#include <algorithm>\n#include <cstdio>\n#include <map>\n#include <optional>\n#include <span>\n#include <sstream>\n#include <string>\n#include <string_view>\n#include <vector>\nint main(){std::vector<int> v;for(int i=0;i<1000;i++)v.push_back((i*7919)%%1000);std::sort(v.begin(),v.end());std::span<int> s(v);long t=s[999]+s.subspan(10,5)[4];std::string str("hello");for(int i=0;i<200;i++)str+=(char)(97+i%%26);str.replace(2,3,"xyz");str.insert(0,"a");t+=str.compare(0,3,"aab",3)+(long)str.find("xyz")+(long)str.size();std::string_view sv(str);t+=(long)sv.substr(3,4).size()+sv.back();std::optional<int> o=3;t+=*o;std::map<std::string,int> m;m["b"]=2;t+=m.at("b");std::ostringstream os;os<<t;std::istringstream is(os.str());long u=0;is>>u;std::printf("hardened ok %%ld %%ld\\n",t,u);return 0;}\n' > /tmp/hard-ok.cpp; \
    clang --config=/opt/asan-static/musl.cfg /tmp/trap.c -o /tmp/trap; \
    h="clang++ --config=/opt/asan-static/musl-hardened.cfg -stdlib=libc++ -std=c++20 -Os"; \
    d="clang++ --config=/opt/asan-static/musl.cfg -stdlib=libc++ -std=c++20 -Os"; \
    $h /tmp/hard-span.cpp -o /tmp/hard-span; $h /tmp/hard-find.cpp -o /tmp/hard-find; $h /tmp/hard-ok.cpp -o /tmp/hard-ok; \
    $d /tmp/hard-span.cpp -o /tmp/hard-span-default; $d /tmp/hard-find.cpp -o /tmp/hard-find-default; \
    expect() { \
      want_st=$1; want_out=$2; shift 2; st=0; \
      timeout -s KILL 60 "$@" >/tmp/expect.out 2>/tmp/expect.err || st=$?; \
      out=$(cat /tmp/expect.out); \
      if [ "$st" != "$want_st" ] || [ "$out" != "$want_out" ]; then \
        printf '%s: exit %s and stdout [%s], not exit %s and stdout [%s]; stderr:\n' "$*" "$st" "$out" "$want_st" "$want_out" >&2; \
        cat /tmp/expect.err >&2; exit 1; \
      fi; \
      echo "$*: exit $st, as expected"; \
    }; \
    st=0; /tmp/trap >/dev/null 2>&1 || st=$?; \
    case $st in 132|133) ;; *) echo "__builtin_trap() exited $st, not by SIGILL or SIGTRAP" >&2; exit 1 ;; esac; \
    echo "a trap exits $st on this target"; \
    expect "$st" before /tmp/hard-span; \
    expect "$st" before /tmp/hard-find; \
    expect 0 "hardened ok 1352 1352" /tmp/hard-ok; \
    expect 0 "$(printf 'before\nafter')" /tmp/hard-span-default; \
    expect 0 "$(printf 'before\nafter -1')" /tmp/hard-find-default; \
    echo "the hardened libc++ traps on a broken precondition, in the headers and in libc++.a; the default copy does not"

# The vptr handler (4d).  The archive defines the two handlers and the cache
# and nothing else, and has no global constructor.  With ubsan-vptr-musl.cfg, a
# static_cast to a class that is not the object's dynamic type, then a member
# call through it, traps after "before"; so does the same program with the
# release's UBSan flags added (-fsanitize-trap=all, then -fno-sanitize-trap=vptr
# after it, as section 5 says).  A correct program does not trap: casts and
# calls through single, multiple and virtual inheritance, 300 times each, so
# that the handler both misses and fills the 128-slot cache, a dynamic_cast,
# and an exception thrown and caught.  Two links must not work: the same
# vptr flags without the archive fail on the handler's name, --gc-sections
# included, as musl.cfg passes it; and the bad program built with musl.cfg has
# no check and reaches "after".
RUN set -eu; \
    ulimit -c 0; \
    a="$VPTR_TRAP/libclang_rt.ubsan_vptr_trap.a"; \
    syms=$(llvm-nm -g --defined-only "$a" | awk 'NF==3{print $2" "$3}' | LC_ALL=C sort | tr '\n' ';'); \
    [ "$syms" = "B __ubsan_vptr_type_cache;T __ubsan_handle_dynamic_type_cache_miss;T __ubsan_handle_dynamic_type_cache_miss_abort;" ] \
      || { echo "the vptr handler archive defines: $syms" >&2; exit 1; }; \
    ! llvm-readelf -SW "$a" | grep -qE '\.(init_array|ctors)' || { echo "the vptr handler has a global constructor" >&2; exit 1; }; \
    nocore='#include <sys/prctl.h>\n#include <sys/resource.h>\nstatic void nocore(void){struct rlimit r={0,0};setrlimit(RLIMIT_CORE,&r);prctl(PR_SET_DUMPABLE,0);}\n'; \
    printf "$nocore"'#include <cstdio>\nstruct B{virtual ~B(){}virtual int v(){return 1;}};\nstruct D:B{int d=4;int f(){return d;}};\nstruct E:B{int e=5;};\nstatic B *volatile p;\nint main(){nocore();p=new E;std::puts("before");std::fflush(stdout);D *d=static_cast<D *>(p);std::printf("after %%d\\n",d->f());return 0;}\n' > /tmp/vptr-bad.cpp; \
    printf '#include <cstdio>\n#include <memory>\n#include <stdexcept>\n#include <string>\n#include <vector>\nstruct B{virtual ~B(){}virtual int v(){return 1;}};\nstruct D:B{int d=4;int f(){return d;}int v() override{return 2;}};\nstruct M1{virtual ~M1(){}long m1=7;};\nstruct M2{virtual ~M2(){}long m2=8;long g(){return m2;}};\nstruct MI:M1,M2{long h(){return m1+m2;}};\nstruct VB{virtual ~VB(){}int vb=9;};\nstruct V1:virtual VB{int v1=1;};\nstruct V2:virtual VB{int v2=2;};\nstruct VD:V1,V2{int s(){return vb+v1+v2;}};\nstatic B *volatile pb;static M2 *volatile pm;static VB *volatile pv;static V1 *volatile p1;\nint main(){long t=0;for(int i=0;i<300;i++){pb=new D;t+=static_cast<D *>(pb)->f()+pb->v();delete pb;pm=static_cast<M2 *>(new MI);t+=static_cast<MI *>(pm)->h()+pm->g();delete static_cast<MI *>(pm);pv=static_cast<VB *>(new VD);t+=dynamic_cast<VD *>(pv)->s();delete pv;p1=new VD;t+=static_cast<VD *>(p1)->s();delete p1;}\ntry{throw std::runtime_error("x");}catch(const std::exception &e){t+=(long)std::string(e.what()).size();}\nstd::vector<std::unique_ptr<B>> v;for(int i=0;i<4;i++)v.emplace_back(new D);for(auto &q:v)t+=q->v();std::printf("vptr ok %%ld\\n",t);return 0;}\n' > /tmp/vptr-ok.cpp; \
    printf "$nocore"'#include <stdio.h>\nint main(void){nocore();puts("before");fflush(stdout);__builtin_trap();}\n' > /tmp/vtrap.c; \
    clang --config=/opt/asan-static/musl.cfg /tmp/vtrap.c -o /tmp/vtrap; \
    v="clang++ --config=/opt/asan-static/ubsan-vptr-musl.cfg -stdlib=libc++ -Os"; \
    $v /tmp/vptr-bad.cpp -o /tmp/vptr-bad; \
    $v -fsanitize=undefined -fno-sanitize-recover=all -fsanitize-trap=all -fno-sanitize=function -fno-sanitize-trap=vptr /tmp/vptr-bad.cpp -o /tmp/vptr-bad-ubsan; \
    $v /tmp/vptr-ok.cpp -o /tmp/vptr-ok; \
    clang++ --config=/opt/asan-static/musl.cfg -stdlib=libc++ -Os /tmp/vptr-bad.cpp -o /tmp/vptr-bad-plain; \
    if clang++ --config=/opt/asan-static/musl.cfg -stdlib=libc++ -Os -fsanitize=vptr -fno-sanitize-trap=vptr \
         -fno-sanitize-recover=vptr -fno-sanitize-link-runtime /tmp/vptr-bad.cpp -o /tmp/vptr-nohandler 2>/tmp/vptr-nohandler.log; then \
      echo "a vptr link without the handler succeeded" >&2; exit 1; \
    fi; \
    grep -q "undefined symbol: __ubsan_handle_dynamic_type_cache_miss_abort$" /tmp/vptr-nohandler.log \
      || { cat /tmp/vptr-nohandler.log >&2; exit 1; }; \
    expect() { \
      want_st=$1; want_out=$2; shift 2; st=0; \
      timeout -s KILL 60 "$@" >/tmp/expect.out 2>/tmp/expect.err || st=$?; \
      out=$(cat /tmp/expect.out); \
      if [ "$st" != "$want_st" ] || [ "$out" != "$want_out" ]; then \
        printf '%s: exit %s and stdout [%s], not exit %s and stdout [%s]; stderr:\n' "$*" "$st" "$out" "$want_st" "$want_out" >&2; \
        cat /tmp/expect.err >&2; exit 1; \
      fi; \
      echo "$*: exit $st, as expected"; \
    }; \
    st=0; /tmp/vtrap >/dev/null 2>&1 || st=$?; \
    case $st in 132|133) ;; *) echo "__builtin_trap() exited $st, not by SIGILL or SIGTRAP" >&2; exit 1 ;; esac; \
    expect "$st" before /tmp/vptr-bad; \
    expect "$st" before /tmp/vptr-bad-ubsan; \
    expect 0 "vptr ok 15909" /tmp/vptr-ok; \
    expect 0 "$(printf 'before\nafter 5')" /tmp/vptr-bad-plain; \
    echo "a bad downcast traps through the vptr handler; a correct program does not; a link without the handler fails"

ENV MUSL_SYSROOT=$SYSROOT \
    ASAN_RESOURCE_DIR=$RESOURCE_DIR
