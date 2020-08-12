# LUKS-Encryption
**Disk encryption using LUKS** <br/>
These steps should work on Ubuntu 18.04 LTS and newer versions. <br/>
**What is LUKS?** <br/>
[The Linux Unified Key Setup (LUKS) is the standard for Linux hard disk encryption](https://gitlab.com/cryptsetup/cryptsetup/blob/master/README.md)

*Note: Before you start encrypting your disks, please make sure to backup your data.* <br/>
### Steps:
1. **Booting Ubuntu 18.04 live image :** It’s required to encrypt partitions before installing Ubuntu. Therefore, boot from Ubuntu 18.04 live image and chose “Try Ubuntu" option.

2. **Creating partitions :** use GParted (already installed on Ubuntu 18.04 live image) to create the following partiions.
	- Boot Partition (label: boot, filesystem: ext4, size: 1GB)
	- Root Partition (label: rootfs, filesystem: ext4, size: user specified)
	- Data Partition (label: home, filesystem: ext4, size: user specified)
	- GRUB (label: GRUB, filesystem: ext4, size: 2MiB) 
	- EFI System Partition (label: EFI-SP, filesystem: fat32, size: 128MiB)
Once all partitions are successfully formed, we start to encrypt new partitions /dev/sdaX (rootfs) and /dev/sdaY (home) using the terminal.

** Aside: **

- ** Why 5 partitions? **
	- ** Boot Partition: ** All the files required to boot your computer is kept on different partition, called the Boot partition. 
	- ** Root Partitiion: ** This is where the OS will be installed. You really don’t need to give lots of space to root partition. In my experience 100 GB is more than enough.
	- ** Data Partition: ** If you have chosen to hold your data in the same partition as the system partition, skip this step and proceed. I would strongly recommend you to keep data and system partion separate.
	- ** GRUB Partitiion: **  GRUB bootloader is the software that loads the Linux kernel. You'll be prompted by the GRUB menu which can contain a list of the operating systems installed (in the case of dual boot.)
	- ** EFI System Partion:**  A special partition required for a computer with UEFI to be able to boot. This step is only if your computer doesn't already have an ESP. If your computer already has an ESP, skip this step and proceed.

- ** Number of drives? **
	- If you have two drives (like my case), install Boot, Root, GRUB and EFI System Partition on the SDD and Data Partiion on the HDD. 

- ** Why Boot Parition in separate? **
Initially, I had Boot and Root partition mounted on a single partition. With this partition scheme I was not able to successfully install Ubuntu. Using an encrypted Boot directory make things more complicated and I decided to keep my Boot directory unencrypted.
One benefit of having a separate Boot partition from the regular Root partition is that you can reduce on-disk file system complexity, which reduces the demands on the boot loader to bootstrap the kernel.
 
3. **Encrypting the partitions:**
The cryptsetup command is used to initialize LUKS partitions. After initializing a LUKS partition, the partition was opened providing the passphrase.
```
sudo cryptsetup luksFormat --hash=sha512 --key-size=512 /dev/sdaX 
sudo cryptsetup open --type=luks /dev/sdaX rootfs
sudo cryptsetup luksFormat --hash=sha512 --key-size=512 /dev/sdaY
sudo cryptsetup open --type=luks /dev/sdaY home
```
4. **Creating logical volumes to install Ubuntu and create Home directory**
Create logical volumes on top of LUKS volumes to install Ubuntu on root partition and create home directory in home partition.
```
sudo pvcreate /dev/mapper/rootfs
sudo vgcreate vgroot /dev/mapper/rootfs
sudo lvcreate -n lvroot -l 100%FREE vgroot
sudo pvcreate /dev/mapper/home
sudo vgcreate vghome /dev/mapper/home
sudo lvcreate -n lvhome -l 100%FREE vghome
```


5. Once logical volumes are created, start the Ubuntu installation from the shortcut icon on Desktop. Select “Something else” after initial steps are completed. 
	- Here, make sure that the main disk is selected as the “Device for boot loader installation”
	- Select the created boot partion and select /boot as the mount point. Check the format partition option.
	- Select the logical volume /dev/mapper/vgroot-lvroot created to mount /. Check the format partition option. 
	- Similarly, select the logical volume /dev/mapper/vghome-lvhome created to mount /home and specified to format it with ext4.
	- Select EFI-SP partition and specify it as EFI File System.
After confirming to write the changes to the disk, the installation should continue normally.

6. Once the installation is complete, click “Continue Testing” to make necessary changes to load the encrypted partitions at startup.
7. Use the following commands to note down UUIDs of the encrypted partitions.
```
sudo blkid </dev/DEV_ROOTFS>
sudo blkid </dev/DEV_HOME>
```
8. Next, mount the installed Ubuntu OS on /mnt and use chroot command to change the root directory to /mnt.
```
sudo mount /dev/mapper/vgroot-lvroot /mnt
sudo mount </dev/DEV_BOOT> /mnt/boot
sudo mount /dev/mapper/vghome-lvhome /mnt/home
sudo mount --bind /dev /mnt/dev
sudo chroot /mnt
mount -t proc proc /proc
mount -t sysfs sys /sys
mount -t devpts devpts /dev/pts
```
9. Create a file named /etc/crypttab in the chrooted environment. Add following lines to /etc/crypttab (replacing <UUID_ROOTFS> and <UUID_HOME> from the values obtained from blkid command earlier).
```
# <target name> <source device> <key file> <options>
rootfs UUID=<UUID_ROOTFS> none luks,discard
home UUID=<UUID_HOME> none luks,discard
```
10. Finally, use following command to update the Linux kernel to load encrypted partitions at startup.
```
update-initramfs -k all -c
```

** Aside : **
** Steps 6-10 : ** These steps help you load the encrypted partitions at the startup. Following these steps to update the Linux kernel to load the partitions. You just need to enter the parapharase once to unclock the whole system.

11. After restarting, Ubuntu should prompt you to enter the passphrase to unlock the disks at startup.

#### References: <br/>
[1] : [Full_Disk_Encryption_Howto_2019](https://help.ubuntu.com/community/Full_Disk_Encryption_Howto_2019) <br/>
[2] : [Encrypting disks on Ubuntu 19.04](https://medium.com/@chrishantha/encrypting-disks-on-ubuntu-19-04-b50bfc65182a)
