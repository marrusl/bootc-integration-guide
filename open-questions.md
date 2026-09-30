---
layout: default
title: Open questions
permalink: /open-questions/
---

This guide's core model, read-only image content, how `/var` and `/etc` behave, the three-way merge on upgrades, checks out against upstream bootc documentation and RHEL's own documentation for image mode. What follows is narrower: specifics that need a live system or a real build to settle rather than documentation alone, plus a couple of wording calls that are ours to make.

Each entry names what the guide currently says, what's still open, and the smallest concrete thing that would close it. If you can answer one, open a pull request or an issue. Publishing pre-1.0 with the door open is the whole point.

## If you have a booted RHEL 10 system

### How SELinux labels behave across a deploy

There's no SELinux section in this guide. That's deliberate rather than an omission: whether labels get reapplied fresh at each deploy from the image's own policy, and whether customizing a label means changing policy in the build rather than running `restorecon` against a live system, isn't something we could confirm without a system to check it against.

**What would settle it:** boot an image, compare the label state against what the shipped policy expects (`matchpathcon`, `ls -Z`), and check whether a local relabel survives the next deployment or gets overwritten. That's enough for a short section.

### Whether a confined service can write through the `/opt` symlink

The `/opt` section tells you to move writable directories into `/var` and symlink them back. The content then carries the label of its new home (`/var/log/...` is `var_log_t`), not whatever label the package's `/opt` tree had. A confined service reaching that content through its old `/opt` path may be denied where it would have been allowed at the original path. Upstream flags the same class of risk for the analogous pattern of symlinking a config file from `/etc` into `/usr`. The `/opt` section currently gives the fix without this caveat, because we don't know whether it bites in practice or only in principle.

**What would settle it:** on a booted system with SELinux enforcing, apply the symlink pattern to a confined service, write through the `/opt` path, and check `ausearch -m avc -ts recent`. If a policy adjustment turns out to be needed, that's a sentence the `/opt` section is missing.

## If you have an entitled RHEL build host

### The kernel-module build example

The guide's Containerfile example builds a driver against the image's own kernel in a discarded builder stage, using DKMS. It has never actually been built. Every line traces back to a real source, but four of those sources are the DKMS project itself, or general coreutils and Containerfile documentation, rather than RHEL's own packaged DKMS specifically:

- Whether the packaged `dkms` self-signs modules by default inside a builder stage, and where the key material lands. The guide's Secure Boot passage makes the general point that a key generated inside a builder stage is discarded with the stage; if `dkms` generates one on its own by default, that is a concrete case worth naming.
- The exact path the built module lands at under `/var/lib/dkms/`
- Whether the pinned `kernel-devel-"$kver"` install resolves cleanly against RHEL's package naming
- Whether `install -D ... -t` creates the target directory tree the way the example assumes

**What would settle it:** one build of the example against `registry.redhat.io/rhel10/rhel-bootc:latest`. Three of the four come back from the build itself, since each fails loudly if the example is wrong: a build error, not something a partner ships and discovers later. The self-signing question needs one extra look inside the built image, `modinfo` on the module for a signature block and a glance at `/etc/dkms/framework.conf`. A successful build also confirms a related claim in the guide: that `dkms` itself needs no CodeReady Builder packages, since the build installs it with only default repos plus EPEL.

One prerequisite that isn't obvious: the build has to run somewhere entitled. The base image ships with no repository configuration and no entitlement certificates of its own (`/etc/yum.repos.d/` and `/etc/pki/entitlement/` are both empty, and `dnf repolist` inside it reports no repositories), so it picks up RHEL content from a registered build host or from entitlement certificates mounted as build secrets. Pulling the image needs only a registry login; building the example needs entitlement. On an unregistered host the build stops at the first `dnf install` for want of repositories, which tells you nothing about whether the example is correct.

### The walkthrough for getting an image has not been run end to end

The [Getting a RHEL bootc image]({{ '/' | relative_url }}#getting-a-rhel-bootc-image) section is assembled from Red Hat's registry authentication documentation, the RHEL documentation for image mode, and three live checks against the registries: `rhel-bootc` returns `UNSUPPORTED` on `registry.access.redhat.com` and `401` unauthenticated on `registry.redhat.io`, and the CentOS Stream bootc tags are public on Quay. Nobody has walked the whole path on a fresh account: sign up, log in, pull, build.

Two specifics that a real run would settle:

- Whether a Red Hat Developer Program account is sufficient to pull `rhel10/rhel-bootc`, or whether the terms acceptance in that error message is a step of its own that a developer account does not clear.
- The rootless-versus-root credential mismatch. The credential paths come from `containers-auth.json(5)` and the conclusion follows from them, but the failure has not been reproduced, so the error text a partner actually sees is not quoted.
- Whether a Fedora, CentOS Stream, or RHEL rebuild host registered with `subscription-manager` passes its entitlement into a build the way a registered RHEL host does. On Fedora and CentOS Stream, `containers-common` installs the same `mounts.conf` and `/usr/share/rhel/secrets` symlinks RHEL uses (checked against the Fedora rawhide and CentOS Stream 10 spec files, 2026-09-29), so the packaging says yes. Nobody has run the build.
- Whether signing in through the Red Hat Authentication extension for Podman Desktop on macOS or Windows is enough for `dnf install` inside a `rhel-bootc` build. The extension's own README registers the Podman machine with `subscription-manager` and installs protected content in a `rhel9/toolbox` build, which is the same mechanism. A `rhel-bootc` build has not been run through it.

**What would settle it:** one pass through the section on a machine with no Red Hat credentials on it yet, noting anywhere the steps don't match what happens. The tag question travels with it: the section deliberately points at the Ecosystem Catalog instead of listing tags, and if a stable minor-version tag scheme turns out to exist it is worth naming.

## If you have a hypervisor

### The test VM page has not been run end to end

[Booting your image in a test VM]({{ '/test-vm/' | relative_url }}) condenses the disk-image and KVM procedures from the RHEL documentation, and every command on it traces there, to bcvk's own documentation, or to the Podman Desktop bootc extension's README. Three of its claims go one step past what those sources say:

- The installer ISO on a hypervisor other than KVM. RHEL 10 lists the ISO type as Technology Preview, and the page names Parallels as the case in mind without having booted it there.
- A `config.toml` user on a VMDK attached directly to Workstation or Fusion, without the cloud-init metadata the vSphere procedure supplies.
- bcvk against `rhel-bootc`. Its README examples use Fedora and CentOS Stream images, and the RHEL images have not been run through `bcvk ephemeral run-ssh` for this guide.

**What would settle it:** one run of each, on the hypervisor in question, noting where the steps or the output differ from the page. The ISO on Parallels is the most useful, since it is the page's answer for every hypervisor without a disk-image type of its own.

## Calls we haven't made yet

### Should transient root get a mention alongside state overlays?

When a vendor's `/opt` tree genuinely can't be restructured, the guide names one escape hatch, a state overlay, and treats it as a last resort. bootc documents a second, more drastic option: a transient root that makes the whole filesystem writable until reboot. This isn't a factual gap: the mechanism is documented and understood. It's a judgment call about whether naming a bigger hammer, in a section that already argues against reaching for the first one, helps readers or just adds noise.

**What would settle it:** a proposed sentence, in an issue or a pull request. If it reads as useful rather than as more to skim past, it goes in.
