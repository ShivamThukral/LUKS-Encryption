# LUKS-Encryption
**Disk encryption using LUKS**
These steps should work on Ubuntu 18.04 LTS and newer versions. 
**What is LUKS?**
[The Linux Unified Key Setup (LUKS) is the standard for Linux hard disk encryption](https://gitlab.com/cryptsetup/cryptsetup/blob/master/README.md)

*Note: Before you also think of encrypting your disks, please make sure to backup your data.*
###Steps:
1. **Booting Ubuntu 18.04 live image :** It’s required to encrypt partitions before installing Ubuntu. Therefore, boot from Ubuntu 18.04 live image and chose “Try Ubuntu" option.

2. **Creating partitions : ** use GParted (already installed on Ubuntu 18.04 live image) to create the following partiions.
	- Boot Partition (label: boot, filesystem: ext4, size: 1GB)
	- Root Partition (label: rootfs, filesystem: ext4, size: user specified)
	- Data Partition (label: home, filesystem: ext4, size: user specified)
	- GRUB (label: GRUB, filesystem: ext4, size: 2MiB) 
	- EFI System Partition (label: EFI-SP, filesystem: fat32, size: 128MiB)
Once all partitions are successfully formed, we start to encrypt new partitions /dev/sdaX (rootfs) and /dev/sdaY (home) using the terminal.

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


5. Once logical volumes are created, start the Ubuntu installation from the shortcut icon on Desktop. Select“Something else” after initial steps are completed. 
	- Here, make sure that main disk is selected as the “Device for boot loader installation”
	- Select the created boot partion and select /boot as the mount point. Check the format partition option.
	- Select the logical volume /dev/mapper/vgroot-lvroot created to mount /. Check the format partition option. 
	- Similarly, select the logical volume /dev/mapper/vghome-lvhome created to mount /home and specified to format it with ext4. Check the format partition option.
	- Select EFI-SP partition and specific it has EFI File System.
After confirming to write the changes to the disk, the installation continued normally.

6. Once the installation is complete, click “Continue Testing” to make necessary changes to load the encrypted partitions at startup.
7. Used following commands to note down UUIDs of the encrypted partitions.
```
sudo blkid </dev/DEV_ROOTFS>
sudo blkid </dev/DEV_HOME>
```
8. Next, mount the installed Ubuntu OS on /mnt and used chroot command to change the root directory to /mnt.
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
9. Created a file named /etc/crypttab in the chrooted environment. Add following lines to /etc/crypttab (replacing <UUID_ROOTFS> and <UUID_HOME> from the values obtained from blkid command earlier).
```
sudo nano /etc/crypttab
# <target name> <source device> <key file> <options>
rootfs UUID=<UUID_ROOTFS> none luks,discard
home UUID=<UUID_HOME> none luks,discard
```
10. Finally, use following command to update the Linux kernel to load encrypted partitions at startup.
```
update-initramfs -k all -c
```
11. After restarting, Ubuntu should prompt you to enter the passphrase to unlock the disks at startup.
