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

### Whether `BindPaths=` into a read-only `/opt` starts cleanly

The `/opt` section offers `BindPaths=` in a shipped systemd unit as a way to get a writable directory into the read-only tree. The mechanism is documented: upstream bootc's build guidance and RHEL's image mode documentation both carry the same example line, and systemd's own documentation settles the semantics, including that the unit gets its own mount namespace and that the bind mount is writable unless the source mount is read-only. What none of those sources shows is the pattern running on a deployed system. Three things about it are reasoned rather than observed:

- Whether the bind mount succeeds at all when the destination sits on the composefs-backed read-only `/opt`. Mounting over an existing directory shouldn't need the underlying filesystem to be writable, and the directory is there because the package created it at build time, but that's an inference.
- What SELinux does with it. The source directory carries its own labels (`/var/log/...` is `var_log_t`), and a confined service reaching that content through an `/opt` path may be denied where it would have been allowed at the original path.
- Whether the source directory exists at all on a system that's already been deployed. Image content in `/var` lands only on the first deployment, so a unit shipping nothing but the two `BindPaths=` lines has no guarantee its sources are present, and a missing source fails the unit at namespace setup unless the definition is prefixed with `-`. A `tmpfiles.d` entry or `StateDirectory=` is the fix, and the `/opt` section doesn't currently say so.

**What would settle it:** on a booted system, `systemd-run -p BindPaths=/var/log/example:/opt/example/logs --pty /bin/bash`, then `findmnt /opt/example/logs` inside that shell and a write into it with SELinux enforcing, checking `ausearch -m avc -ts recent` afterward. That covers all three in one pass.

## If you have an entitled RHEL build host

### The kernel-module build example

The guide's Containerfile example builds a driver against the image's own kernel in a discarded builder stage, using DKMS. It has never actually been built. Every line traces back to a real source, but four of those sources are the DKMS project itself, or general coreutils and Containerfile documentation, rather than RHEL's own packaged DKMS specifically:

- Whether the packaged `dkms` self-signs modules by default inside a builder stage, and where the key material lands. The guide's Secure Boot passage makes the general point that a key generated inside a builder stage is discarded with the stage; if `dkms` generates one on its own by default, that is a concrete case worth naming.
- The exact path the built module lands at under `/var/lib/dkms/`
- Whether the pinned `kernel-devel-"$kver"` install resolves cleanly against RHEL's package naming
- Whether `install -D ... -t` creates the target directory tree the way the example assumes

**What would settle it:** one build of the example against `registry.redhat.io/rhel10/rhel-bootc:latest`. Three of the four come back from the build itself, since each fails loudly if the example is wrong: a build error, not something a partner ships and discovers later. The self-signing question needs one extra look inside the built image, `modinfo` on the module for a signature block and a glance at `/etc/dkms/framework.conf`.

One prerequisite that isn't obvious: the build has to run somewhere entitled. The base image ships with no repository configuration and no entitlement certificates of its own (`/etc/yum.repos.d/` and `/etc/pki/entitlement/` are both empty, and `dnf repolist` inside it reports no repositories), so it picks up RHEL content from a registered build host or from entitlement certificates mounted as build secrets. Pulling the image needs only a registry login; building the example needs entitlement. On an unregistered host the build stops at the first `dnf install` for want of repositories, which tells you nothing about whether the example is correct.

### Whether `dkms` really needs no CodeReady Builder packages

The guide's kernel-module section notes that some EPEL packages need the CodeReady Builder repository enabled, and that `dkms` itself doesn't. That claim was checked against CentOS Stream's package metadata, which mirrors RHEL's repository layout closely but isn't RHEL's own metadata.

**What would settle it:** on an entitled RHEL 10 system, `dnf repoquery --requires --resolve dkms` against your own repositories, not a Stream mirror. A successful build of the kernel-module example above settles it in passing too, since that build installs `dkms` with CRB disabled.

## Calls we haven't made yet

### Should `BindPaths=` be a recommendation, or an option we only describe?

The `/opt` section currently recommends it: the "What to do" line tells vendors to ship `BindPaths=` in their own unit when only their service writes to the path. The pattern is established, not invented here. Upstream bootc's build guidance and RHEL's image mode documentation both offer it as an alternative to the symlink pattern, and both stop at the example line. The scope caveat this guide adds after it, that the remap exists only inside that unit's mount namespace, appears in neither.

That caveat is what makes this a judgment call rather than a fact question. A reader who skims past it comes away thinking the path is writable. What they actually get is one path telling two different stories: the service writes successfully, while an admin running `ls` on the same path sees stale image content and concludes logging is broken, a log shipper aimed there collects nothing, and a helper CLI writing there fails read-only. In all three cases the data was fine and the path lied. The symlink pattern has no such split, which is why this guide leads with it and why upstream does too.

There's a wording question underneath. The caveat names the mechanism, then describes the consequence entirely from the outside, in terms of what other processes see. At least one careful reader came away wondering whether the mount namespace walls the service off from the rest of the filesystem. It doesn't: the service sees everything normally, and what it writes lands in the real `/var/log/<vendor>`, reachable by the whole host. Only the path alias is local to the unit.

**What would settle it:** two calls, in an issue or a pull request. Whether the "What to do" line keeps recommending `BindPaths=` or steps back to describing it as an option, and whether the caveat should lead with the reassuring half, that the service sees the whole filesystem and the data is in the real location, before naming what other processes see.

### Should transient root get a mention alongside state overlays?

When a vendor's `/opt` tree genuinely can't be restructured, the guide names one escape hatch, a state overlay, and treats it as a last resort. bootc documents a second, more drastic option: a transient root that makes the whole filesystem writable until reboot. This isn't a factual gap: the mechanism is documented and understood. It's a judgment call about whether naming a bigger hammer, in a section that already argues against reaching for the first one, helps readers or just adds noise.

**What would settle it:** a proposed sentence, in an issue or a pull request. If it reads as useful rather than as more to skim past, it goes in.

### A least-privilege note on the persisted pull-secret case

The credentials section draws a line between build-time secrets, which should never end up in a layer, and a secret that has to exist on the deployed host (a pull secret for a bound image, for example), which RHEL's own documentation deliberately persists in the image. The section opens by pointing out that anyone who can pull an image can read every file in it, and that's still true of a persisted pull secret.

**What would settle it:** nothing external: this one's ours to write. A sentence recommending the persisted credential be scoped to read-only pull access would close the gap. A pull request with proposed wording is the fastest path.
