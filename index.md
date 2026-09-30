---
layout: default
title: "Getting Your App on RHEL Image Mode: What You Need to Know"
---

If your product ships to customers as a host RPM or an installer (agents, monitoring tools, security scanners, drivers, enterprise applications) and they are adopting RHEL image mode (bootc), this guide is for you. The message up front: what you need to change might be less than you think. Most of what you already ship works unchanged. The walls are a short list, clearly marked, and each one has a standard fix. Most of those fixes live in the image build your customer already runs, and a smaller set needs a change only you can make. [How much work is this?](#how-much-work-is-this) sorts them, so you can tell early which one you are behind. You don't need to become a bootc expert. You need to know where the walls are.

If your product is already container-native, you can skip most of this guide. Start with [the first question](#first-question-does-it-need-to-be-in-the-os-image-at-all) and stop there.

Why this guide exists: upstream bootc docs are written for the people who build OS images. RHEL docs are written for the people who run the systems. This guide is for the vendor whose software ends up inside an image someone else builds.

**Version 0.75, written against RHEL 10 image mode and bootc as of 2026-09-29.**

A handful of specifics are still being checked against a live system or a real build rather than documentation alone. See [open questions]({{ '/open-questions/' | relative_url }}) for what's unverified and how to help settle it.

## How image mode works

On image mode, the operating system ships as a container image. The customer builds it with a Containerfile, the same way they build application containers, starting from a Red Hat base image. An install step writes that image to disk once, and from then on the OS content on the running system is the image's content, read-only. How it gets onto disk (a disk image from bootc image builder, a kickstart, or `bootc install`) is the customer's concern and does not touch your software. Updates are atomic: bootc pulls a newer image, stages it beside the current one, and the machine switches to it on reboot, and it can roll back the same way. Your software becomes one layer of that image, installed during that build.

Where to go deeper:

- [Image mode for RHEL](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux-10/image-mode): Red Hat's product page for image mode. The shortest orientation to what Red Hat is shipping and how it is positioned, which is often what a colleague actually wants when they ask you what this is.
- [bootc project](https://github.com/bootc-dev/bootc): never seen bootc? Start with the project README for the what and the why.
- [bootc filesystem docs](https://github.com/bootc-dev/bootc/blob/main/docs/src/filesystem.md): the full version of the filesystem model summarized in the table below.
- [bootc building guidance](https://github.com/bootc-dev/bootc/blob/main/docs/src/building/guidance.md): upstream's Containerfile patterns for adapting packages; the closest upstream counterpart to this guide.
- [RHEL 10 image mode documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/index): the full RHEL reference for building, deploying, and managing image mode systems.

## The one thing to understand first

RHEL image mode has a different filesystem model than traditional RHEL. Once you internalize this, everything else follows:

| Path | Traditional RHEL | Image Mode |
|------|-----------------|------------|
| `/` (root), `/usr`, `/opt`, `/usr/local` | Read-write | **Read-only** (ships with the image) |
| `/etc` | Read-write, RPM-managed | Read-write, but merges differently on upgrades |
| `/var` | Read-write | **Read-write** (the main place for persistent runtime data) |
| `/run` | tmpfs | tmpfs (same as traditional) |
| `/tmp` | Disk-backed | **tmpfs** (RAM-backed, 50% of system memory, empty after reboot) |

That's the headline: **read-only root, read-write `/var` and `/etc`.** Your binaries, libraries, and default configs ship in the image. Your runtime data, logs, and caches go in `/var`. Local config customization goes in `/etc`.

## First question: does it need to be in the OS image at all?

Before adapting your RPM, ask a more basic question: does your software need to be part of the OS image, or can it run as a container on top of it? Image mode systems run containers natively with Podman, and for agents, scanners, and monitoring tools, a container is often the simpler path. Two patterns:

- **Quadlets.** Ship your agent as a container image plus a small systemd unit file (a Quadlet `.container` file). systemd starts it at boot like any service. You release container images on your own cadence, and the host filesystem rules below stop applying to your code.
- **Logically bound images.** The OS image references your container image, so it is pulled and lifecycled together with the host image while still updating on its own cadence. Upstream's stated use cases are logging, monitoring, configuration management, and security agents: exactly the software this guide is written for. See [logically bound images](https://github.com/bootc-dev/bootc/blob/main/docs/src/logically-bound-images.md).

If your software can run as a container, most of this guide stops applying to you. The catalog below is for software that has to live on the host: drivers and kernel modules, host-level tooling, or anything that genuinely can't be containerized.

## How much work is this?

The patterns in this guide are not equally expensive, and the difference that matters most is not how technical the fix is. It is how much of the fix is yours to make, and how much your customer can do in their own image build with what you ship today. Four buckets, in rising order of your involvement. Each ends with something you publish, because a path you have documented is different from one a customer found to work. Software that can run as a container instead is [the first question](#first-question-does-it-need-to-be-in-the-os-image-at-all), and it is worth answering before any of the rest.

**Bucket 1: nothing changes.** A `.repo` file, your GPG key, a `RUN dnf install` or `RUN ./install.sh`, and you are done: plain files, nothing about your software changes. Most software lands here. Confirm it rather than assume it: run `bootc container lint` against a test build and read your scriptlets with `rpm -qp --scripts`. The silent failure in [the build environment](#the-build-environment-is-a-container-not-a-booted-system) leaves your service unenabled without failing the build: a green build is not by itself evidence. What you ship is the statement: "installs with `dnf install`, supported on RHEL image mode" is the whole of it.

**Bucket 2: a workaround the customer can apply.** Your software is fine, but the build has to be adjusted, and the customer can make the adjustment in their own Containerfile with what you ship today. The common case is an `/opt` tree mixing read-only content with directories your software writes to: logs, caches, a definitions database. Move those to `/var`, symlink them back, and ship `tmpfiles.d` entries so the targets exist on every machine, not only newly provisioned ones. See [`/opt`](#opt-is-read-only-at-runtime). Customers work this out on their own all the time. What you ship is the snippet, documented, so the workaround is the supported path.

**Bucket 3: a build recipe only you can write.** You distribute a kernel module as source. The customer can run a multistage build, but the recipe has to come from you: a builder stage with `kernel-devel` pinned to the image's kernel, the module built against it, and the result installed in the final stage. See [Kernel modules](#kernel-modules-build-against-the-images-kernel). What you ship is that recipe, tested against `rhel-bootc`. If you already ship a prebuilt module RPM, it installs in the final stage unchanged, against a kernel it was built for or one that is kABI-compatible with it, and you are in bucket 1.

**Bucket 4: a change to the product.** Your installer assumes a running host: it probes the machine, contacts a license server, or asks a question, and the build has no hardware, no running systemd, no D-Bus, and no real answer to give. Three things to try before concluding you are here. Look for a flag that skips the probe or runs the installer non-interactively: many vendor installers have one, and with it the customer can install at build time and move the machine-specific step to a first-boot unit of their own, which is bucket 2. If the value being derived is the same on every machine, it is a static file you can ship. And test, rather than assume, whether the software re-derives it at startup and repairs itself anyway: the build goes green either way. Failing all three, the change is yours, and it comes in two parts: a flag that lets the installer run without the probe, and a oneshot service gated on a stamp file in `/var` that does the probe on the real machine. See [first boot](#do-machine-specific-setup-at-first-boot). This is the most work of the four, and the most of all when the installer has no way past the check and fails loudly.

| Bucket | What it looks like | Who does the work | What you ship |
|--------|--------------------|-------------------|---------------|
| **1** | Installs unchanged with `RUN dnf install` or your install script | Nobody | A support statement |
| **2** | Works after an adjustment in the build: writable `/opt` paths symlinked to `/var`, with `tmpfiles.d` entries | The customer, in their Containerfile | The snippet, documented |
| **3** | A kernel module distributed as source | You write the recipe, the customer runs it | A tested multistage build |
| **4** | An installer that needs a running host and has no way to skip the check | You, in the product | A bypass flag and a first-boot unit |

## Getting a RHEL bootc image

Before any of the patterns below matter, you need a RHEL bootc image on a machine you control. This is where partner engineers lose the most time, and the reason is that two separate credentials are involved and they are easy to mistake for one:

- **A registry login** gets you the base image. Everyone needs this.
- **A RHEL entitlement** makes `dnf install` work *inside* the build. Separate credential, separate failure, and it bites after the first one is working.

Sort out the first, then the second.

### Where the images are

RHEL bootc base images live in Red Hat's authenticated registry:

- `registry.redhat.io/rhel10/rhel-bootc:latest`
- `registry.redhat.io/rhel9/rhel-bootc:latest`

They are not on `registry.access.redhat.com`, the registry that serves UBI without a login. Asking for them there returns a specific error, and it is worth recognizing it rather than reading it as a typo:

```
{"errors":[{"code":"UNSUPPORTED","message":"This repo requires terms acceptance
and is only available on registry.redhat.io"}]}
```

Unauthenticated requests to `registry.redhat.io` itself return `401`. There is no anonymous path to these images.

For the tag to build against, check the [Red Hat Ecosystem Catalog](https://catalog.redhat.com/search) rather than a list printed here: `latest` moves, and which minor-version tags exist changes over the release's life. If your test results need to be reproducible, resolve the tag to a digest once (`skopeo inspect`) and pin that.

### Credential one: a registry login

`registry.redhat.io` accepts a Red Hat login or a registry token, and nothing else. If you already have a Red Hat account, that is the same login you use for the Customer Portal. If you don't, two free routes to one exist: joining the [Red Hat Developer Program](https://developers.redhat.com), and a [30-day trial subscription](https://access.redhat.com/products/red-hat-enterprise-linux/evaluation). Either gets you an account that can pull. See [Red Hat Container Registry Authentication](https://access.redhat.com/RegistryAuthentication) for the full picture.

Log in and pull:

```
$ podman login registry.redhat.io
$ podman pull registry.redhat.io/rhel10/rhel-bootc:latest
```

`skopeo login` and `buildah login` work the same way and share the same credential file.

One thing to get right: log in as the identity that will run the build. Red Hat's documentation writes this step as `sudo podman login`, which writes root's credential file. A rootless `podman build` afterwards reads yours, finds nothing, and fails to pull the base image with an authentication error that looks like the login didn't take. Credentials land in `${XDG_RUNTIME_DIR}/containers/auth.json` on Linux and `$HOME/.config/containers/auth.json` on macOS; see [`containers-auth.json(5)`](https://github.com/containers/image/blob/main/docs/containers-auth.json.5.md) for the full search order.

The Linux default is also temporary. `${XDG_RUNTIME_DIR}` is a tmpfs that goes away when your session ends, so after a logout or a reboot the login is gone, and the next build fails with the same authentication error as if you had never logged in. To keep it, log in to the persistent file instead:

```
$ podman login --authfile ~/.config/containers/auth.json registry.redhat.io
```

That path is the second stop in the documented search order, so `podman build`, `buildah`, and `skopeo` find it without being told. Know what is now on disk: the file holds your username and password base64-encoded, which anyone who can read the file can decode, so only the file's permissions protect it. `podman login` writes it mode 0600 in a directory it creates mode 0700, so there is nothing to set and nothing to loosen. On a workstation that is fine. On a shared machine it is one more reason for the next paragraph, because a personal login here is your Customer Portal password, not a registry token.

On macOS and Windows, the Red Hat Authentication extension for [Podman Desktop](https://podman-desktop.io/), upstream or the Red Hat build, does this step for you: sign in with your Red Hat account and it logs the Podman machine in to `registry.redhat.io`. It also registers that machine with your developer subscription, which is the next credential.

For CI or any shared build host, don't put a person's Customer Portal credentials on it. Red Hat provides [registry service accounts](https://access.redhat.com/terms-based-registry/) for exactly this: tokens scoped to registry pulls, one per shared system.

### Credential two: entitlement for the build

A registry login gets you the image. It does not get you the RPM content inside it.

The RHEL bootc base image ships with no repository configuration and no entitlement certificates of its own: `/etc/yum.repos.d/` and `/etc/pki/entitlement/` are both empty, and `dnf repolist` inside it reports no repositories. It picks up RHEL content from the build host. So a build that pulls the base image fine will stop at its first `RUN dnf install` for want of repositories, and the error names a missing package rather than a missing subscription.

The short version: **run your builds on a registered, subscribed RHEL system.** Red Hat's Podman on such a host passes the host's entitlement into the build for you, and nothing in your Containerfile has to know about it. This is the path with the fewest moving parts, and it is the one Red Hat's own documented examples assume.

If you don't have a RHEL subscription to register that host with, the no-cost [Red Hat Enterprise Linux Developer Subscription](https://access.redhat.com/solutions/4078831) is the route most partner engineers take to a working build box. Note that it is a separate thing from the developer *account* in the previous section: the account lets you pull the image, the subscription is what makes `dnf` work once you're building against it. Signing up for the Developer Program does not enroll you in it.

A developer laptop running macOS or Windows is not a registered RHEL host, and neither is a stock CI runner. Two shortcuts cover most laptops. Fedora, CentOS Stream, and the RHEL rebuilds package `subscription-manager` and the same Podman mount configuration, so registering one of them with the developer subscription gets you the same result as the RHEL host above. On macOS and Windows, the [Red Hat Authentication extension](https://github.com/redhat-developer/podman-desktop-redhat-account-ext) for Podman Desktop, upstream or the Red Hat build, registers the Podman machine with the developer subscription when you sign in, and builds inside that machine are entitled the same way. Everywhere else the entitlement has to come from certificates you mount as build secrets. That is a legitimate documented pattern, covered in [Repo files, GPG keys, and credentials](#repo-files-gpg-keys-and-credentials) along with why the certificates must not end up in a layer. Whether your subscription terms cover a given CI setup, or a developer subscription covers a given build box, is a question for your Red Hat agreement rather than this guide.

### Trying the model before you have credentials

If you want to shake out filesystem-model problems today and the account paperwork is still moving, the CentOS Stream bootc images are public and need no login:

```
$ podman pull quay.io/centos-bootc/centos-bootc:stream10
```

Most of what this guide explains is a property of bootc rather than of RHEL, so a read-only `/opt`, a `/var` that isn't seeded, or a scriptlet that calls `systemctl start` will show up there just as they would on RHEL. It is a fast way to find out where the work is. It is not RHEL: the package set, the kernel, and the support story all differ, so anything you intend to claim support for has to be built and tested against `rhel-bootc`.

### Confirming you have what you think you have

Two things are worth reading out of the image before you build against it:

```
$ podman run --rm registry.redhat.io/rhel10/rhel-bootc:latest cat /usr/lib/os-release
$ podman run --rm registry.redhat.io/rhel10/rhel-bootc:latest sh -c 'cd /usr/lib/modules && echo *'
```

The first tells you which RHEL you actually pulled, which matters when `latest` has moved under you. The second is the image's kernel version, and it is the number a kernel module has to be built against: see [Kernel modules](#kernel-modules-build-against-the-images-kernel) for why the build host's `uname -r` is the wrong answer.

**What to do:**

- Get a Red Hat login (Developer Program or trial if you don't have one), then `podman login registry.redhat.io` as the identity that runs your builds, on Linux with `--authfile` pointed at a path that survives a reboot.
- Run your builds on a registered, subscribed RHEL system, or a Fedora, CentOS Stream, or RHEL rebuild host registered with `subscription-manager` the same way. The no-cost developer subscription is the usual way to get one; it is a separate signup from the developer account. Off such a host, mount entitlement certificates as build secrets.
- On macOS and Windows, sign in through the Red Hat Authentication extension for Podman Desktop, upstream or the Red Hat build: it handles both the registry login and the Podman machine's subscription.
- Use registry service accounts, not personal credentials, on CI and shared build hosts.
- Pin to a digest if your results need to be reproducible.
- Then boot what you built. [Booting your image in a test VM]({{ '/test-vm/' | relative_url }}) has the shortest supported routes and what to look at once it is up.

## At image build time

Everything in this group happens inside the image build, before any machine boots.

### Software installs at build time, not at runtime

On traditional RHEL, you can `dnf install` anytime. On image mode, the OS image is built from a Containerfile, and the running system is read-only. All installation happens during the image build:

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest
RUN dnf install -y your-package && dnf clean all
```

The wall is *when*, not *how*. RPM packaging is not a requirement: the build can run your install script (`RUN ./install.sh`), `COPY` in unpackaged content, or unpack a tarball, and the result ships in the image like anything else. Whatever your installer does, it runs under the build-environment caveats in the next section (no systemd, no hardware, no booted kernel).

What fails is installation on a running system. `dnf` is present, so the command runs, but it fails with a read-only filesystem error against `/usr`. So does the `curl | bash` script that worked as a `RUN` step in the build, and so does an RPM an admin installs post-deploy. (There is one transient exception for debugging; see [Debugging](#debugging-on-a-running-system).)

**What to do:** Provide customers with a Containerfile snippet they can add to their image build: a documented `RUN dnf install` line, or a `RUN ./install.sh` with any build-unsafe steps removed. This is the image mode equivalent of "add our repo and install our RPM."

### The build environment is a container, not a booted system

When `dnf install` runs in a Containerfile, it runs inside a container build. This is a chroot-like environment, not a booted Linux system. There is:

- **No running systemd.** The build has processes, but no service manager and no D-Bus.
- **No hardware.** No block devices, no NICs, no GPUs, no TPMs, no `/sys` populated with real device info.
- **No target-environment network.** Outbound access works (it is how `dnf install` fetches packages), but the target's services are absent: no production DHCP, no local daemons to connect to, and any server you reach sees the build container's identity, not the deployed machine's.
- **No kernel of its own.** `uname -r` reports the build host's kernel, not the target image's.

If your RPM's `%post` scriptlets restart services, probe hardware, load kernel modules, or contact a license server, they will fail or do the wrong thing during the container build.

First, the good news: `systemctl enable` works during container builds. It just creates symlinks, no running systemd needed, and it is the supported way to enable your service. Both upstream bootc and RHEL docs use `RUN systemctl enable ...` in Containerfiles. The same goes for `disable`, `preset`, and `mask`. Enablement also has two owners: your package states a default (a preset file, or `systemctl enable` via the standard scriptlet macros), and the image build can override it either way with `RUN systemctl enable` or `RUN systemctl disable`.

Anything that talks to a *running* systemd fails: `start`, `stop`, `restart`, `daemon-reload`, `is-active`. So do these common scriptlet moves:

- Hardware inventory or fingerprinting at install time: sees nothing useful
- D-Bus registration calls: no bus exists
- Interactive prompts (a license key, an EULA): no TTY and no open stdin, so the prompt fails or misbehaves instead of waiting for input
- `firewall-cmd`: needs the running firewalld daemon over D-Bus. Ship zone configuration files instead, or run `firewall-offline-cmd` in the build; it edits firewalld config without the daemon
- Writing content to `/run`: it exists during the build, but nothing written there ships in the image
- `sysctl` commands or `modprobe`: no kernel interface

One failure in this family is silent, and it is the worst kind. Older packaging often guards its systemd calls behind a check for `/run/systemd/system`, the classic pattern for serving sysv and systemd hosts from one package. That directory does not exist in a container build, so the guarded branch skips without an error: the build succeeds, and the deployed image simply has no service enabled. The wall does not announce itself. Audit your scriptlets, including inherited ones, with `rpm -qp --scripts your-package.rpm`.

**What to do:** Make your `%post` scriptlets container-build-safe: install files, enable services, nothing else. Move machine-specific work to [first boot](#do-machine-specific-setup-at-first-boot). Replace prompts with a config file or a first-boot step. Document whether your package enables its service by default, so image builders know whether they need a `RUN systemctl enable` line.

### You build on one machine and deploy to another

The container build runs on a build host: a developer laptop, a CI runner, a build farm node. The resulting image deploys to completely different hardware. RPMs that probe the build environment at install time get wrong answers:

- Hardware detection sees the build host (or an empty container), not the target server
- Cross-architecture builds (an aarch64 image built on an x86_64 host) run under emulation: the reported architecture is the target's, but the kernel is still the build host's
- Network interface discovery returns container networking, not production NICs
- The hostname is an ephemeral container ID, not the production hostname
- License managers that phone home during `%post` register the wrong system

If you've supported golden-image pipelines before, note the difference. A traditional golden image is usually snapshotted from a *booted* host, so install-on-the-target workflows still worked: the installer ran on a real system with systemd, hardware, and a running kernel. A bootc image build is a container build that never boots. That is the difference that bites even teams with years of golden-image experience, and it's why RPMs designed for install-on-the-target-host workflows hit this wall.

**What to do:** Defer all environment-specific configuration to runtime. Your RPM installs binaries and default configs. A first-boot service figures out the actual hardware, network, and identity. See [first-boot setup](#do-machine-specific-setup-at-first-boot).

### Kernel modules: build against the image's kernel

The trap isn't the toolchain; it's the kernel. A container build can install compilers fine. But everything in it defaults to the build host's kernel, and your module has to be built against the image's kernel instead.

The pieces:

- `uname -r` during the container build reports the build host's kernel, not the image's. If your build uses it to pick the module install path, the module lands in the wrong `/usr/lib/modules/` directory. Inside the build, find the image's kernel with the documented pattern: `kver=$(cd /usr/lib/modules && echo *)`.
- Compilation moves into the image build. On the deployed host `/usr/lib/modules` is read-only, so a build step that would have run on the target, DKMS or anything else, has nowhere to put its result.
- The kernel and bootloader are managed by bootc, not by RPM scriptlets. Your `%post` must not touch `/boot` or GRUB configs. Kernel arguments go through `kargs.d`: ship a TOML file at `/usr/lib/bootc/kargs.d/<name>.toml` in the image, for example `kargs = ["mydriver.option=1"]`, with an optional `match-architectures` key; see [kernel arguments](https://github.com/bootc-dev/bootc/blob/main/docs/src/building/kernel-arguments.md).
- If your driver is needed before the root filesystem mounts (storage controllers), also include it in the initramfs: a dracut config file plus a rebuild against the explicit target kernel version.

RHEL's documented flow is the default: a multi-stage Containerfile whose builder stage installs `make`, `gcc`, and `kernel-devel`, builds the driver RPM with `rpmbuild` against the image's own kernel, and whose final stage installs that RPM (`%post` runs `depmod`). If you already ship a binary module RPM, that is still what you ship: it installs in the final stage the same way, as long as the module inside it was built against that image's kernel. If your packaging is already DKMS-based, the same layout works with DKMS in the builder stage, and the example below uses it because it has the most moving parts. Note that `dkms` ships from EPEL, not RHEL's own repos: community-maintained, its own update cadence, outside RHEL support coverage. Some EPEL packages also need the CodeReady Builder repo enabled; `dkms` itself does not. If one of your build dependencies does, follow Red Hat's documentation for enabling CRB, since the repository id and the enablement path depend on how your build is entitled.

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest AS builder
# dkms ships from EPEL, not RHEL's own repos. Pin kernel-devel to the
# image's kernel: the repos may already carry a newer one, and a mismatch
# recreates the wrong-kernel trap this section exists to close.
RUN kver=$(cd /usr/lib/modules && echo *) && \
    dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm && \
    dnf install -y dkms make gcc kernel-devel-"$kver"
COPY mydriver-1.0/ /usr/src/mydriver-1.0/
# Build against the image's kernel. The -k flag is the whole trick:
# DKMS's default target is the running kernel, which in a container
# build is the build host's, not the image's.
RUN kver=$(cd /usr/lib/modules && echo *) && \
    dkms add -m mydriver -v 1.0 && \
    dkms build -m mydriver -v 1.0 -k "$kver"

FROM registry.redhat.io/rhel10/rhel-bootc:latest
# Receive the built module from the builder stage and register it
RUN --mount=type=bind,from=builder,source=/var/lib/dkms/mydriver/1.0,target=/tmp/mydriver \
    kver=$(cd /usr/lib/modules && echo *) && \
    install -D -m 0644 /tmp/mydriver/"$kver"/*/module/mydriver.ko* \
        -t /usr/lib/modules/"$kver"/extra/ && \
    depmod -a "$kver"
```

One consistency note on the example: installing `epel-release` from a URL is trust on first use, the same pattern this guide's GPG guidance warns against. It is defensible here because it is EPEL's own documented bootstrap, it runs in a builder stage that is discarded, and only the compiled module crosses into the final image.

One wall that does not move: Secure Boot module signing. Under Secure Boot signature enforcement, an out-of-tree module that is unsigned, or signed with a key the machine doesn't trust, will not load, on image mode exactly as on traditional RHEL. The obligation is unchanged. What moves is where the signing happens: into the build, because that is where the module now gets built. That makes a builder stage a poor place to generate a key, since anything generated there is discarded along with the stage, and a module signed that way carries a signature no machine can verify. Nothing in the build reports a problem, and the failure surfaces at load time on a Secure Boot system. If your customers run Secure Boot, sign during the build with a key you control and mount it as a build secret; see [Repo files, GPG keys, and credentials](#repo-files-gpg-keys-and-credentials).

**What to do:** Build your module against the image's kernel, in a builder stage with `kernel-devel` pinned to that version, using whatever you build with today: `rpmbuild`, `make`, or DKMS with `-k`. Ship the built artifact in the final image, and leave no compile step for the deployed host.

### Repo files, GPG keys, and credentials

Your `.repo` file and your GPG key go into the image at build time like any other file, and they stay on the deployed system. That is the normal case, and it is useful: `bootc usr-overlay` puts a transient writable overlay on `/usr` (see [Debugging](#debugging-on-a-running-system)), and with your repo in place a customer can `dnf install` a tool or a test build from it for the length of a session. Nothing about image mode asks you to strip the repo out.

GPG keys are public, but public is not the same as trusted. Your key is the trust anchor: every RPM in the build is accepted because that key vouches for it. Don't have customers fetch it from a URL at build time. Whatever the server returns that day becomes the root of trust, and a compromised host can supply both the packages and the key that validates them.

Instead, ship the key as a file in your integration materials. The customer vendors it into their build context, copies it into the image at `/etc/pki/rpm-gpg/`, and references it with `gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-yourvendor` in the `.repo` file (or runs `rpm --import` on that same path). Publish the key's fingerprint through a separate channel so they can verify what they vendored. File-based keys also keep working in air-gapped builds, which is why RHEL's own offline guidance recommends the same pattern.

If your repository needs a credential, that is the one thing to keep out of the image. Anyone who can pull the image can read every file in it: pulling and unpacking the layers with standard tools (`skopeo copy`, `podman save`, or simply running the image) exposes anything embedded, and build args and environment variables are easier still, since they show up in the image history via `podman inspect`. Build secrets exist for this: `--mount=type=secret` in the Containerfile makes a credential available during the build without landing it in a layer. The mount protects only what stays in it: a credential copied into a config file or a cache directory persists in a layer just as surely as `COPY` would, and build logs persist too, so an `echo`, a verbose flag, or a failing command can write the value into them. Read the secret from `/run/secrets/<id>` and keep it out of anything that outlives the build. RHEL entitlement certificates on a build host that is not a registered RHEL system use the same mechanism; see [Credential two](#credential-two-entitlement-for-the-build).

**What to do:**

- Ship your `.repo` file and GPG key as files; publish the key's fingerprint out of band.
- If the repo needs a credential, put it in a build secret, never in a layer, `ENV`, or a build arg.

### Run `bootc container lint` in the build

One line at the end of the Containerfile catches several of the mistakes in this guide automatically:

```dockerfile
RUN bootc container lint
```

It fails the build on a physical `/var/run` directory, warns on `/var` content without `tmpfiles.d` entries, and checks other image mode invariants. Cheap insurance.

The Containerfile belongs to whoever builds the image, so the line is theirs to run. It is worth running yourself too, when you test your add-on against a build: it catches what your package contributes while the build is still yours. Put the line in the Containerfile snippet you ship and the check is there from the customer's first build.

**What to do:** Run `bootc container lint` on your own test builds, and put `RUN bootc container lint` at the end of the Containerfile snippet you ship.

## At first boot and deploy

The image is built. Now it lands on real machines. This group is short because most of the build-time gotchas above collapse into a single pattern that lives here: the first-boot service. To watch it happen, boot the image in a VM: [Booting your image in a test VM]({{ '/test-vm/' | relative_url }}).

### Do machine-specific setup at first boot

Move the machine-specific work here, where the real hardware, network, and identity exist. The pattern is one oneshot service plus a stamp file.

What belongs where:

- **Build time (your RPM, the image build):** install files, ship default configs, enable the service.
- **First boot (this service):** hardware discovery, network and identity, license registration and activation, anything that writes machine-specific state into `/etc` or `/var`.

The canonical unit:

```ini
# /usr/lib/systemd/system/mypackage-firstboot.service
[Unit]
Description=One-time setup for mypackage
After=network-online.target
Wants=network-online.target
ConditionPathExists=!/var/lib/mypackage/.setup-done

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/lib/mypackage/firstboot.sh

[Install]
WantedBy=multi-user.target
```

Two mechanism notes. First, the trigger: `ConditionFirstBoot=yes` looks like the obvious choice, but systemd counts a boot as the first one only when `/etc/machine-id` is missing or contains `uninitialized`; see [First Boot Semantics](https://www.freedesktop.org/software/systemd/man/latest/machine-id.html#First%20Boot%20Semantics) in `machine-id(5)`. An empty file does not count, and the bootc base images ship it empty, a default rpm-ostree documents under [`machineid-compat`](https://coreos.github.io/rpm-ostree/treefile/), so on a stock image the condition is never true. Cloned VMs fail it the same way, since `virt-sysprep` truncates the file rather than removing it. The stamp file works in every flow. Second, the stamp: have the script create `/var/lib/mypackage/.setup-done` as its last step, only on success. A failed run then retries on the next boot, so make the script idempotent.

**What to do:** Ship a oneshot service gated on a stamp file in `/var`. Do discovery, registration, and activation there, not in `%post`. Write the stamp only on success. Document what the service needs (network access, a license key in `/etc`) so customers can supply it.

### `/var` starts from the image, then belongs to the machine

`/var` is the primary writable, persistent location on an image mode system. Three rules:

1. **Image content in `/var` is applied only on first deployment.** If you add a file to `/var/lib/myapp/` in a new image version, it will **not** appear on systems that were already deployed. After initial setup, `/var` is machine-local state.

2. **Directories must be explicitly created.** Don't assume `/var/lib/mypackage/` or `/var/log/mypackage/` exist at runtime just because your RPM created them at build time. Use `tmpfiles.d`:

```
# /usr/lib/tmpfiles.d/mypackage.conf
d /var/lib/mypackage 0755 root root -
d /var/log/mypackage 0750 mypackage mypackage -
```

Or use `StateDirectory=mypackage` in your systemd unit file (recommended: it creates and manages `/var/lib/mypackage` automatically).

3. **Never create a physical `/var/run` directory.** `/var/run` is a symlink to `/run`. A package that creates it as a real directory is shipping a bug, and `bootc container lint` fails the build on it. Nothing under `/run` survives a reboot.

`/var` also survives rollbacks, which has consequences of its own; see [Rollbacks](#rollbacks-etc-reverts-var-does-not).

One more path worth knowing: `/tmp` is a tmpfs on image mode, sized at half of system memory and empty after every reboot. If your software stages large files there, it is spending RAM, and `/var` is the better place for anything sizable or anything that has to survive a restart.

**What to do:** Create your `/var` directories with `tmpfiles.d` or `StateDirectory=`, and treat `/var` as machine-local from day two. `bootc container lint` warns when image content under `/var` has no matching `tmpfiles.d` entry.

## At runtime

The system is up. The image content is read-only; `/var` and `/etc` are not.

### `/opt` is read-only at runtime

Many enterprise packages install to `/opt/<vendor>/`. On traditional RHEL, `/opt` is a normal writable directory. On image mode, `/opt` is **read-only at runtime**: it ships as part of the image, just like `/usr`. This is a property of the RHEL bootc base images specifically; RHEL CoreOS and some other ostree-based systems symlink `/opt` into `/var` instead, so don't generalize from one platform to the other.

Installing your package to `/opt` in a Containerfile works fine: the files become part of the image. The problem is that many `/opt/<vendor>/` trees mix content that should be read-only with content that needs to be writable:

- Binaries and libraries: should ship in the image, read-only is fine
- Log files: must be writable at runtime
- Caches and state: must be writable at runtime
- Plugin directories: may need runtime writes
- Signature/definition databases: may need runtime updates

A flat `/opt/myvendor/` that contains all of these will not work as-is.

The standard fix is to separate the writable content, and the recommended place to do it is the image build.

Move writable directories to `/var` and create symlinks back. Ship this snippet in your integration docs:

```dockerfile
RUN dnf install -y myvendor-agent && \
    mv /opt/myvendor/logs /var/log/myvendor && \
    ln -sr /var/log/myvendor /opt/myvendor/logs && \
    mv /opt/myvendor/data /var/lib/myvendor && \
    ln -sr /var/lib/myvendor /opt/myvendor/data
```

That snippet needs a companion, and this is the part that's easy to miss. The `mv` puts your log and data directories into the image's `/var`, and image content in `/var` lands only on a system's first deployment. On a machine that was already running before your package arrived, those directories are never created, the symlinks point at nothing, and the first write fails. Ship `tmpfiles.d` entries in your package so they exist on every system, per [the `/var` rules](#var-starts-from-the-image-then-belongs-to-the-machine). Use `tmpfiles.d` rather than `StateDirectory=` here, because a symlink target has to exist whether or not your service has ever started, and `bootc container lint` warns when image content under `/var` has no matching entry, which catches the omission in the build.

One property of the symlink pattern is worth naming, because it is why this guide leads with it. The path stays self-describing: `/opt/myvendor/logs` is a symlink, visible as one to anything that stats it, and any tool that follows symlinks reaches the real content. File integrity monitoring, log collection, and compliance scanners keep working against the paths they already know.

If the tree genuinely can't be restructured, there is a second documented option: a state overlay (`RUN systemctl enable ostree-state-overlay@opt.service`) makes `/opt` writable with a persistent overlay, where image files win over local edits on update. Treat it as the escape hatch, not the default: every write to the overlay is machine-local drift that no image digest accounts for, so you trade away part of the auditability the read-only model exists to provide. Upstream recommends the symlink pattern first because it keeps the most content read-only.

**What to do:** Split writable content out of `/opt`. Give customers the symlink snippet for their Containerfile, and ship `tmpfiles.d` entries so the `/var` targets exist on every machine, not just newly deployed ones. The best long-term fix is in your next release: move binaries to `/usr/lib/<package>/` and runtime data to `/var/lib/<package>/` so the split disappears.

### `/usr/local`: same rules as `/usr`

On image mode, `/usr/local` is a regular directory (not a symlink) but is **read-only at runtime**, just like the rest of `/usr`. Software that installs to `/usr/local/bin` at build time works fine. Software that expects to drop binaries there at runtime (self-updaters, plugin managers, post-deploy scripts) will fail.

**What to do:** Install at build time. If you genuinely need to write binaries at runtime, use `/var/lib/<package>/bin` and add it to `PATH`, or run that component as a container.

### Self-updating software will not work

Any software that updates itself at runtime, downloading new binaries or replacing files in `/opt` or `/usr/local/bin`, will fail on image mode. The filesystem is read-only.

Updates to your software go through the image build pipeline: the customer rebuilds their image with your new package version, pushes it to a registry, and systems stage it on the next `bootc upgrade`, booting into it at the following reboot (`bootc upgrade --apply` reboots immediately).

**What to do:** If your software needs to update definitions or signatures at runtime (antivirus, IDS), store those in `/var/lib/<package>/`, which is writable. Binary updates go through the image rebuild flow. If that cadence is too slow for your agent, ship it as a logically bound container image instead; see [the first question](#first-question-does-it-need-to-be-in-the-os-image-at-all).

### Debugging on a running system

On traditional RHEL, troubleshooting starts with something like `dnf install strace`. Image mode has a documented equivalent: `bootc usr-overlay` puts a transient writable overlay on `/usr`, so `dnf install strace` works, and everything you installed disappears on reboot. This is the supported way to get debug tools onto a running system temporarily.

For more than a quick session:

- Maintain a debug variant of the image with the tools pre-installed. A machine moves to it with `bootc switch` and back again when the session ends; each switch takes effect at a reboot.
- Run tools from a privileged container: `podman run --pid=host --network=host --privileged -it registry.redhat.io/rhel10/support-tools`

**What to do:** Include an image mode section in your troubleshooting docs. List the debug tools your support team needs so customers can bake them into their image builds proactively, and teach your support staff `bootc usr-overlay` for live sessions.

One expectation to reset in your support organization: on image mode, your software reaches the running system through the image build pipeline, and that pipeline is now part of the triage path. "Which build produced this system" is a first-class question. Ask for the Containerfile along with the usual logs.

### Configuration management works differently

Ansible, Puppet, Chef, and similar tools can still manage image mode systems, but they should manage **configuration** (files in `/etc`, service state), not **packages**. Any playbook or recipe that calls `dnf install` at runtime will fail.

**What to do:** Split the automation you recommend to customers: package installation lines go in the Containerfile, and playbooks or recipes manage `/etc` files and service state on the running system.

## Across upgrades and rollbacks

The subtlest gotchas live here: what merges, what carries forward, and what reverts when the OS image moves.

### Your defaults, their customizations, and the `/etc` merge

On traditional RHEL, vendor default configs and local customizations both live in `/etc`, managed by RPM's `%config` directives. On image mode, the split is cleaner:

| What | Where | Why |
|------|-------|-----|
| **Your vendor defaults** | `/usr/share/<package>/` or `/usr/lib/<package>/` | Ships with the image, read-only, updated when the image is rebuilt |
| **Local customization** | `/etc/<package>/` or `/etc/<package>.conf` | Persistent, survives upgrades, set on the machine |

**Why this matters:** On image mode, `/etc` uses a 3-way merge on upgrades. The merge happens on the machine when the new deployment is created (during `bootc upgrade` or `bootc switch`), not during the image build, and it operates per file: a locally modified file is kept wholesale. There is no line-level merging and there are no conflict markers. If a file in `/etc` has been modified locally, the local version wins, and your updated default will **not** be applied. Metadata changes count too: a `chown` or a permissions change on a config file pins it locally the same as an edit. If your defaults and the local changes are in the same file, you have a problem.

**What to do:** Ship your defaults in `/usr/lib/` or `/usr/share/` and treat `/etc` as local override space: have the application read `/etc/myapp.conf` if it exists and fall back to `/usr/share/myapp/myapp.conf.default`. It is the same pattern systemd and modern Linux services already use. Your defaults then update cleanly with the image, and local customizations persist independently.

**Specific `/etc` gotcha, `/etc/passwd` and `/etc/group`:** If **any** local change touches `/etc/passwd` (upstream's stock example is setting a root password), subsequent image updates that add new system users will **not** take effect on that machine. So don't write to `/etc/passwd` from your package: create system users with `sysusers.d`, or use `DynamicUser=yes` for service accounts. Avoid build-time `useradd` for a second reason too: it can allocate a different UID on each image rebuild, which breaks ownership of data already persisted in `/var`.

`sysusers.d` has one wrinkle of its own: by default it allocates UIDs per machine, so the same user can hold different UIDs across a fleet, which matters as soon as you `chown` persistent data in `/var`. The fix is the static allocation upstream recommends whenever persistent data is involved: pin the UID and GID in your entry (`u myuser 900:900 "My vendor service"`), and ownership stays stable across machines and rebuilds. The cost is a collision risk if something else claims the same number.

Also worth knowing: bootc supports a transient `/etc` (`transient = true` under `[etc]` in `/usr/lib/ostree/prepare-root.conf`, set in the image at build time), rebuilt from the image on every boot. A system with it enabled gives up persistent local `/etc` changes entirely. If your software writes config to `/etc` at runtime, know that this deployment variant exists.

### Rollbacks: `/etc` reverts, `/var` does not

A system can be rolled back to the previous OS image with `bootc rollback`. The two writable areas behave differently when that happens:

- **`/etc` reverts** to the prior deployment's state. Rollback reorders existing deployments, and each keeps its own `/etc`.
- **`/var` carries forward** unchanged. Your runtime data may now be "ahead" of both the OS and your config.

**What to do:** Treat `/var` data as potentially newer than the running code. If your app migrates its data format on upgrade, make the migration rollback-tolerant, or version your data files and fail cleanly rather than corrupting them. And don't keep state in `/etc` that must survive a rollback: state goes in `/var`, config goes in `/etc`.

## The mental model

Think of an image mode system like a phone or an appliance. The OS image is the firmware: it ships complete and read-only. Your app is part of that firmware, installed at build time, and its runtime data lives in a separate writable area (`/var`).

The image build (Containerfile) is where your RPM gets installed. The running system is where your app does its work, reading from the read-only image and writing to `/var` and `/etc`.

If you remember one thing: **your software ships in the image, anything machine-specific waits for first boot, and machine state lives in `/var`.**

## Quick reference

The patterns in this guide, in one table, for looking things up after a first read. Each row links to the section with the details and the fix.

| Pattern | Works on image mode? | What to do instead | Details |
|---------|---------------------|--------------------|---------|
| Agent or scanner that could ship as a container | Yes, often the simplest path | Quadlet or logically bound image | [First question](#first-question-does-it-need-to-be-in-the-os-image-at-all) |
| Pulling `rhel-bootc` without a login | No, it is not on the unauthenticated registry | `podman login registry.redhat.io` first | [Getting an image](#credential-one-a-registry-login) |
| `sudo podman login`, then a rootless `podman build` | No, the build reads a different credential file | Log in as the identity that runs the build | [Getting an image](#credential-one-a-registry-login) |
| Registry login gone after a reboot | Expected: the Linux default `auth.json` lives on a tmpfs | `podman login --authfile ~/.config/containers/auth.json` | [Getting an image](#credential-one-a-registry-login) |
| `RUN dnf install` on an unsubscribed build host | No, the base image carries no repositories | Build on a registered RHEL system, or mount entitlements | [Getting an image](#credential-two-entitlement-for-the-build) |
| `curl \| bash` installer at deploy time | No, but it works as a build step | Run it in the image build | [Installation](#software-installs-at-build-time-not-at-runtime) |
| RPM installed by admin post-deployment | No | Include it in the image build | [Installation](#software-installs-at-build-time-not-at-runtime) |
| `%post` runs `systemctl start` | No | Use `enable` only; start happens at boot | [Build environment](#the-build-environment-is-a-container-not-a-booted-system) |
| Scriptlet guards `systemctl` behind a `/run/systemd/system` check | Silently skipped: build passes, image has no service | Drop the guard; `enable` works in builds | [Build environment](#the-build-environment-is-a-container-not-a-booted-system) |
| `%post` prompts for input (license key, EULA) | No, builds have no TTY and no open stdin | Config file in `/etc`, or first boot | [Build environment](#the-build-environment-is-a-container-not-a-booted-system) |
| `%post` opens ports with `firewall-cmd` | No, firewalld isn't running | Ship zone config, or run `firewall-offline-cmd` in the build | [Build environment](#the-build-environment-is-a-container-not-a-booted-system) |
| `%post` probes hardware | No, sees the build environment | Defer to a first-boot service | [Build vs. deploy machine](#you-build-on-one-machine-and-deploy-to-another) |
| `%post` phones home for licensing | No, wrong host identity | Defer to a first-boot service | [Build vs. deploy machine](#you-build-on-one-machine-and-deploy-to-another) |
| First-boot unit gated on `ConditionFirstBoot=yes` | Never fires: the base image ships an empty `machine-id`, which systemd does not count as a first boot | Gate on a stamp file in `/var` instead | [First boot](#do-machine-specific-setup-at-first-boot) |
| DKMS compile on the deployed host | No | Compile in the build against the image's kernel; ship the result | [Kernel modules](#kernel-modules-build-against-the-images-kernel) |
| `%post` edits `/boot` or GRUB config | No, bootc owns kernel and bootloader | Ship kernel arguments as a `kargs.d` TOML | [Kernel modules](#kernel-modules-build-against-the-images-kernel) |
| Credentials in `.repo` files or image layers | No, layers are readable by anyone who can pull | Use build secrets | [Repo files and credentials](#repo-files-gpg-keys-and-credentials) |
| Assuming `/var/lib/<pkg>` exists at runtime | Not guaranteed | `tmpfiles.d` or `StateDirectory=` | [/var rules](#var-starts-from-the-image-then-belongs-to-the-machine) |
| Package creates a physical `/var/run` directory | No, fatal lint error | It's a symlink to `/run`; leave it alone | [/var rules](#var-starts-from-the-image-then-belongs-to-the-machine) |
| Install to `/opt/<vendor>/` (all content) | Partially, read-only at runtime | Symlink writable subdirs to `/var` | [/opt](#opt-is-read-only-at-runtime) |
| Writing runtime data to `/opt/<pkg>/data/` | No, read-only | Use `/var/lib/<pkg>/` | [/opt](#opt-is-read-only-at-runtime) |
| Drop binaries into `/usr/local` at runtime | No, read-only | Install at build time | [/usr/local](#usrlocal-same-rules-as-usr) |
| Self-updating agent binaries | No | Update via the image rebuild pipeline | [Self-updating software](#self-updating-software-will-not-work) |
| Auto-update signature DB under `/opt` | No, read-only | Store mutable data in `/var/lib/` | [Self-updating software](#self-updating-software-will-not-work) |
| `dnf install` a debug tool on a live system | Only transiently | `bootc usr-overlay`, or a tools container | [Debugging](#debugging-on-a-running-system) |
| Ansible/Puppet `dnf install` at runtime | No | Packages in the Containerfile; config mgmt for `/etc` | [Configuration management](#configuration-management-works-differently) |
| Config files in `/etc` only | Works, but upgrade behavior differs | Defaults in `/usr`, overrides in `/etc` | [The /etc merge](#your-defaults-their-customizations-and-the-etc-merge) |
| `%post` creates users with `useradd` | Risky, `/etc/passwd` drift | Use `sysusers.d` | [The /etc merge](#your-defaults-their-customizations-and-the-etc-merge) |
| Expecting `/etc` and `/var` to move together on rollback | They don't | Design for the asymmetry | [Rollbacks](#rollbacks-etc-reverts-var-does-not) |
| Catching all of the above in a build | Yes, recommended for every build | Run `bootc container lint` on your test builds; ship the line in your snippet | [Lint](#run-bootc-container-lint-in-the-build) |
