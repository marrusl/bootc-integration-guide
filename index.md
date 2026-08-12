---
layout: default
title: "Getting Your App on RHEL Image Mode: What You Need to Know"
---

If your product ships to customers as a host RPM or an installer (agents, monitoring tools, security scanners, drivers, enterprise applications) and those customers are adopting RHEL image mode (bootc), this guide is for you. The message up front: what you need to change might be less than you think. Most of what you already ship works unchanged. The walls are a short list, clearly marked, and each one has a standard fix. Two cases usually mean real refactoring rather than a Containerfile tweak: a kernel module, and an `/opt` tree that can't be restructured. Everything else is closer to a recipe. You don't need to become a bootc expert. You need to know where the walls are.

If your product is already container-native, you can skip most of this guide. Start with [the first question](#first-question-does-it-need-to-be-in-the-os-image-at-all) and stop there.

Why this guide exists: upstream bootc docs are written for the people who build OS images. RHEL docs are written for the people who run the systems. This guide is for the vendor whose software ends up inside an image someone else builds.

Three things this guide is not. It isn't an effort estimate. Some fixes are one Containerfile line; a self-updater or one of the refactoring cases above is release-cycle work, and only you can size it for your product. It isn't a certification or support statement. It tells you what works technically, not what any partner program covers. And it isn't a snapshot of a frozen platform. bootc keeps moving, and some of the walls below are actively being lowered.

**Version 0.7, written against RHEL 10 image mode and bootc as of 2026-08-11.**

A handful of specifics are still being checked against a live system or a real build rather than documentation alone. See [open questions]({{ '/open-questions/' | relative_url }}) for what's unverified and how to help settle it.

## How image mode works

On image mode, the operating system ships as a container image. The customer builds it with a Containerfile, the same way they build application containers, starting from a Red Hat base image. The running system boots from that image, and the OS content is read-only. Updates are atomic: the system pulls a newer image and switches to it on reboot, and it can roll back the same way. Your software becomes one layer of that image, installed during the customer's build.

Where to go deeper:

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

That's the headline: **read-only root, read-write `/var` and `/etc`.** Your binaries, libraries, and default configs ship in the image. Your runtime data, logs, and caches go in `/var`. Local config customization goes in `/etc`.

## First question: does it need to be in the OS image at all?

Before adapting your RPM, ask a more basic question: does your software need to be part of the OS image, or can it run as a container on top of it? Image mode systems run containers natively with Podman, and for agents, scanners, and monitoring tools, a container is often the simpler path. Two patterns:

- **Quadlets.** Ship your agent as a container image plus a small systemd unit file (a Quadlet `.container` file). systemd starts it at boot like any service. You release container images on your own cadence, and the host filesystem rules below stop applying to your code.
- **Logically bound images.** The customer's OS image references your container image, so it is pulled and lifecycled together with the host image while still updating on its own cadence. Upstream's stated use cases are logging, monitoring, configuration management, and security agents: exactly the software this guide is written for. See [logically bound images](https://github.com/bootc-dev/bootc/blob/main/docs/src/logically-bound-images.md).

If your software can run as a container, most of this guide stops applying to you. The catalog below is for software that has to live on the host: drivers and kernel modules, host-level tooling, or anything that genuinely can't be containerized.

## Quick reference

The patterns in this guide, in one table. Each row links to the section with the details and the fix.

| Pattern | Works on image mode? | What to do instead | Details |
|---------|---------------------|--------------------|---------|
| Agent or scanner that could ship as a container | Yes, often the simplest path | Quadlet or logically bound image | [First question](#first-question-does-it-need-to-be-in-the-os-image-at-all) |
| `curl \| bash` installer at deploy time | No, but it works as a build step | Run it in the image build | [Installation](#software-installs-at-build-time-not-at-runtime) |
| RPM installed by admin post-deployment | No | Include in the customer's Containerfile | [Installation](#software-installs-at-build-time-not-at-runtime) |
| `%post` runs `systemctl start` | No | Use `enable` only; start happens at boot | [Build environment](#the-build-environment-is-a-container-not-a-booted-system) |
| Scriptlet guards `systemctl` behind a `/run/systemd/system` check | Silently skipped: build passes, image has no service | Drop the guard; `enable` works in builds | [Build environment](#the-build-environment-is-a-container-not-a-booted-system) |
| `%post` prompts for input (license key, EULA) | No, builds have no TTY and no open stdin | Config file in `/etc`, or first boot | [Build environment](#the-build-environment-is-a-container-not-a-booted-system) |
| `%post` opens ports with `firewall-cmd` | No, firewalld isn't running | Ship zone config, or run `firewall-offline-cmd` in the build | [Build environment](#the-build-environment-is-a-container-not-a-booted-system) |
| `%post` probes hardware | No, sees the build environment | Defer to a first-boot service | [Build vs. deploy machine](#you-build-on-one-machine-and-deploy-to-another) |
| `%post` phones home for licensing | No, wrong host identity | Defer to a first-boot service | [Build vs. deploy machine](#you-build-on-one-machine-and-deploy-to-another) |
| First-boot unit gated on `ConditionFirstBoot=yes` | Unreliable: machine-id can be pre-initialized | Gate on a stamp file in `/var` instead | [First boot](#do-machine-specific-setup-at-first-boot) |
| DKMS compile on the deployed host | No | Compile in the build against the image's kernel; ship the result | [Kernel modules](#kernel-modules-build-against-the-images-kernel) |
| `%post` edits `/boot` or GRUB config | No, bootc owns kernel and bootloader | Ship kernel arguments as a `kargs.d` TOML | [Kernel modules](#kernel-modules-build-against-the-images-kernel) |
| Credentials in `.repo` files or image layers | No, layers are readable by anyone who can pull | Use build secrets | [Repos and credentials](#repos-credentials-and-entitlements) |
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
| Catching all of the above in CI | Yes, recommended for every build | Add `RUN bootc container lint` to the build | [Lint](#run-bootc-container-lint-in-the-build) |

## At image build time

Everything in this group happens inside the customer's Containerfile build, before any machine boots.

### Software installs at build time, not at runtime

On traditional RHEL, you can `dnf install` anytime. On image mode, the OS image is built from a Containerfile, and the running system is read-only. All installation happens during the image build:

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest
RUN dnf install -y your-package && dnf clean all
```

The wall is *when*, not *how*. RPM packaging is not a requirement: the build can run your install script (`RUN ./install.sh`), `COPY` in unpackaged content, or unpack a tarball, and the result ships in the image like anything else. Whatever your installer does, it runs under the build-environment caveats in the next section (no systemd, no hardware, no booted kernel).

What fails is installation on a running system. `dnf` is present, but installs fail against the read-only `/usr`. The same `curl | bash` script that works as a `RUN` step in the build fails with a read-only filesystem error after deployment, and so does an RPM an admin tries to install post-deploy. Your software needs to go in during the customer's image build, whatever format it arrives in. (There is one transient exception for debugging; see [Debugging](#debugging-on-a-running-system).)

**What to do:** Provide customers with a Containerfile snippet they can add to their image build: a documented `RUN dnf install` line, or a `RUN ./install.sh` with any build-unsafe steps removed. This is the image mode equivalent of "add our repo and install our RPM."

### The build environment is a container, not a booted system

When `dnf install` runs in a Containerfile, it runs inside a container build. This is a chroot-like environment, not a booted Linux system. There is:

- **No running systemd.** The build has processes, but no service manager and no D-Bus.
- **No hardware.** No block devices, no NICs, no GPUs, no TPMs, no `/sys` populated with real device info.
- **No target-environment network.** Outbound access works (it is how `dnf install` fetches packages), but the target's services are absent: no production DHCP, no local daemons to connect to, and any server you reach sees the build container's identity, not the deployed machine's.
- **No kernel of its own.** `uname -r` reports the build host's kernel, not the target image's.

If your RPM's `%post` scriptlets restart services, probe hardware, load kernel modules, or contact a license server, they will fail or do the wrong thing during the container build.

First, the good news: `systemctl enable` works during container builds. It just creates symlinks, no running systemd needed, and it is the supported way to enable your service. Both upstream bootc and RHEL docs use `RUN systemctl enable ...` in Containerfiles. The same goes for `disable`, `preset`, and `mask`. Enablement also has two owners: your package states a default (a preset file, or `systemctl enable` via the standard scriptlet macros), and the customer's Containerfile can override it either way with `RUN systemctl enable` or `RUN systemctl disable`.

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
- DKMS cannot run on the deployed host: `/usr/lib/modules` is read-only, and there is no runtime compilation. Compilation moves into the build.
- The kernel and bootloader are managed by bootc, not by RPM scriptlets. Your `%post` must not touch `/boot` or GRUB configs. Kernel arguments go through `kargs.d`: ship a TOML file at `/usr/lib/bootc/kargs.d/<name>.toml` in the image, for example `kargs = ["mydriver.option=1"]`, with an optional `match-architectures` key; see [kernel arguments](https://github.com/bootc-dev/bootc/blob/main/docs/src/building/kernel-arguments.md).
- If your driver is needed before the root filesystem mounts (storage controllers), also include it in the initramfs: a dracut config file plus a rebuild against the explicit target kernel version.

RHEL's documented flow is the default: a multi-stage Containerfile whose builder stage installs `make`, `gcc`, and `kernel-devel`, builds the driver RPM with `rpmbuild` against the image's own kernel, and whose final stage installs that RPM (`%post` runs `depmod`). If your packaging is already DKMS-based, the same layout works with DKMS in the builder stage. Note that `dkms` ships from EPEL, not RHEL's own repos: community-maintained, its own update cadence, outside RHEL support coverage. Some EPEL packages also need the CodeReady Builder repo enabled; `dkms` itself does not. If one of your build dependencies does, follow Red Hat's documentation for enabling CRB, since the repository id and the enablement path depend on how your build is entitled.

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

One wall that does not move: Secure Boot module signing. Under Secure Boot signature enforcement, an unsigned out-of-tree module will not load, on image mode exactly as on traditional RHEL, and one signed with a key the machine doesn't trust is rejected the same way. The obligation carries over unchanged, whether you build with `rpmbuild` or DKMS. One mechanic does move. Signing has to happen in the build now, because that is where the module gets built, and a builder stage is a poor place to keep a key: anything generated there is discarded with the stage, so a module signed that way carries a signature no machine can verify. DKMS's signing support leans on a machine-local MOK that an admin enrolls once, and that arrangement does not survive the move into a container build. Either way nothing in the build reports a problem, and the failure surfaces at load time on a Secure Boot system. If your customers run Secure Boot, sign with your own key during the build (mount it as a build secret; see [Repos, credentials, and entitlements](#repos-credentials-and-entitlements)) and publish the certificate for them to enroll. Enrollment is a machine-level firmware action, so it belongs in their provisioning, not in your package.

**What to do:** Pre-build your module in a builder stage against the image's `kernel-devel`, with `rpmbuild`, `make`, or DKMS pinned via `-k`. Ship the built artifact in the final image. Never rely on DKMS on the deployed host.

### Repos, credentials, and entitlements

When your customer adds your RPM repository to their Containerfile, the `.repo` file and any credentials in it become part of the image layers. Anyone who can pull the image can read every file in it: pulling and unpacking the layers with standard tools (`skopeo copy`, `podman save`, or simply running the image) exposes embedded credentials. Build args and environment variables are even easier to read: they show up in the image history via `podman inspect`.

Build secrets exist for this. `--mount=type=secret` in the Containerfile makes a credential available during the build without landing it in a layer. The mount protects only what stays in it: a credential copied into `/etc/yum.repos.d/`, a config file, or a cache directory persists in a layer just as surely as `COPY` would, and build logs persist too, so an `echo`, a verbose flag, or a failing command can write the value into them. Read the secret directly from `/run/secrets/<id>` and keep it out of anything that outlives the build. All of this is about build-time credentials like repo access; a secret that must exist on the deployed host (a pull secret for bound container images, for example) is a different case, and RHEL's own docs deliberately persist it in the image.

GPG keys are public, but public is not the same as trusted. Your key is the trust anchor: every RPM in the build is accepted because that key vouches for it. Don't have customers fetch it from a URL at build time. Whatever the server returns that day becomes the root of trust, and a compromised host can supply both the packages and the key that validates them.

Instead, ship the key as a file in your integration materials. The customer vendors it into their build context, copies it into the image at `/etc/pki/rpm-gpg/`, and references it with `gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-yourvendor` in the `.repo` file (or runs `rpm --import` on that same path). Publish the key's fingerprint through a separate channel so customers can verify what they vendored. File-based keys also keep working in air-gapped builds, which is why RHEL's own offline guidance recommends the same pattern.

RHEL entitlements in CI are the same mechanism: mounting subscription certificates as build secrets is a legitimate, documented pattern, and baking them into image layers is the anti-pattern. Whether your subscription terms cover a given CI setup and build volume is a question for your Red Hat agreement, not this guide.

**What to do:**

- Put credentials in build secrets, never in layers, `ENV`, or build args.
- Ship your GPG key as a file; publish its fingerprint out of band.
- In CI, mount entitlement certificates as build secrets too.

### Run `bootc container lint` in the build

One line at the end of the customer's Containerfile catches several of the mistakes in this guide automatically:

```dockerfile
RUN bootc container lint
```

It fails the build on a physical `/var/run` directory, warns on `/var` content without `tmpfiles.d` entries, and checks other image mode invariants. Cheap insurance.

**What to do:** Put `RUN bootc container lint` at the end of the Containerfile snippet you ship, and recommend the step in your integration docs.

## At first boot and deploy

The image is built. Now it lands on real machines. This group is short because most of the build-time gotchas above collapse into a single pattern that lives here: the first-boot service.

### Do machine-specific setup at first boot

Move the machine-specific work here, where the real hardware, network, and identity exist. The pattern is one oneshot service plus a stamp file.

What belongs where:

- **Build time (your RPM, the customer's Containerfile):** install files, ship default configs, enable the service.
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

Two mechanism notes. First, the trigger: `ConditionFirstBoot=yes` looks like the obvious choice, but it keys off an uninitialized `/etc/machine-id`, and some provisioning flows initialize that before first boot, so the unit never fires. The stamp file works in every flow. Second, the stamp: have the script create `/var/lib/mypackage/.setup-done` as its last step, only on success. A failed run then retries on the next boot, so make the script idempotent.

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

The standard fix is to separate the writable content, and the two patterns are owned by different people.

Customer-side, in the Containerfile: move writable directories to `/var` and create symlinks back. Ship this snippet in your integration docs:

```dockerfile
RUN dnf install -y myvendor-agent && \
    mv /opt/myvendor/logs /var/log/myvendor && \
    ln -sr /var/log/myvendor /opt/myvendor/logs && \
    mv /opt/myvendor/data /var/lib/myvendor && \
    ln -sr /var/lib/myvendor /opt/myvendor/data
```

Vendor-side, in the systemd unit your package ships: `BindPaths=` mounts writable directories into the read-only tree at service start, with no change to the installed file layout:

```ini
[Service]
BindPaths=/var/log/myvendor:/opt/myvendor/logs
BindPaths=/var/lib/myvendor:/opt/myvendor/data
```

One scope note on `BindPaths=`: it remaps the path only inside that unit's mount namespace. Your service sees a writable `/opt/myvendor/logs`; every other process (helper CLIs, cron jobs, other units, an admin shell) still sees the read-only image content at that path. Use it when all writes come from the service itself; if other tools write there too, use the symlink pattern instead.

If neither pattern fits because your `/opt` tree can't be restructured, there is a third documented option: a state overlay (`RUN systemctl enable ostree-state-overlay@opt.service`) makes `/opt` writable with a persistent overlay, where image files win over local edits on update. Treat it as the escape hatch, not the default: every write to the overlay is machine-local drift that no image digest accounts for, so you trade away part of the auditability the read-only model exists to provide. Upstream recommends the symlink pattern first because it keeps the most content read-only.

**What to do:** Split writable content out of `/opt`. Give customers the symlink snippet for their Containerfile, or ship `BindPaths=` in your own unit file when only your service writes there. The best long-term fix is in your next release: move binaries to `/usr/lib/<package>/` and runtime data to `/var/lib/<package>/` so the split disappears.

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

- Maintain a debug variant of the customer's image with the tools pre-installed. A machine moves to it with `bootc switch` and back again when the session ends; each switch takes effect at a reboot.
- Run tools from a privileged container: `podman run --pid=host --network=host --privileged -it registry.redhat.io/rhel10/support-tools`

**What to do:** Include an image mode section in your troubleshooting docs. List the debug tools your support team needs so customers can bake them into their image builds proactively, and teach your support staff `bootc usr-overlay` for live sessions.

One expectation to reset in your support organization: on image mode, your software reaches the running system through the customer's image build pipeline, and that pipeline is now part of the triage path. "Which build produced this system" is a first-class question. Ask for the Containerfile along with the usual logs.

### Configuration management works differently

Ansible, Puppet, Chef, and similar tools can still manage image mode systems, but they should manage **configuration** (files in `/etc`, service state), not **packages**. Any playbook or recipe that calls `dnf install` at runtime will fail.

**What to do:** Split the automation you recommend to customers: package installation lines go in the Containerfile, and playbooks or recipes manage `/etc` files and service state on the running system.

## Across upgrades and rollbacks

The subtlest gotchas live here: what merges, what carries forward, and what reverts when the OS image moves.

### Your defaults, their customizations, and the `/etc` merge

On traditional RHEL, vendor default configs and customer customizations both live in `/etc`, managed by RPM's `%config` directives. On image mode, the split is cleaner:

| What | Where | Why |
|------|-------|-----|
| **Your vendor defaults** | `/usr/share/<package>/` or `/usr/lib/<package>/` | Ships with the image, read-only, updated when the customer rebuilds |
| **Customer customization** | `/etc/<package>/` or `/etc/<package>.conf` | Persistent, survives upgrades, customer-controlled |

**Why this matters:** On image mode, `/etc` uses a 3-way merge on upgrades. The merge happens on the machine when the new deployment is created (during `bootc upgrade` or `bootc switch`), not during the image build, and it operates per file: a locally modified file is kept wholesale. There is no line-level merging and there are no conflict markers. If the customer has modified a file in `/etc`, their version wins, and your updated default will **not** be applied. Metadata changes count too: a `chown` or a permissions change on a config file pins it locally the same as an edit. If your defaults and the customer's customizations are in the same file, you have a problem.

**What to do:** Ship your defaults in `/usr/lib/` or `/usr/share/` and treat `/etc` as the customer's override space: have the application read `/etc/myapp.conf` if it exists and fall back to `/usr/share/myapp/myapp.conf.default`. It is the same pattern systemd and modern Linux services already use. Your defaults then update cleanly with the image, and customer customizations persist independently.

**Specific `/etc` gotcha, `/etc/passwd` and `/etc/group`:** If **any** local change touches `/etc/passwd` (upstream's stock example is setting a root password), subsequent image updates that add new system users will **not** take effect on that machine. So don't write to `/etc/passwd` from your package: create system users with `sysusers.d`, or use `DynamicUser=yes` for service accounts. Avoid build-time `useradd` for a second reason too: it can allocate a different UID on each image rebuild, which breaks ownership of data already persisted in `/var`.

`sysusers.d` has one wrinkle of its own: by default it allocates UIDs per machine, so the same user can hold different UIDs across a fleet, which matters as soon as you `chown` persistent data in `/var`. The fix is the static allocation upstream recommends whenever persistent data is involved: pin the UID and GID in your entry (`u myuser 900:900 "My vendor service"`), and ownership stays stable across machines and rebuilds. The cost is a collision risk if something else claims the same number.

Also worth knowing: bootc supports a transient `/etc` (`transient = true` under `[etc]` in `/usr/lib/ostree/prepare-root.conf`, set in the image at build time), rebuilt from the image on every boot. A customer who enables it gives up persistent local `/etc` changes entirely. If your software writes config to `/etc` at runtime, know that this deployment variant exists.

### Rollbacks: `/etc` reverts, `/var` does not

Customers can roll back to the previous OS image with `bootc rollback`. The two writable areas behave differently when they do:

- **`/etc` reverts** to the prior deployment's state. Rollback reorders existing deployments, and each keeps its own `/etc`.
- **`/var` carries forward** unchanged. Your runtime data may now be "ahead" of both the OS and your config.

**What to do:** Treat `/var` data as potentially newer than the running code. If your app migrates its data format on upgrade, make the migration rollback-tolerant, or version your data files and fail cleanly rather than corrupting them. And don't keep state in `/etc` that must survive a rollback: state goes in `/var`, config goes in `/etc`.

## The mental model

Think of an image mode system like a phone or an appliance. The OS image is the firmware: it ships complete and read-only. Your app is part of that firmware, installed at build time, and its runtime data lives in a separate writable area (`/var`).

The customer's image build (Containerfile) is where your RPM gets installed. The running system is where your app does its work, reading from the read-only image and writing to `/var` and `/etc`.

If you follow one rule, follow this: **install files at build time, configure and run at boot time, write data to `/var`.**
