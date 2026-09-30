---
layout: default
title: Booting your image in a test VM
permalink: /test-vm/
---

You have built an image with your package in it. Before you can say anything about how it behaves, it has to boot, and a virtual machine is where nearly all of that testing happens. Everything [the guide]({{ '/' | relative_url }}) explains, except Secure Boot module signing and real hardware probing, shows up in a VM exactly as it does on metal.

Three things to know before picking a route:

- The container image becomes a VM either directly, with bcvk, or by way of a disk image, with bootc-image-builder. bcvk reads your own container storage, so a rootless `podman build` is enough. The builder reads root's: for that path build with `sudo podman build`, or hand the builder a registry reference after a `sudo podman login` to that registry.
- A disk image needs a user and an SSH key baked in, or there is no way to log in. The builder takes them from a `config.toml`; bcvk injects a key for you.
- Once booted, the machine updates from a registry, not from the build host, so iterating means pushing the rebuilt image somewhere the VM can reach. bcvk can bind the host's container storage into the VM instead.

Two tools cover it. bcvk boots the container image straight into a VM in one command and is the fastest way to see your package running. bootc-image-builder makes the disk image your customers will deploy, with an output type per hypervisor, and is the path you claim support against. Start with bcvk, and come back to the builder when you need a disk image or a hypervisor other than KVM.

## The image you will boot

If you do not have a Containerfile yet, this is the smallest complete one. Each line is a section of the guide:

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:latest

# Your repo and its signing key, shipped as files
COPY myvendor.repo /etc/yum.repos.d/
COPY RPM-GPG-KEY-myvendor /etc/pki/rpm-gpg/

RUN dnf -y install myvendor-agent && dnf clean all
RUN systemctl enable myvendor-agent.service

RUN bootc container lint
```

A package with writable directories under `/opt` adds the symlink lines from [the `/opt` section]({{ '/' | relative_url }}#opt-is-read-only-at-runtime) after the install. Build it with `podman build -t localhost/myvendor-test .`, and the rest of this page uses that name.

## Fastest: boot the image with bcvk

[bcvk](https://github.com/bootc-dev/bcvk) is the bootc project's virtualization kit. It boots a container image straight into a VM, with no disk image step and no `config.toml`:

```
$ sudo dnf install bcvk
$ bcvk ephemeral run-ssh localhost/myvendor-test:latest
```

The install line works as is on Fedora 42 and later. On RHEL 9 or 10, [enable EPEL](https://docs.fedoraproject.org/en-US/epel/getting-started/) first. The host also needs QEMU/KVM, `virtiofsd`, and Podman; bcvk's [installation page](https://github.com/bootc-dev/bcvk/blob/main/docs/src/installation.md) has the full list. The second line is a shell inside a VM booted from your image, gone when you exit.

Its [quick start](https://github.com/bootc-dev/bcvk/blob/main/docs/src/quick-start.md) covers the rest. Two commands worth knowing here: `bcvk libvirt run --name myvendor-test localhost/myvendor-test:latest` creates a VM that persists, with `bcvk libvirt ssh myvendor-test` to get in, and adding `--update-from-host` to that run binds your container storage into the VM so a `bootc upgrade` inside it picks up the rebuilt image without a registry.

Where it stands: bcvk is an upstream project from the bootc team, packaged in Fedora and EPEL, Linux only. It is not commercially supported yet, and support is planned. The team wants to hear about bugs and feature requests, so file them in the [issue tracker](https://github.com/bootc-dev/bcvk/issues). Use it for the loop of edit, rebuild, boot, look. When you write down what you support, test it through the path below as well, since that is the artifact your customers deploy.

## The supported path: a disk image from bootc-image-builder

This is the RHEL-documented flow, condensed from [Creating QEMU disk images](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/creating-bootc-compatible-base-disk-images-by-using-bootc-image-builder#creating-qcow2-images-by-using-bootc-image-builder) and [Deploying with KVM](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/deploying-the-rhel-bootc-images#deploying-a-container-image-by-using-kvm-with-a-qcow2-disk-image) in the RHEL documentation. It is the path for a disk image you can hand to any hypervisor, and the one to test against before you claim support.

Write a `config.toml` next to your Containerfile with the login you will use:

```toml
[[customizations.user]]
name = "test"
key = "ssh-ed25519 AAAA... you@example.com"
groups = ["wheel"]
```

Make the image visible to root, then run the builder. It writes `./output/qcow2/disk.qcow2`:

```
$ sudo podman build -t localhost/myvendor-test .
$ mkdir -p ./output
$ sudo podman run --rm -it --privileged --pull=newer \
    --security-opt label=type:unconfined_t \
    -v /var/lib/containers/storage:/var/lib/containers/storage:Z \
    -v ./config.toml:/config.toml:Z \
    -v ./output:/output:Z \
    registry.redhat.io/rhel10/bootc-image-builder:latest \
    --type qcow2 --config /config.toml \
    localhost/myvendor-test:latest
```

Boot it:

```
$ sudo virt-install --name myvendor-test --memory 4096 --vcpus 2 \
    --disk ./output/qcow2/disk.qcow2 --import
```

`virt-install` attaches a console, and `sudo virsh domifaddr myvendor-test` gives you the address to SSH to as `test`. The builder image comes from `registry.redhat.io`, so this step needs the registry login from [Getting a RHEL bootc image]({{ '/' | relative_url }}#getting-a-rhel-bootc-image) and not the entitlement: nothing is installed here, only converted. The full list of things `config.toml` can set, including partition sizes, is in [Supported image customizations](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/creating-bootc-compatible-base-disk-images-by-using-bootc-image-builder#supported-image-customizations-for-a-configuration-file).

## VMware

vSphere takes a VMDK. The builder command above with `--type vmdk` writes `./output/vmdk/disk.vmdk`, and the `config.toml` user works there too. Workstation and Fusion can attach that disk to a new VM directly. For vSphere itself, the [vSphere section](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/deploying-the-rhel-bootc-images#deploying-a-container-image-and-creating-a-rhel-virtual-machine-in-vsphere) of the RHEL documentation covers the import with `govc` and the cloud-init metadata a production VM wants; a test VM does not need the metadata.

## Anything else: an installer ISO

For a hypervisor with no disk-image type of its own, the builder can produce an installer ISO. The builder command above with `--type anaconda-iso` writes `./output/bootiso/install.iso`, which boots on anything that boots a RHEL ISO and installs your image unattended; see [Creating bootable ISOs](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/creating-bootc-compatible-base-disk-images-by-using-bootc-image-builder#creating-iso-images-by-using-bootc-image-builder) for the Kickstart it embeds. RHEL 10 lists the ISO type as Technology Preview, so treat it as a test convenience rather than the path you claim support against.

## On a Mac

Podman Desktop's [bootc extension](https://github.com/podman-desktop/extension-bootc) wraps the builder: pick the image, choose a disk type, and a Create VM button on the Disk Images page boots the result on macOS and Linux. It needs the Podman machine in rootful mode, and on Apple silicon it builds only for the machine's own architecture, so what you test is an aarch64 build of your product, which may not exist. Windows builds the disk image but cannot launch the VM yet. Parallels and UTM boot the installer ISO like any other RHEL ISO, provided it is an aarch64 build, which on Apple silicon is the only kind the builder makes.

## What to look at once it boots

The build going green and the VM booting are not the tests. These are, each tied to the section of the guide that predicted it:

```
$ sudo bootc status
$ systemctl status myvendor-agent
$ journalctl -u myvendor-firstboot
$ ls -l /opt/myvendor
$ ls -la /var/lib/myvendor /var/log/myvendor
$ sudo touch /usr/probe; sudo touch /opt/myvendor/probe
```

- `bootc status` names the image the machine booted from. Its digest is what goes in a support statement.
- Your service is enabled and running without anyone having run `systemctl start`. If it is not enabled, look for the guarded scriptlet in [the build environment]({{ '/' | relative_url }}#the-build-environment-is-a-container-not-a-booted-system).
- The first-boot unit ran once, exited cleanly, and left its stamp file: [first boot]({{ '/' | relative_url }}#do-machine-specific-setup-at-first-boot).
- The `/opt` entries that need to be writable are symlinks into `/var`, and their targets exist: [`/opt`]({{ '/' | relative_url }}#opt-is-read-only-at-runtime).
- Both `touch` commands fail with a read-only filesystem error. That is the model working: [the one thing to understand first]({{ '/' | relative_url }}#the-one-thing-to-understand-first).

Then make the machine behave like a customer's. Reboot it and confirm the first-boot unit did not run again. Rebuild the image with a changed default config file and a new file under `/var/lib/myvendor`, push it, and upgrade. A VM built from a local image needs to be pointed at the registry once:

```
$ sudo bootc switch quay.io/yourorg/myvendor-test:latest
$ sudo bootc upgrade --apply
```

After it comes back: the changed default applies where `/etc` was untouched and does not where it was edited, per [the `/etc` merge]({{ '/' | relative_url }}#your-defaults-their-customizations-and-the-etc-merge). The new `/var` file is not there, because `/var` belongs to the machine after first deployment, per [the `/var` rules]({{ '/' | relative_url }}#var-starts-from-the-image-then-belongs-to-the-machine). And `sudo bootc rollback` followed by a reboot puts `/etc` back and leaves `/var` alone, per [rollbacks]({{ '/' | relative_url }}#rollbacks-etc-reverts-var-does-not).

## Where the RHEL documentation goes further

None of this changes what your package has to do. That is the same on every target. When a customer asks about a route this page does not cover, the RHEL documentation has it:

- [AWS, with an AMI](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/deploying-the-rhel-bootc-images#deploying-a-container-image-to-aws-with-an-ami-disk-image)
- [Bare metal, with `bootc install` from a booted ISO](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/deploying-the-rhel-bootc-images#deploying-a-container-image-by-using-bootc)
- [Kickstart and custom partitioning in the ISO](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/creating-bootc-compatible-base-disk-images-by-using-bootc-image-builder#creating-iso-images-by-using-bootc-image-builder)

A handful of steps on this page have not been run end to end yet. See [open questions]({{ '/open-questions/' | relative_url }}) for which ones.
