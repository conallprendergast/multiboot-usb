# Creating a Multiboot live USB!

## Introduction
This document provides instructions on how to create a bootable usb, that can be used to boot into one of many isos that exist on the usb. This usb will be both legacy bootable, and EFI bootable.

## Prerequisites
The following packages are required:

    grub2-pc grub2-efi-x64 shim-x64 efibootmgr


# Step 1 - Partitioning

## Useful commands:
    * **fdisk /dev/sdX**: Used for partitioning block devices
    * **mkfs.fat -F32 /dev/sdXN**: Used for creating a file system on a partition
    
## Instructions:
* The USB must use a GPT partition table. This can be achieved using the fdisk "g" command.
* Four partitions need to be created as follows (fdisk "n" creates a partition, "t" changes type)
    - BIOS boot partition (GPT type "21686148-6449-6E6F-744E-656564454649", fdisk type 4). 1 MiB, no filesystem
    - EFI system partition (GPT type "C12A7328-F81F-11D2-BA4B-00A0C93EC93B", fdisk type 1). 200 MiB, vfat filesystem
    - Linux boot partition (GPT type "0FC63DAF-8483-4772-8E79-3D69D8477DE4", fdisk type 20). 100MiB, vfat filesystem
    - Data partition (GPT type "0FC63DAF-8483-4772-8E79-3D69D8477DE4", fdisk type 20). All remaining space, vfat filesystem
    
    
It is also possible to combine the "Linux boot" partition and the "data" partition into one, if you want to have the grub files and isos on the same partition, but for the purposes of this guide we will assume that the intention is to store the boot files and isos on seperate partitions.


# Step 2 - Create a hybrid MBR
A hybrid MBR is neccessary in order to trick some legacy devices into booting from our USB. This can be done by executing 

    gdisk /dev/sdX
    
Type "r" to enter "Recovery/Transformation" mode.

Type "h" to create a hybrid MBR and follow to instructions. Add all our partitions to the new MBR and set the bootable flag on the last one.


# Step 3 - Installing Grub
Grub will be installed twice, once for legacy booting, and once for EFI booting. 

* Mount the second and third partitions on local filesystem (on folders "efi" and "boot" respectively)
    
        mkdir efi && mount /dev/sdX2 efi
        mkdir boot && mount /dev/sdX3 boot
    
* Install grub for legacy booting (note: grub-install may be named grub2-install on some systems)

        grub-install --removable --boot-directory=boot --efi-directory=efi /dev/sdX --target=i386-pc
    
* Install grub for EFI booting (note: grub-install may be named grub2-install on some systems)

        grub-install --removable --boot-directory=boot --efi-directory=efi /dev/sdX --target=x86_64-efi


## Step 3.1 - Testing Grub
Grub should now be installed on the USB in both legacy and UEFI mode. It should be possible to now boot into the USB.

You should see, after booting into the USB, that you are dropped into a grub shell ("grub> "). This is normal, and is due to the fact that you do not have a grub configuration file (grub.cfg) located in the boot partition of your USB.


# Step 4 - Adding our ISOs

ISO files can be added to our 4th partition in whatever directory you choose. For this example, I will use the directory "/isos".

    mount /dev/sdX4 /mnt

Add your isos into the desired folder, eg.

    /mnt/isos/clonezilla-live-2.5.5-38-amd64.iso
    /mnt/isos/CentOS-7-live-GNOME-x86_64.iso
    /mnt/isos/ubuntu-18.04-desktop-amd64.iso


# Step 5 - Configuring Grub
This is the most difficult, and hardest to debug stage. It is very dependant on the iso files that you wish to boot into. Note that not all iso files are compatible with multibooting.

Grub configuration files are very version and distro dependent. It is useful to look at other multiboot projects for inspiration, which can be found at the bottom of this article. It is very useful to extract the grub.cfg from the iso itself, and modify it to allow ISO booting. This is sometimes possible by simply adding one parameter eg. "iso-scan/filename=/path/to/iso"
    
Add a grub configuration file in the "grub" (or "grub2") folder in our boot partition.     

We need to tell grub the root partition in which to find isos. 
Get the UUID of the /dev/sdX4 partition:

    blkid /dev/sdX4
    
    
Add this into our new grub.cfg file to set a new "root" as the partition containing the isos:

    search --no-floppy --fs-uuid --set=root <UUID_from_above>
    export root
    
Set the isopath variable to reflect where we added our ISOs on our 4th partition

    set isopath=/isos
    export isopath
 
    
Add the grub configuration for the required ISOs
The grub config for the above three example isos are as follows:

**Clonezilla**:
[source]
menuentry "Clonezilla Live ${clonezillav} amd64 to RAM" --class clonezilla {
  set isoname="clonezilla-live-2.5.5-38-amd64.iso"
  set isofile="${isopath}/clonezilla/${isoname}"
  loopback loop $isofile
  linux (loop)/live/vmlinuz boot=live findiso=${isofile} union=overlay components quiet toram=live,syslinux
  initrd (loop)/live/initrd.img
}

**Centos 7**
[source]
menuentry "CentOS 7 Live GNOME" --class centos {
  set isoname="CentOS-7-live-GNOME-x86_64.iso"
  set isofile="${isopath}/centos/${isoname}"
  loopback loop $isofile
  linux (loop)/isolinux/vmlinuz0 root=live:CDLABEL=CentOS-7-live-GNOME-x86_64 rootfstype=auto ro rd.live.image quiet rhgb rd.luks=0 rd.md=0 rd.dm=0 iso-scan/filename=${isofile}
  initrd (loop)/isolinux/initrd0.img
}

**Ubuntu 18.04**
[source]
menuentry "Ubuntu 18.04 Live Desktop amd64" --class ubuntu {
  set isoname="ubuntu-18.04-desktop-amd64.iso"
  set isofile="${isopath}/ubuntu/${isoname}"
  loopback loop $isofile
  linux (loop)/casper/vmlinuz boot=casper iso-scan/filename=${isofile} quiet splash
  initrd (loop)/casper/initrd.lz
}
menuentry "Ubuntu 18.04 Install Desktop amd64" --class ubuntu {
  set isoname="ubuntu-18.04-desktop-amd64.iso"
  set isofile="${isopath}/ubuntu/${isoname}"
  loopback loop $isofile
  linux (loop)/casper/vmlinuz boot=casper iso-scan/filename=${isofile} quiet splash only-ubiquity
  initrd (loop)/casper/initrd.lz
}


This is by far the hardest step and can take quite a bit of tweaking to get correct. Sometimes the names of the "vmlinuz" file or the "initrd" file changes between versions, so it is often neccessary to mount the ISO to find the correct names.

If everything is done correctly you should now see, and be able to boot into, each of these isos.

# Other multiboot projects
https://github.com/mfaerevaag/multibootusb
https://github.com/thias/glim
