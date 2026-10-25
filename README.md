# static-asan

[![CI](https://github.com/theorem-labs/static-asan/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/theorem-labs/static-asan/actions/workflows/ci.yml?query=branch%3Amain)

AddressSanitizer (ASan) for fully static binaries against musl libc.

A binary built with this toolchain has no `PT_INTERP`, no `DT_NEEDED` and no
dynamic symbol table. It still gives ordinary ASan reports. Upstream LLVM does not
support this link. `clang -fsanitize=address -static` fails with
`undefined reference to '_DYNAMIC'`, because the runtime intercepts libc functions
through `dlsym` at run time, and a static binary has nothing for `dlsym` to search.

This repository contains two things:

- `patches/`: a patch series against musl and LLVM's compiler-rt.
- `toolchain/Dockerfile`: a Docker build that applies the series to pinned upstream
  sources and produces a clang toolchain with a static ASan runtime.

In the static runtime, the linker attaches each interceptor with `--wrap=<func>`.
The runtime does not use `dlsym`.

## Provenance

This is asan-static commit `385966a2672b4be07efec847baee1dd5e497b25f`, the toolchain for the
binaries in Theorem Labs' OSS-Fuzz proofs: asan-static `a408805db578be529ff2923ddee5a608ba9753d2`
plus musl patch 0003, which keeps a static program's TLS block in `.bss` and replaces musl
patch 0002. It is a fresh snapshot of that commit's `toolchain/` and `patches/`, without
asan-static's history.

The snapshot differs from asan-static in three ways:

- The `From:` line of each patch names Theorem Labs.
- Internal references removed from commit messages and from source comments. The
  comment edits keep the number of lines the same, so no line number in the patched
  sources moves.
- `toolchain/Dockerfile` pins the Alpine packages by series (`~x.y`) instead of exactly
  (`=x.y.z-rN`), with a comment that says why (see "Pinned versions"). Since
  2026-10-25 the packages it fetches for the target architecture (section 0a of the
  Dockerfile) also accept any Alpine build number of their pinned version, because
  Alpine replaced `musl-1.2.5-r12` with `r13` and the exact pin stopped the build.

Both patch series apply with `patch -p1 --fuzz=0` to the pinned bases. The source
trees that result from the two series differ in six comment lines of two files, and in
nothing else: five in `compiler-rt/lib/asan/asan_new_delete.cpp` (patch 0023) and one in
`compiler-rt/lib/sanitizer_common/sanitizer_sealed.h` (patch 0019).

The builds of a408805d identified the toolchain by a hash over `toolchain/` and
`patches/`:

```sh
find toolchain patches -type f | LC_ALL=C sort | xargs sha256sum | sha256sum
```

| Tree | Hash |
| --- | --- |
| vendored copy of a408805d that built the earlier binaries | `6c1cdc74b5df1be13f0ccd59bea4c9897e6cf34a7df279f867c65c1624ba8c83` |
| this repository at a408805d | `b0faa34e5a463da1ed884c3d4d0e8ef26a7829c5ad91e00517bcbd3994e3b36d` |

The two a408805d trees differ only in comments and patch headers. To show that the
difference does not change the toolchain, we built the riscv64 image from both
trees. We built this repository from a clean clone of its own commit. Then we
compared the files that the three `/opt/asan-static/*.cfg` configs give to the
linker. The `clang -###` link line of each config gave the list of files.

- 8 of the 10 linked files are byte-identical. These include
  `libclang_rt.asan.a.wrap` and `libclang_rt.builtins.a`.
- `libclang_rt.asan.a` differs in 18 of its 121 members.
  `libclang_rt.asan_cxx.a` differs in its one member, `asan_new_delete.cpp.o`.
- In each of these 19 objects, the only section that differs is `.debug_line`.
  After `llvm-objcopy --strip-debug`, the objects are byte-identical.
- In `.debug_line`, only the DWARF 5 MD5 checksums of source files differ. The
  line-number program is the same, because the comment edits keep the line
  count of each file.
- The checksums that differ are those of `sanitizer_sealed.h` (patch 0019) in
  all 19 objects, and of `asan_new_delete.cpp` (patch 0023) in
  `asan_new_delete.cpp.o`.

Other files under `/opt` also differ, but the static configs do not link them.
They are the shared `.so` runtimes, and the archives of the stock resource
directory and of the other sanitizers. We did not analyse them.

The build records of the earlier binaries give the md5 of the runtime archive they
linked, `/opt/asan-static-rt/lib/clang/20/lib/riscv64-unknown-linux-musl/libclang_rt.asan.a`:
`91ba024da5ddacd96dc1c7588e5763b3`. A rebuild of the vendored a408805d copy with the
riscv64 command under "Building" gives that md5. A build of this repository at a408805d
gives `4880198ff493a316727c4cc1d4f78e21`, the same on a second build without the Docker
cache; the archives differ only in `.debug_line`, as described above.

Musl patch 0003 changes `libc.a`, not the runtime. A build of asan-static 385966a2 itself
gives the same `libclang_rt.asan.a`, md5 `91ba024da5ddacd96dc1c7588e5763b3`, and a build of
this repository gives the same `4880198ff493a316727c4cc1d4f78e21` as at a408805d. Against
the asan-static 385966a2 build, this repository's musl sysroot (`libc.a`, the start-up
objects, libunwind, libc++ and libc++abi: 21 files) is byte-identical, and the runtime
archives differ only in `.debug_line`, as above.

The Alpine packages these builds installed are the ones the `~series` pins resolved to
on the day of the build (see "Pinned versions").

## Pinned versions

| Input | Pin |
| --- | --- |
| musl | `f21a96538f78fa8e2040831b4209b35f2fb581da` (master, `v1.2.6-20-gf21a9653`) |
| llvm-project | `87f0227cb60147a26a1eeb4fb06e3b505e9c7261` (`llvmorg-20.1.8`) |
| base image | `alpine:3.22@sha256:14358309a308569c32bdc37e2e0e9694be33a9d99e68afb0f5ff33cc1f695dce` (3.22.5, published 2026-06-22) |
| clang, lld, llvm (Alpine packages) | `~20.1.8`, the same release as the compiler-rt sources |
| other Alpine packages | `cmake~3.31`, `ninja-build~1.12`, `build-base~0.5`, `linux-headers~6.14`, `git~2.49`, `python3~3.12`, `patch~2.8`, `binutils-<arch>~2.42` and others, as listed in `toolchain/Dockerfile` |

The base image is pinned by digest. The Alpine packages are pinned by series only.
Alpine keeps only the newest build of each package in a branch, so an exact pin stops
resolving when Alpine ships an update. A `~series` pin always resolves, but a later
build can get a newer package build within the same series.

## Building

Get the two upstream sources. The build reads them at the pinned commits with
`git archive`, so the state of your working trees does not matter.

```sh
git clone https://git.musl-libc.org/git/musl musl
git clone https://github.com/llvm/llvm-project llvm-project
```

Build the image with Docker BuildKit:

```sh
docker build -f toolchain/Dockerfile \
  --build-context musl-src=$PWD/musl \
  --build-context llvm-src=$PWD/llvm-project \
  --build-context patches=$PWD/patches \
  -t tc-asan-musl toolchain
```

Three build arguments select the target. They default to aarch64.

| Target | `TRIPLE` | `MUSL_TARGET` | `APK_ARCH` |
| --- | --- | --- | --- |
| aarch64 | `aarch64-unknown-linux-musl` | `aarch64-linux-musl` | `aarch64` |
| x86_64 | `x86_64-unknown-linux-musl` | `x86_64-linux-musl` | `x86_64` |
| riscv64 | `riscv64-unknown-linux-musl` | `riscv64-linux-musl` | `riscv64` |
| 32-bit Arm | `armv7-unknown-linux-musleabihf` | `armv7-linux-musleabihf` | `armv7` |

The OSS-Fuzz proofs' binaries use this riscv64 configuration:

```sh
docker build -f toolchain/Dockerfile \
  --build-context musl-src=$PWD/musl \
  --build-context llvm-src=$PWD/llvm-project \
  --build-context patches=$PWD/patches \
  --build-arg TRIPLE=riscv64-unknown-linux-musl \
  --build-arg MUSL_TARGET=riscv64-linux-musl \
  --build-arg APK_ARCH=riscv64 \
  --build-arg TARGET_CFLAGS="-march=rv64g -mabi=lp64d -ffunction-sections -fdata-sections" \
  --build-arg VERIFY_RUNTIME=defer \
  -t tc-asan-rv64g-musl toolchain
```

`TARGET_CFLAGS` goes into every component that a link includes: musl, libunwind,
both compiler-rt builds and the clang config files. On riscv64 the linker combines
the ISA attributes of all inputs. Thus one `rv64gc` object in the link sets the
compressed-instruction flag on the output. The Dockerfile explains this in more
detail.

`VERIFY_RUNTIME=defer` skips the last check of the build, which runs a static ASan
binary. That binary reserves about 14TiB of shadow memory on a 64-bit target. If the
Docker daemon sets an `RLIMIT_AS` that is smaller, the binary cannot start. With
`defer`, the image contains `/opt/asan-static/RUNTIME-UNVERIFIED`, which records that
the check did not run.

### Using the image

Inside the image, one command compiles and links a static ASan binary:

```sh
clang --config=/opt/asan-static/asan-musl.cfg prog.c -o prog
```

The image installs these clang config files in `/opt/asan-static/`:

| Config | Link |
| --- | --- |
| `musl.cfg` | static musl, no sanitizer |
| `asan-musl.cfg` | static musl with the static ASan runtime |
| `asan-musl-sealed.cfg` | `asan-musl.cfg` with sealed init (patch 0012) |
| `asan-musl-silent.cfg` | `asan-musl.cfg` with silent reports (patch 0014) |
| `asan-musl-dynamic.cfg` | ordinary dynamic ASan with the stock runtime |
| `asan-musl-nowrap.cfg`, `asan-musl-bfd.cfg` | two incorrect static links, for tests that require the link to fail |

The sealed and silent configs use the same runtime archive as `asan-musl.cfg`. Each
adds one `--defsym` to the link.

## Cross-building

The image runs on the build machine and compiles for the target. The base image is
pinned to `$BUILDPLATFORM`, and the three arguments above name the target. You do not
need target hardware or an emulator to build the image.

Three verification steps at the end of the Dockerfile run target binaries. On a cross
build, the build machine must be able to run them. On Linux, register `qemu-user`
with `binfmt_misc` for the target, for example with the `qemu-user-static` package or
with `docker run --privileged tonistiigi/binfmt --install <arch>`. Only these three
steps need the emulator.

`--build-arg TOOLCHAIN_PLATFORM=linux/<arch> --platform linux/<arch>` builds the
image on the target architecture under emulation instead. This gives the same
toolchain but is much slower. On one 144-core x86_64 host, a cold riscv64 build took
16m42s under emulation and 3m14s as a cross build.

## Running an emulated image

Two properties of `qemu-user` affect programs that you build for another
architecture and run under emulation:

- An emulated program that runs as PID 1 and gets a fatal signal can hang instead of
  exit. Run it with `docker run --init`, so that a separate init process is PID 1.
- LeakSanitizer cannot run under `qemu-user`. LSan stops the world with a
  `CLONE_VM|CLONE_UNTRACED` clone and `ptrace`, and `qemu-user` supports neither.
  This is a limit of the emulator, not of this toolchain.

ASan on riscv64 also needs a large address space. The primary allocator reserves
128GB, and the shadow reservation is larger. Give the container an `RLIMIT_AS` that
is not small.

## Binary size

The static configs pass `--gc-sections`, and the runtimes are built with
`-ffunction-sections -fdata-sections`. Patch 0018 gives each interceptor trampoline
its own section. The second resource directory ships no `libclang_rt.*.syms` files,
so the link has no dynamic list. Together, these changes let the linker remove
interceptors that the program does not reference.

Measured on riscv64 with `-march=rv64g -mabi=lp64d`, stripped:

| Config | hello world | C++ program (iostream, map, thread, regex, fstream) |
| --- | --- | --- |
| `musl.cfg`, no sanitizer | 31KB | 771KB |
| `asan-musl.cfg` before these changes | 1500KB | 2453KB |
| `asan-musl.cfg` | 552KB | 1407KB |

The wrap list did not change. Every call that the program or the libc makes goes
through the same interceptor as before.

## The patch series

`toolchain/Dockerfile` applies the patches in sorted order with
`patch -p1 --fuzz=0`. Each patch explains its change in its commit message.
`patches/README.md` tells how to regenerate them.

| Patch | Change |
| --- | --- |
| `musl/0001` | Exports the size of `struct pthread`, so that the leak checker can scan the thread descriptor. |
| `musl/0003` | A static program's TLS block always lives in `builtin_tls`, 131072 bytes of `.bss`, so start-up maps nothing before `__asan_init`; a block that does not fit stops start-up with one line on fd 2 and a crash. Replaces `musl/0002`, which checked the result of the start-up TLS `mmap`. |
| `compiler-rt/0001` | Adds `COMPILER_RT_STATIC_INTERCEPTION`: interceptors become `__wrap_<func>`, and `REAL()` comes from `__real_<func>` at link time. |
| `compiler-rt/0002` | Generates the `--wrap` list (`libclang_rt.asan.a.wrap`) and makes a link without it fail. |
| `compiler-rt/0003` | Extends the TLS range that LSan scans over musl's thread descriptor. |
| `compiler-rt/0004` to `0007` | Describe musl's `msghdr` and `cmsghdr`, intercept the names musl defines, enable interceptors that were gated on glibc, and read `gethostbyaddr`'s length as `socklen_t`. |
| `compiler-rt/0008` to `0011` | Corrections for 32-bit musl targets, where `time_t` is wider than `long`: time types, `shmid_ds`, `internal_stat`, `mktime`. |
| `compiler-rt/0012` | Sealed init, enabled by `__sanitizer_sealed_init` at link time: the runtime reads no environment and no `/proc`, installs no signal handlers, and dies by `SIGABRT` on a detection. |
| `compiler-rt/0013` | Sizes the riscv64 mmap range by the window that an unhinted `mmap` uses. |
| `compiler-rt/0014` | Silent reports, enabled by `__sanitizer_silent_reports`: the runtime detects and dies as usual but writes nothing. |
| `compiler-rt/0015` | Sealed: an allocation that the runtime cannot get for itself spins forever with no syscall. See "Known limitation". |
| `compiler-rt/0016`, `0017` | Sealed: applies the 0015 rule to the remaining refusal sites, including an arena reservation that came up at address 0. |
| `compiler-rt/0018` | One section per interceptor trampoline, so that `--gc-sections` can remove unused interceptors. |
| `compiler-rt/0019` | Sealed: reserves the shadow first with `MAP_FIXED_NOREPLACE`. On riscv64, reads the address-space size from the initial stack frame, so one runtime supports Sv39 and Sv48. |
| `compiler-rt/0020` | Sealed: issues `madvise(MADV_DONTNEED)` again, so that freed shadow pages go back to the kernel. |
| `compiler-rt/0021` | Makes lld extract the real libc function at parse time, so that a weakly referenced function is not left null. |
| `compiler-rt/0022` | Sealed: repairs the arena reservation after a refused commit, and returns the program's NULL. |
| `compiler-rt/0023` | Sealed: the throwing forms of `operator new` throw `std::bad_alloc` when the allocation is refused. |

## Known limitation: the sealed runtime spins when it cannot get memory

In a binary linked with `asan-musl-sealed.cfg`, patch 0015 changes what happens when
the runtime cannot get memory for itself. Examples are a refused `mmap`, a shadow
range that it cannot reserve, and an internal allocator with no space left. The
runtime does not report and does not exit. It spins forever on the current frame and
makes no further syscalls. Patches 0016, 0017, 0019 and 0022 use the same spin.

The spin keeps the syscall trace of the sealed binary a prefix of the trace of the
same program built without the sanitizer. This is the property that the
OSS-Fuzz proofs need. The cost is that a process in this state runs until
something outside it stops it. It uses CPU and gives no diagnostic. Use a timeout
when you run sealed binaries.

These cases are not affected:

- An allocation failure while an error report is in progress aborts with `SIGABRT`,
  so a detection does not disappear.
- A `malloc` that the program cannot get stays the program's result: an ASan report,
  or NULL with `allocator_may_return_null=1`.
- Binaries that are not sealed behave as upstream ASan does.

Later versions of asan-static replace the spin with `SIGABRT`. This snapshot does not
include that change, because the OSS-Fuzz proofs' binaries were built without it.

## CI

The [CI](.github/workflows/ci.yml) workflow builds the image for each of the four
targets on each push to `main`, on each pull request and once a week. The last steps of
`toolchain/Dockerfile` are the tests. A static musl binary runs, a static ASan binary
reports an overflow that only a live interceptor can report, and a link without the
wrap list fails. The cross targets run these steps under `qemu-user`. After the
build, CI links a program with `asan-musl.cfg` and checks that the binary has no
`PT_INTERP`, no `DT_NEEDED` and no dynamic symbol table. riscv64 is built with the
arguments of the OSS-Fuzz proofs' toolchain, except that `VERIFY_RUNTIME` stays
`require`. Runs reuse Docker's layer cache from earlier runs on `main`; the weekly run
uses no layer cache and rebuilds everything.

## Not included

Upstream asan-static also has an end-to-end test suite (`tests/`) and conformance
checks (`conformance/`) that compare compiler-rt's description of musl with musl's
headers. They are not part of this snapshot. Some comments and commit messages refer
to them, and to test case names in them.

Compiler-rt patches 0013, 0014, 0015, 0016, 0022 and 0023 have an all-zero commit hash
in their first `From` line, and `musl/0003` has no `From <hash>` line at all: its header
starts at `From: Theorem Labs`. These patches were edited by hand and not regenerated.
`patch` ignores these header lines.

## License

MIT for Theorem Labs' work: the toolchain Dockerfile, the READMEs and the changes
the patches make; see `LICENSE`. The code the patches change keeps its own licence:
LLVM's compiler-rt is under the Apache License v2.0 with LLVM Exceptions
(`LICENSE-LLVM.txt`), and musl is under the MIT License (`LICENSE-musl.txt`).
