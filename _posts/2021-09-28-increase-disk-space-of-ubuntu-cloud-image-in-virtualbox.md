---
layout: post
title:  "Increase disk space of Ubuntu cloud image in Virtualbox"
date:   2021-09-28 10:30:00
categories: ubuntu cloud-image virtualbox disk-resize
---

Ubuntu [cloud images](https://cloud-images.ubuntu.com/) are prepared with 10GB disk size by default. But this might be very less in most of the cases so, we need a way to resize (increase) the disk.
This is generally not a problem if you use the cloud image with OpenStack or other cloud providers, as they provide option to specify disk size while creating a VM (which is then used by cloud-init to resize the disk during first boot). But the same is not possible in VirtualBox as it doesn't allow to increase vdi disk size from user interface.

Following process can be used to resize the vdi disk size.
Open a command prompt window and run the below command where `ubuntu-focal-20.04-cloudimg.vdi` is name of the vdi file to be resized and 20000 is new size in MB (20GB in this case).
```
"c:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifymedium ubuntu-focal-20.04-cloudimg.vdi --resize 20000
```

Once the above step is complete, boot the VM for first time. VM will get increased disk space.

#### If VM is run before resizing the disk
The cloud-init based disk resize only happens for the first boot of the VM. So, if VM was run without increased disk initially then below steps are needed to be followed -
First stop the VM and run `VBoxManage.exe modifymedium` command mentoned above. It will resize the vdi disk. Then boot the VM and login to its shell. In VM shell run below command -
Find the disk that needs to be resized using `fdisk -l`. As example -
```
Disk /dev/sda: 30 GiB, 32212254720 bytes, 62914560 sectors
Disk model: HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 8D39E52D-3E8A-46DC-948D-F939AA9E611F

Device      Start      End  Sectors  Size Type
/dev/sda1  227328 62914526 62687199 29.9G Linux filesystem
/dev/sda14   2048    10239     8192    4M BIOS boot
/dev/sda15  10240   227327   217088  106M EFI System
```

Then row the linux partition with `growpart /dev/sda 1` where /dev/sda is the disk found in previous command and 1 denotes first partition will be increased (here /dev/sda1).
Finally we can run `resize2fs /dev/sda1` to reflect the changes in filesystem. Also need to reboot the VM once after this.

Once the VM is running again, we can check the increased size with `df -h`
