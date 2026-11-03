# Patch series

What `toolchain/Dockerfile` applies with `patch -p1 --fuzz=0` to trees taken from the
pinned base commits, in sorted order: two musl patches (0001 and 0003; 0002 is retired) and twenty-nine against compiler-rt. What
each does is in the table in the top-level [README](../README.md) and, at length, in each
patch's own commit message. `--fuzz=0` is deliberate -- after a base bump a hunk that no
longer matches must fail rather than apply at a plausible-looking offset.

Bases: musl `f21a9653` (master, 20 commits past v1.2.6) and llvm-project `87f0227cb`
(the `llvmorg-20.1.8` release commit).

Regenerate after editing the checkouts, which are both on a branch named `asan-static`:

```sh
git -C llvm-project format-patch -o patches/compiler-rt --no-signature -N 87f0227cb..HEAD
git -C musl        format-patch -o patches/musl        --no-signature -N f21a9653..HEAD
```

then rename to the short names already here. Regenerating rewrites the commit hash in the
`From` line of every patch after the one that changed, so patches nothing was done to
still show up in the diff.
