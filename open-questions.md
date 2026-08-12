---
layout: default
title: Open questions
permalink: /open-questions/
---

This guide's core model, read-only image content, how `/var` and `/etc` behave, the three-way merge on upgrades, checks out against upstream bootc documentation and RHEL's own image mode documentation. What follows is narrower: specifics that need a live system or a real build to settle rather than documentation alone, plus a couple of wording calls that are ours to make.

Each entry names what the guide currently says, what's still open, and the smallest concrete thing that would close it. If you can answer one, open a pull request or an issue. Publishing pre-1.0 with the door open is the whole point.

## If you have a booted RHEL 10 system

### Does package-mode RHEL default to a tmpfs `/tmp`?

Does a traditional, package-mode RHEL 10 install default to a tmpfs `/tmp`, the way image mode does, or does it stay disk-backed? The filesystem table skips a `/tmp` row until that's settled.

**What would settle it:** `findmnt /tmp` and `systemctl is-enabled tmp.mount` on a booted RHEL 10 host installed in package mode.

### How SELinux labels behave across a deploy

There's no SELinux section in this guide. That's deliberate rather than an omission: whether labels get reapplied fresh at each deploy from the image's own policy, and whether customizing a label means changing policy in the build rather than running `restorecon` against a live system, isn't something we could confirm without a system to check it against.

**What would settle it:** boot an image, compare the label state against what the shipped policy expects (`matchpathcon`, `ls -Z`), and check whether a local relabel survives the next deployment or gets overwritten. That's enough for a short section.

## If you have an entitled RHEL build host

### The kernel-module build example

The guide's Containerfile example builds a driver against the image's own kernel in a discarded builder stage, using DKMS, and signs the result if you supply your own key. It has never actually been built. Every line traces back to a real source, but four of those sources are the DKMS project itself, or general coreutils and Containerfile documentation, rather than RHEL's own packaged DKMS specifically:

- Whether the packaged `dkms` self-signs modules by default inside a builder stage, and where the key material lands. The guide's Secure Boot paragraph is written to hold either way, so nothing in it is wrong if the answer turns out to be no; knowing the answer would just let it say more.
- The exact path the built module lands at under `/var/lib/dkms/`
- Whether the pinned `kernel-devel-"$kver"` install resolves cleanly against RHEL's package naming
- Whether `install -D ... -t` creates the target directory tree the way the example assumes

**What would settle it:** one build of the example against `registry.redhat.io/rhel10/rhel-bootc:latest`. Three of the four come back from the build itself, since each fails loudly if the example is wrong: a build error, not something a partner ships and discovers later. The self-signing question needs one extra look inside the built image, `modinfo` on the module for a signature block and a glance at `/etc/dkms/framework.conf`.

One prerequisite that isn't obvious: the build has to run somewhere entitled. The base image ships with no repository configuration and no entitlement certificates of its own (`/etc/yum.repos.d/` and `/etc/pki/entitlement/` are both empty, and `dnf repolist` inside it reports no repositories), so it picks up RHEL content from a registered build host or from entitlement certificates mounted as build secrets. Pulling the image needs only a registry login; building the example needs entitlement. On an unregistered host the build stops at the first `dnf install` for want of repositories, which tells you nothing about whether the example is correct.

### Whether `dkms` really needs no CodeReady Builder packages

The guide's kernel-module section notes that some EPEL packages need the CodeReady Builder repository enabled, and that `dkms` itself doesn't. That claim was checked against CentOS Stream's package metadata, which mirrors RHEL's repository layout closely but isn't RHEL's own metadata.

**What would settle it:** on an entitled RHEL 10 system, `dnf repoquery --requires --resolve dkms` against your own repositories, not a Stream mirror. A successful build of the kernel-module example above settles it in passing too, since that build installs `dkms` with CRB disabled.

## Calls we haven't made yet

### Should transient root get a mention alongside state overlays?

When a vendor's `/opt` tree genuinely can't be restructured, the guide names one escape hatch, a state overlay, and treats it as a last resort. bootc documents a second, more drastic option: a transient root that makes the whole filesystem writable until reboot. This isn't a factual gap: the mechanism is documented and understood. It's a judgment call about whether naming a bigger hammer, in a section that already argues against reaching for the first one, helps readers or just adds noise.

**What would settle it:** a proposed sentence, in an issue or a pull request. If it reads as useful rather than as more to skim past, it goes in.

### A least-privilege note on the persisted pull-secret case

The credentials section draws a line between build-time secrets, which should never end up in a layer, and a secret that has to exist on the deployed host (a pull secret for a bound image, for example), which RHEL's own documentation deliberately persists in the image. The section opens by pointing out that anyone who can pull an image can read every file in it, and that's still true of a persisted pull secret.

**What would settle it:** nothing external: this one's ours to write. A sentence recommending the persisted credential be scoped to read-only pull access would close the gap. A pull request with proposed wording is the fastest path.
