# debian-bootc-core

> **Note:** This repo was created primarily using directed AI, though its
> contents have been manually tested and inspected where possible. I
> believe it's important for anyone using open-source tools on GitHub to
> have this context before relying on them. This image is still in
> testing — the goal is a simple, minimal Debian bootc image that's easy
> to fork and use as-is. KDE and GNOME desktop flavors are planned
> alongside this core image, following the same approach. Thanks to the
> upstream repository
> [frostyard/debian-bootc-core](https://github.com/frostyard/debian-bootc-core)
> for the foundational work this was forked from.

A [Debian](https://www.debian.org/) container image preconfigured for
[bootc](https://github.com/bootc-dev/bootc) usage. This is a forked
repository — see [Updating Installed Systems From Your
Repo](#updating-installed-systems-from-your-repo) if you're building your
own fork.

## Goal

Use this repo as your own bootc image source, build it, boot it in a VM
(or install it to bare metal), and later update installed systems with
`bootc switch`/`bootc upgrade`.

*Unlike a traditional Linux distribution where you install packages onto a
live system, you manage this system by editing the `Containerfile`,
building a new container image, and instructing your host to boot from
that image.*

> ⚠️ **First boot:** The base Debian image ships with no usable root
> password and no other account, so booting it as-is leaves you with no
> way to log in. See [Quick Start](#quick-start) below — it injects an
> account at disk-image build time via `config.toml`.

## Prerequisites

- Linux host with `podman`, `qemu-img`, `just`, `gh`
- [`incus`](https://linuxcontainers.org/incus/) for the built-in VM launch
  recipe (`just launch-incus`) — or use `virt-install`/`libvirt` directly
  with the disk image produced below
- Optional, for image signing: `cosign`

> **Note:** This project uses `just` as a command runner. Run `just` with
> no arguments to list all recipes, or inspect the `Justfile` to see the
> underlying `podman`/`bootc-image-builder` commands being run.

---

## Quick Start

This builds a bootable qcow2 disk image from the published GHCR image using
[`bootc-image-builder`](https://github.com/osbuild/bootc-image-builder),
injecting the account(s) defined in `config.toml`.

### 1. Set up your account

Edit [`config.toml`](config.toml) and uncomment the user block(s) you want.
At minimum, set a real password in place of `changeme`:

```toml
[[customizations.user]]
name = "root"
password = "changeme"
```

`bootc-image-builder` accepts either a plaintext password (hashed
automatically at build time) or a pre-hashed one (`openssl passwd -6`) —
if it starts with `$6$`, `$5$`, or `$2b$` it's treated as already hashed.
To create a regular admin user instead of enabling root, add them to the
`sudo` group instead (see the commented example in the file).

If the GHCR package is private, authenticate first:

```bash
sudo podman login ghcr.io
```

### 2. Build the disk image

```bash
just disk-image-from-ghcr
```

This writes `output/qcow2/disk.qcow2`. Resize it first if you want more
disk space:

```bash
qemu-img resize output/qcow2/disk.qcow2 100G
```

### 3. Boot it

The repo's `launch-incus` recipe expects `debian-bootc-core.img` in the
current directory (already gitignored), so point it at the image you just
built:

```bash
cp output/qcow2/disk.qcow2 debian-bootc-core.img
just launch-incus
```

Log in with the account you defined in `config.toml`.

---

## Building Locally Instead

### 1. Fork or clone

```bash
gh repo fork Danathar/debian-bootc-core --clone=false
git clone https://github.com/<your-user>/debian-bootc-core.git
cd debian-bootc-core
```

### 2. Build the container image

```bash
just build-container
```

### 3. Build the disk image from your local build

```bash
just build-disk-image
```

This uses the same `config.toml` and writes to `output/qcow2/disk.qcow2`,
same as [Quick Start](#quick-start) step 2. Continue from step 3 above to
boot it.

---

## Updating Installed Systems From Your Repo

Once installed, switch to your image and reboot:

```bash
sudo bootc switch ghcr.io/<your-user>/debian-bootc-core:latest
sudo reboot
```

Members of the `sudo` group can already run `bootc status`, `bootc
update`, and `bootc upgrade` without a password (see
`/etc/sudoers.d/001-bootc`); switching to a different image reference
still needs a full `sudo`.
