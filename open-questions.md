---
layout: default
title: Open questions
permalink: /open-questions/
---

This guide's core model, read-only image content, how `/var` and `/etc` behave, the three-way merge on upgrades, checks out against upstream bootc documentation and RHEL's own image mode documentation. What follows is narrower: the specific places where that verification stopped short of certainty, a detail the documentation didn't cover, a claim that only checks out against a neighboring source rather than the one we'd want, or an answer that's sitting on a real system somewhere, waiting for someone to check it.

Each entry names what the guide currently says, what's still open, and the smallest concrete thing that would close it. If you can answer one, open a pull request or an issue. Publishing pre-1.0 with the door open is the whole point.

## If you have a booted RHEL 10 bootc system

### `/tmp`: disk-backed, or shared like `/run`?

The filesystem table doesn't have a `/tmp` row. That's not an oversight: traditional RHEL ships `/tmp` disk-backed by default (a tmpfs `/tmp` is a default elsewhere, not on RHEL), so the table's usual "same as traditional" answer for `/run` doesn't automatically carry over, and nothing in the documentation we could reach settles what RHEL bootc images actually do here.

**What would settle it:** `findmnt /tmp` or `systemctl status tmp.mount` on a booted system. Whatever it reports goes straight into the table.

### How SELinux labels behave across a deploy

There's no SELinux section in this guide. That's deliberate rather than an omission: whether labels get reapplied fresh at each deploy from the image's own policy, and whether customizing a label means changing policy in the build rather than running `restorecon` against a live system, isn't something we could confirm without a system to check it against.

**What would settle it:** boot an image, compare the label state against what the shipped policy expects (`matchpathcon`, `ls -Z`), and check whether a local relabel survives the next deployment or gets overwritten. That's enough for a short section.

## If you can run one container build

### The kernel-module build example

The guide's Containerfile example builds a driver against the image's own kernel in a discarded builder stage, using DKMS, and signs the result if you supply your own key. It has never actually been built. Every line traces back to a real source, but four of those sources are the DKMS project itself, or general coreutils and Containerfile documentation, rather than RHEL's own packaged DKMS specifically:

- Whether the packaged `dkms` self-signs modules by default inside a builder stage, and where the key material lands. The guide's Secure Boot paragraph is written to hold either way, so nothing in it is wrong if the answer turns out to be no; knowing the answer would just let it say more.
- The exact path the built module lands at under `/var/lib/dkms/`
- Whether the pinned `kernel-devel-"$kver"` install resolves cleanly against RHEL's package naming
- Whether `install -D ... -t` creates the target directory tree the way the example assumes

**What would settle it:** one build of the example against `registry.redhat.io/rhel10/rhel-bootc:latest`. All four questions come back from the same run, and each one fails loudly if the example is wrong: a build error, not something a partner ships and discovers later.

## If you have access we don't

### The CodeReady Builder aside

The guide notes that some EPEL packages need the CodeReady Builder repository enabled, that `dkms` itself doesn't, and it points readers to Red Hat's own documentation for the exact command rather than naming one, because the repository id and how you enable it depend on how the build is entitled.

Two things sit behind that sentence. Red Hat's canonical article on enabling CodeReady Builder lives behind customer portal sign-in, so we could confirm the page exists but not read what it prescribes. And the claim that `dkms` doesn't need CRB was checked against CentOS Stream's package metadata, which mirrors RHEL's repository layout closely but isn't RHEL's own metadata.

**What would settle it:** with customer portal access, the CodeReady Builder article tells us whether the guide should name a repository id directly. With an entitled RHEL 10 system, `dnf repoquery --deplist dkms` against your own repositories, not a Stream mirror, closes the second question outright.

## Calls we haven't made yet

### Should transient root get a mention alongside state overlays?

When a vendor's `/opt` tree genuinely can't be restructured, the guide names one escape hatch, a state overlay, and treats it as a last resort. bootc documents a second, more drastic option: a transient root that makes the whole filesystem writable until reboot. This isn't a factual gap: the mechanism is documented and understood. It's a judgment call about whether naming a bigger hammer, in a section that already argues against reaching for the first one, helps readers or just adds noise.

**What would settle it:** a proposed sentence, in an issue or a pull request. If it reads as useful rather than as more to skim past, it goes in.

### A least-privilege note on the persisted pull-secret case

The credentials section draws a line between build-time secrets, which should never end up in a layer, and a secret that has to exist on the deployed host (a pull secret for a bound image, for example), which RHEL's own documentation deliberately persists in the image. The section opens by pointing out that anyone who can pull an image can read every file in it, and that's still true of a persisted pull secret.

**What would settle it:** nothing external: this one's ours to write. A sentence recommending the persisted credential be scoped to read-only pull access would close the gap. A pull request with proposed wording is the fastest path.
