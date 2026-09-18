# Armbian rootfs cache — arm64 / noble / cli

Prebuilt Armbian **rootfs artifact**, published as a Release asset so board image
builds can skip the debootstrap stage (20-40 minutes each).

This repo deliberately contains **no board overlay** — no `patch/`, no kernel or
U-Boot patches. See below for why.

---

## Why this is a separate repo

The rootfs cache name is

```
rootfs-<ARCH>-<RELEASE>-<cache_type>_<yyyymm>-<AGGREGATED_ROOTFS_HASH>-H<hooks>-B<bash>.tar.zst
```

It carries **no board name**. Measured on the same `armbian/build` revision:

| how it was built | computed cache id | `cache_type` |
|---|---|---|
| `BOARD=hz-evm-rk3588` (with the board repo's overlay) | `de3dd0bda694-H6eccde-Bf1b6db` | `cli` |
| `BOARD=rock-5b` (stock Armbian, no overlay at all) | `de3dd0bda694-H6eccde-Bf1b6db` | `cli` |

Identical. A rootfs build never reads `patch/kernel/` or `patch/u-boot/`, and even
a board's `PACKAGE_LIST_BOARD` lands in the **image**-level package set
(`AGGREGATED_PACKAGES_IMAGE`), not the rootfs one. So the rootfs is a generic
arm64 artifact, not a board artifact, and it does not belong in the board repo.

The single `config/boards/hz-evm-rk3588.conf` kept here exists **only** so the
`BOARD` / parameter surface matches the image repo exactly. Nothing in it affects
the cache id today.

---

## What invalidates a published tarball

| segment | changes when |
|---|---|
| `rootfs-arm64-noble-cli` | `RELEASE`, `BUILD_MINIMAL` or desktop selection changes. **`cli` and `minimal` are different names and do not share a cache.** |
| `yyyymm` | the calendar month rolls over |
| `AGGREGATED_ROOTFS_HASH` | upstream changes the aggregated package list |
| `H<hooks>` | the `custom_apt_repo` extension hook or `DEST_LANG` changes |
| `B<bash>` | a hash of **`lib/functions/rootfs/create-cache.sh`** + **`rootfs-create.sh`** only |

`B<bash>` is why `ARMBIAN_BUILD_SHA` is **pinned** instead of tracking `main`:
the pin keeps it and the package-list hash stable, so a tarball built today is
still valid next week. **Bumping the SHA means you must re-run this workflow.**

The monthly `cron` exists for the `yyyymm` segment — a tarball built in September
can never be reused in October.

---

## Consuming it

In the board repo's image build, before `compile.sh build`:

```bash
mkdir -p armbian-build/cache/rootfs
gh release download armbian-rootfs-cache \
  --repo iorca/Armbian-rootfs-dev \
  --pattern '*.tar.zst' \
  --dir armbian-build/cache/rootfs \
  --clobber
```

Note `--repo`: the asset lives here, not in the board repo. The download step
should be `continue-on-error`, because a missing Release must degrade to
"Armbian rebuilds the rootfs" and never break the build.

Requirements for a cache hit on the consumer side:

1. same `armbian/build` revision (**`04108a20e5de1c956425b3f5a2a7cba47ba829b8`**)
2. `RELEASE=noble`
3. `BUILD_MINIMAL=no` (full CLI) and `BUILD_DESKTOP=no`
4. built in the same calendar month

A mismatch is harmless: the file just sits unused and the rootfs is built from
scratch.

---

## Refreshing it

Normally automatic — the monthly cron re-runs it. To do it by hand:

**Actions → Armbian rootfs cache → Run workflow** (about 20-40 minutes, then it
publishes to the `armbian-rootfs-cache` Release, overwriting the old asset).

### Seeding the Release by hand

If you already have a tarball whose name matches the id the consumer computes,
skip the build: create a Release tagged `armbian-rootfs-cache` and upload it.
The file name is the whole contract, so it must not be renamed. For the pinned
revision above, the expected name is:

```
rootfs-arm64-noble-cli_202609-de3dd0bda694-H6eccde-Bf1b6db.tar.zst
```

(232.6 MB, `sha256 05a09edc7950bb9b9ec709cd61f506b896ef12a0c3999029c2e69c4be2581fcc`)
