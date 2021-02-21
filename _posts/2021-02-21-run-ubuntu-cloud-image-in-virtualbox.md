---
layout: post
title:  "Run Ubuntu cloud image in Virtualbox"
date:   2021-02-21 15:30:00
categories: ubuntu cloud-image virtualbox user-data cloud-init genisoimage
---

A cloud image is prepared differently than the standard server edition to include [cloud-init](https://cloudinit.readthedocs.io/en/latest/) for instantiation in cloud environment. Cloud-init allows images to gather the necessary configuration data (e.g. hostname, password, n/w config etc.) during boot (from a cloud metadata service). Canonical, the developer of popular Ubuntu Linux distro, provides free pre-built cloud images for their server edition. They are a great way to setup a Ubuntu VM in cloud.

But they same flexibility is not available for VMs that are deployed on non-cloud environments. For example, booting Ubuntu cloud image in [Oracle Virtualbox](https://www.virtualbox.org/) results in a VM that you can't login to, as there is no default password applied to the pre-built image due to security reasons and there is no metadata service to provide the config during boot either. So we are forced to use standard server image and configure everything ourselves or use other automation tools like Vagrant.

With newer versions of cloud-init, it is possible to supply the necessary configuration via a virtual CD during the first boot of the image. Here I will try to explain the process with **Ubuntu 20.04 server cloud image** and **Virtualbox 6.1**.

### Get the cloud image
Download Ubuntu 20.04 cloud image from following location - https://cloud-images.ubuntu.com/focal/current/
Remember to take the amd64 OVA image that says "VMware/Virtualbox OVA" (size is around 500M)

### Prepare the ISO image content with user config
This step requires you to have a Linux machine with `genisoimage` tool. It is installed by default in my Ubuntu 18.04 desktop VM.
* Start by creating a directory and 2 files in it -
   ```
   seed-iso
   - meta-data
   - user-data
   ```
   Here the `seed-iso` is the root directory of the ISO and it contains 2 files named `user-data` and `meta-data`.
* Add following content in meta-data file
   ```
   local-hostname: ubuntu.local
   ```
   You can select any other hostname than ubuntu.local
* Add following content in user-data file
   ```
   #cloud-config
   password: pass123
   chpasswd: { expire: False }
   ssh_pwauth: True
   ```
   You can select any other password than *pass123*. In absence of `ssh_pwauth: True` you won't be able to login to the VM via SSH.

### Generate ISO image
Run the following command from *seed-iso* directory created above -
`genisoimage  -output seed.iso -volid cidata -joliet -rock user-data meta-data`
It will create **seed.iso** file in the same directory. Export this ISO file to the host where Virtualbox is installed.

### Load OVA image in Virtualbox
Open the OVA image, downloaded before, from Virtualbox and import. Once imported, we can change the name of the VM and network interfaces (I use NAT and Host-only). If you get a warning for Display Setting of the VM, change the Graphics Controller to VMSVGA.

Next to load the seed.iso, open VM Settings, go to Storage, Controller IDE and Add a Optical Drive. Browse the seed.iso.

If you want to enable Nested Virtualization for the VM and the option is greyed out in GUI, run following command `VBoxManage modifyvm vm-name --nested-hw-virt on`

Then start the VM and you can login to the VM using default user **ubuntu** and the password you have set in user-data file.

