# arch-boxes
[![CI Status](https://gitlab.archlinux.org/archlinux/arch-boxes/badges/master/pipeline.svg)](https://gitlab.archlinux.org/archlinux/arch-boxes/-/pipelines)

- [Vagrant Cloud](https://app.vagrantup.com/archlinux/boxes/archlinux)
- [Download latest qcow2 image](https://gitlab.archlinux.org/archlinux/arch-boxes/-/jobs/artifacts/master/download?job=build:cloud-qemu)

Arch-boxes provides automated builds of the Arch Linux releases for
different providers and post-processors. Check the providers or post-processor sections if you want to know
which are currently supported.

## Dependencies
You'll need the following dependencies:

* packer (for basic usage)
* vagrant (for vagrant images)
* qemu

## Variables
Here is an overview over all variables you can set in `config.json`:

* `iso_url`: the url to the ISO. This can be an url or a filepath
  beginning with `file://`
* `iso_checksum_url`: the url to the checksum file. This can be an url
  or a filepath beginning with `file://`
* `iso_checksum_type`: this specifies the hashing algorithm for the
  checksum.
* `disk_size`: this specifices the disk size in bytes.
* `headless`: this sets GUI on or off.
* `boot_wait`: this specifies the time packer should wait for booting up
  the ISO before entering any command.

## How to start the build process locally
If you want to build the boxes locally without uploading them to the Vagrant
cloud. You can start the build for virtualbox only with the following command:

`packer build -only=qemu config.json`

## How to start the build process for official builds
The official builds are done in our Arch Linux GitLab CI.

`packer build config.json`

## Providers

* virtualbox
* qemu/libvirt

## Post-processors

* vagrant (*.box)
* cloud (*.qcow2)

## Development workflow
Merge requests and general development shall be made on the `master` branch.
For security reasons, we're not allowing our CI runners on other branches than
`master` or `release` so MRs won't properly run their CI.

Therefore, the development flow is:

1. Make a MR against `master`.
2. After code review, a maintainer locally needs to build the changes.
3. The maintainer then needs to merge the changes to `master`.

## Release workflow
Releases are done automatically via [GitLab CI
schedule](https://gitlab.archlinux.org/archlinux/arch-boxes/-/pipeline_schedules).
No manual intervention is required or desired.

The release flow is:

1. A maintainer makes a MR from `master` against `release`.
2. The MR is merged on GitLab.
3. A release is done automatically and regularly from the `release` branch.

## Troubleshooting

### Parallel build fails
If the parallel build fails this is mostly because the KVM device is
already occupied by a different provider. You can use the build option
`parallel=false` for building the images in a queue instead of parallel.
But don't be surprised that that the build process will take longer. Any
other option is to disable KVM support for all other providers except
one.

Start `packer` with `-parallel=false`:

`packer build -parallel=false config.json`

### Checking cloud-init support in our qcow2 images:

```bash
$ packer build -only=cloud -except=sign config.json
$ cp Arch-Linux-cloudimg-2020-02-24.qcow2 disk.qcow2

# Copied from (with minor changes): https://cloudinit.readthedocs.io/en/latest/topics/datasources/nocloud.html
$ { echo instance-id: iid-local01; echo local-hostname: cloudimg; } > meta-data

$ printf "#cloud-config\npassword: passw0rd\nchpasswd: { expire: False }\nssh_pwauth: True\n" > user-data

## create a disk to attach with some user-data and meta-data (require cdrkit)
$ genisoimage -output seed.iso -volid cidata -joliet -rock user-data meta-data

## create a new qcow image to boot, backed by your original image
$ qemu-img create -f qcow2 -b disk.qcow2 boot-disk.qcow2

## boot the image and login as 'arch' with password 'passw0rd'
## note, passw0rd was set as password through the user-data above,
## there is no password set on these images.
$ qemu-system-x86_64 -m 256 \
   -net nic -net user,hostfwd=tcp::2222-:22 \
   -drive file=boot-disk.qcow2,if=virtio \
   -drive file=seed.iso,if=virtio
```
