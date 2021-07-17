# True Full Disk Encryption with a Removable Key

#### Disk Encryption with a Detached LUKS Header and Encrypted Boot Partition on a Removable USB Key

##### Author: ech0r  

##### Date: 2020-06-11

### DEFINITIONS: 

#### LUKS - Linux Unified Key Setup - a robust disk encryption specification for Linux, platform independent standard.  

#### U/EFI - (Unified) Extensible Firmware Interface - replacement for BIOS, this is how modern machines boot. UEFI and EFI are used interchangeably.

#### MFA - Multi-Factor Authentication - a paradigm of using multiple methods to ensure the right person has access to whatever they're accessing.

#### USB drive/key - a small flash memory device that plugs into the USB ports on your computer.

#### Internal drive - the internal drive of your computer, preferably an SSD.

#### LVM - Logical Volume Manager - A cool software abstraction layer between your partitions and your physical devices. It allows you some flexibility with your disk setups. This guide is centered around an LVM on top of LUKS approach.

#### LV - Logical Volume, similar to a disk partition

### INTRODUCTION:

This is a true full disk encryption scheme designed to keep your data safe and by default provides some assurance that your system is in a trustworthy state each time you use it. This setup combines physical MFA (the usb key) combined with the passphrases used to encrypt it. The usb key in this setup functions like the 'key' to your computer, in the same way a car requires a key to start. The usb key is required to boot the machine, without this key, the data on the internal drive is inaccessible. This setup will give you security similar to the 'smartcard' setup found on high-security enterprise and government systems.

Some definitions:

### FAQs:

#### 1. Why would you want this setup? 
	- This is the most secure data-at-rest encryption scheme I can think of. 
	- Digital Privacy is something that is under constant threat - why make it easier for the powers that be?
	- To someone viewing the contents of your internal drive, it would appear unused, nothing indicates that an encrypted volume even exists on the disk.

#### 2. Who would want this setup?
	- Political dissidents (e.g. protesters, whistleblowers)
	- People under surveillance (e.g. citizens of regions with oppressive governments)
	- Any person concerned with the theft of their devices by a malicious actor with the tools to retrieve their personal data.
	
#### 3. How does this setup work?
	- The USB drive is the equivalent of a car key for a car. The pc is unable to boot without it. 
	- If the USB drive is lost or damaged, the data on your internal drive is lost forever.
	- Additionally, the USB drive functions like a physical multi-factor device. 
	- You must physically possess the USB drive AND know the passphrase to decrypt it. Without both of these, the data on the internal drive is inaccessible. 
	- The internal drive is encrypted with a large keyfile instead of a passphrase. This makes it essentially impossible to brute-force with today's technology.
	
#### 4. How does this setup keep my data safe?
	- Without the USB drive, none of the data on your internal drive can be accessed. The tools simply do not exist yet to break encryption like this.
	- Additionally, since the LUKS header is detached and stored on the USB drive, it's not even apparent that the internal drive is being used for anything, this provides some extra security through obscurity.
	
#### 5. Why would I want this setup over traditional 'full disk' encryption? 
	- Most 'full disk' encryption schemes aren't really FULL disk, they operate with an unencrypted boot partition residing on the same disk as your sensitive data. This can open you up to several types of attacks, including the 'Evil Maid' attack.
	- By keeping the most vulnerable part of your system encrypted and with you at all times - e.g. USB drive on a physical keychain. You can ensure that your laptop/pc is in a trustworthy state every time you use it. 
	
### DISLAIMER:

##### This setup only provides data-at-rest encryption. What does this mean? It means when your machine is booted up and running during normal use, it's as vulnerable as any other machine, unless otherwise configured.

##### Be smart and use the correct privacy tools to protect yourself during active use, VPNs, TOR, etc.
	
Cool you've convinced me this is worth doing, now what?

### REQUIREMENTS:

	1. Laptop or PC - the threat model that this is based on is more relevant for a laptop. You can generally control the physical security of a desktop, servers are rarely powered off, so again not as relevant.
	2. USB drive, I recommend using a new drive from a reliable brand, if this is lost, damaged, or stops working, the data on your internal drive is gone forever.
	3. A modern Linux distro - I used Manjaro to develop this.
	4. Some basic Linux knowledge - slide in my DMs if you have any questions.  
  
I'll try to break this guide up into the relevant sections so that it's not such a snoozefest.  

### I. PREPARATION:

##### Throughout this guide: 
	-> /dev/sda will refer to the INTERNAL drive  
	-> /dev/sdb will refer to the USB drive/key

Take care to ensure you are using the correct device names on your system. If you are unsure, you can use the output of `lsblk` to help you find out what your block device mappings are.

#### Part I: Booting install media
	
1. Boot into the live environment of your distro - make sure you are booted in U/EFI mode. In your BIOS/UEFI setup this will mean disabling SecureBoot and selecting U/EFI only.

2. Once at the GRUB menu for your distro, hit 'c' to enter the command-line. Enter the following: `echo $grub_platform` This should say `efi` if you're up in EFI mode.

3. Continue to boot as you normally would, then open a root shell and run the following commands found in the code blocks in the rest of this guide.

#### Part II: Preparing the storage devices

Overwrites each device with pseudo-random bits (in 4MB chunks) to ensure that nothing remains on either drive.

1. `dd if=/dev/urandom of=/dev/sda bs=4M`

2. `dd if=/dev/urandom of=/dev/sdb bs=4M`

#### Part III: Preparing the USB key

##### Creating the EFI partition on the USB key:  
1. `gdisk /dev/sdb`
   Press Enter after entering each of the following items:
   From the interactive menu, select `n`, `1`, `2048`, `+512M`, `EF00` - EF00 is the hex code that identifies the EFI partition.
	
##### Creating the boot partiion on the USB key:  
2. Press Enter after entering each of the following items:
   From the ineractive menu select `n`, `2`, Press Enter to accept default start value, `+250M`, `8300` 

##### Write the partitions to the partition table:  
3. Press `w` to write changes and then quit.

##### Encrypt the boot partition on the USB key - here you will input your passphrase, make it long and strong (like MC Hammer) and be sure to remember it!  

##### The '-i' option determines many ms it will take to decrypt your boot partition each time you boot. You can choose different ciphers/hash parameters if you want - just do your research!
4. `cryptsetup --hash=sha512 --cipher=twofish-xts-plain64 --key-size=512 -i 10000 luksFormat /dev/sdb2`

##### Open the encrypted boot partition:  
5. `cryptsetup open /dev/sdb2 cryptboot`

##### Create the filesystem on the boot partition - ext2 is a good choice because journaling can wear out your USB key:  
6. `mkfs.ext2 /dev/mapper/cryptboot`

##### Mount the boot partition:  
7. `mount /dev/mapper/cryptboot /mnt`

##### Create 20MB keyfile to encrypt the internal drive - this is a larger keyfile than we need, 8M is the max keyfile size for cryptsetup.  

##### This is okay since we'll be using an offset to navigate to the correct position on the large keyfile.  
8. `dd if=/dev/urandom of=/mnt/key.img bs=20M count=1`

##### Encrypt the keyfile - Option '-i' determines how many ms it will take to decrypt your keyfile each time you boot. You can choose different ciphers/hash parameters if you want - just do your research!  
9. `cryptsetup --align-payload=1 --hash=sha512 --cipher=serpent-xts-plain64 --key-size=512 -i 10000 luksFormat /mnt/key.img` 

##### Decrypt your keyfile and map it as a device.  
10. `cryptsetup open /mnt/key.img lukskey`

### PART IV: Preparing the internal/main drive

##### Create file to hold device header  
1. `truncate -s 2M /mnt/header.img`

##### Encrypt drive and put header data into header.img, --keyfile-offset can be a different number, I chose 2048. --cipher and --hash parameters can change if you want - just do your research!  
2. `cryptsetup --hash=sha512 --cipher=serpent-xts-plain64 --key-size=512 --key-file=/dev/mapper/lukskey --keyfile-offset=2048 --keyfile-size=8192 luksFormat /dev/sda --align-payload 4096 --header /mnt/header.img` 

##### Decrypt main drive so we can interact with it. If you chose a different number for --keyfile-offset previously, use the same value here, I'm naming the decrypted drive container 'main':  
3. `cryptsetup open --header /mnt/header.img --key-file=/dev/mapper/lukskey --keyfile-offset=2048 --keyfile-size=8192 /dev/sda main`

##### Unmap keyfile.img:  
4. `cryptsetup close lukskey`

##### Unmount /boot filesystem:  
5. `umount /mnt`

##### Create LVM physical volume on main/internal drive - we will use LVM to manage the partitions within the encrypted container of the main/internal drive:  
6. `pvcreate /dev/mapper/main`

##### Create LVM volume group titled 'root' on physical volume 'main':  
7. `vgcreate vg1 /dev/mapper/main`

##### Create our swap space, typically this is equal to the amount of RAM on your machine:  
8. `lvcreate -L 8G vg1 -n swap`

###### NOTE: The partitioning scheme within the volume group is flexible, if you want to split things up that works. I prefer to keep it consolidated to one 'root' partition for a laptop setup:  

##### Create root partition/LV, this will be where the rest of our installation lives - this command creates a logical volume named 'root' with the remaining space inside the volume group 'vg1':   
9. `lvcreate -l 100%FREE vg1 -n root` 

##### Create the filesystems and mount our new partitions/LVs:  
10. `mkfs.ext4 /dev/vg1/root`

##### Mount root partition/LV:  
11. `mount /dev/vg1/root /mnt`

##### Create our swap space:  
12. `mkswap /dev/vg1/swap`

##### Turn on our swap space:  
13. `swapon /dev/vg1/swap`

##### Create the boot directory for our new installation:  
14. `mkdir /mnt/boot`

##### Mount our encrypted boot partition on our new boot directory:  
15. `mount /dev/mapper/cryptboot /mnt/boot`

##### Create the filesystem for our EFI partition:  
16. `mkfs.fat -F32 /dev/sdb1`

##### Create the EFI directory in our boot directory:  
17. `mkdir /mnt/boot/efi`

##### Finally mount the efi partition at /boot/efi on our new installation:  
18. `mount /dev/sdb1 /mnt/boot/efi`

### PART V: Installation and creating custom kernel runtime hooks  

Here is where steps will diverge for different distributions. This guide was developed on Manjaro 20. 
I can't guarantee success elsewhere - but give it a shot and let me know.  
For Manjaro users, fire up Manjaro Architect and step through the initial install steps. 
At this point we've already mounted our partitions where the installer expects them, e.g `/mnt`.   
We can skip everything in 'Prepare Installation' up to and including `'Mount Partitions'`.  
From here we can proceed with the regular installation. 
Do your configuration/tweaking and finally chroot into your new install. **DO NOT REBOOT YET**.

Once 'chrooted' into your new install you might encounter some terminal weirdness, run these commands to resolve it:  

- `stty sane`  
- `export TERM=linux`

These will restore the normal keymapping in your chroot shell.  

At this point we will need the IDs of our devices, these can be found by running `ls -lth /dev/disk/by-id`. You will need the actual disk IDs later when we build our kernel runtime hooks.  

Install our encrypthook scripts - these will perform the mounting of the necessary partitions and some other housekeeping at boot time.  
1. Place the contents of the `encrypthook_part1` into `/etc/initcpio/hooks/encrypthook`  
	
	`wget https://raw.githubusercontent.com/ech0r/KryptTop/master/encrypthook_part1 -O /etc/initcpio/hooks/encrypthook`

#### IMPORTANT! Read the notes section of [encrypthook_part1](encrypthook_part1) and update the script at `/etc/initcpio/hooks/encrypthook` with your correct device IDs.

#### DO NOT CONTINUE until this step is complete.

2. Place the contents of the `encrypthook_part2` file into `/etc/initcpio/install/encrypthook`  

	`wget https://raw.githubusercontent.com/ech0r/KryptTop/master/encrypthook_part2 -O /etc/initcpio/install/encrypthook`
	
	
##### Update our `/etc/mkinitcpio.conf` file with the following:  
Don't replace your `HOOKS=()` array - but make sure the `encrypthook` hook is placed between `block` and `lvm2`. Additionally make sure hooks `encrypt`, `systemd`, and `sd-lvm2` are removed.
Also make sure this file contains `MODULES=(loop)`  When you are done `HOOKS=()` should look something like this:

	HOOKS=(base udev autodetect modconf block encrypthook lvm2 filesystems keyboard fsck)

### Part VI: Setting up UEFI with secureboot

For this section we will need two tools outside of the normal Manjaro repositories. These tools are `cryptboot` and `sbupdate`.  

`cryptboot` is a tool for generating and enrolling .efi keys on our machine - these allow secureboot to trust the .efi files at boot time.  

`sbupdate` is a tool for generating signed .efi files that contain the information UEFI needs to properly boot our machine.  

4. Download, build, and install cryptboot.  
  
 Once installed we will use cryptboot to create efi keys to sign our .efi files.  
###### NOTE: USERNAME is the account name of a normal user you created during installation.  

	su -l USERNAME
	curl -o cryptboot.tar.gz https://aur.archlinux.org/cgit/aur.git/snapshot/cryptboot.tar.gz
	tar -xvf cryptboot.tar.gz
	cd cryptboot
	makepkg -si
	exit

###### NOTE: the exit command should have bounced you back to your chroot shell.

Edit `/etc/crypttab` to contain `cryptboot /dev/disk/by-uuid/XXXX none luks` where 'XXXX' is the UUID is the encrypted boot partition of your usb key. I am looking for `sdb2` here.  

5. `echo "cryptboot /dev/disk/by-uuid/$(ls -lth /dev/disk/by-uuid | grep -iP 'sdb2' | awk '{print $9}') none luks" >>  /etc/crypttab`

Make sure `/etc/cryptboot.conf` contains the following lines:  

    BOOT_CRYPT_NAME="cryptboot"
    BOOT_DIR="/boot"
    EFI_DIR="/boot/efi"
    EFI_KEYS_DIR="/boot/efikeys"

Generate our EFI keys and enroll them

6. `cryptboot-efikeys create`
7. `cryptboot-efikeys enroll`

**IMPORTANT!** Go to `/boot/efikeys` and rename every file starting with 'db.' to 'DB.' e.g. `db.key` -> `DB.key`

Delete `/etc/crypttab` entry - we only needed this to generate our EFI keys  

8. `sed -i '$ d' /etc/crypttab`

9. Download, build, and install sbupdate  
 
###### NOTE: USERNAME is the account name of a normal user you created during installation.  
	su -l USERNAME
	curl -o sbupdate-git.tar.gz https://aur.archlinux.org/cgit/aur.git/snapshot/sbupdate-git.tar.gz
	tar -xvf sbupdate-git.tar.gz
	cd sbupdate-git
	makepkg -si
	exit

10. Edit `/etc/default/sbupdate` to contain the following lines - where KERNEL is the compressed kernel image you want to boot, in my case it was `vmlinuz-5.4-x86_64`.  

###### NOTE: vg1-root is the volume group we created earlier.  
	KEY_DIR="/boot/efikeys"  
	ESP_DIR="/boot/efi"
	OUT_DIR="/EFI/Manjaro"
	CMDLINE_DEFAULT="/KERNEL root=/dev/mapper/vg1-root rw quiet"  



Generate initframfs image and run sbupdate - where PRESET is your initramfs preset, the preset I used at the time of this writing was `linux54`.  

11. `mkinitcpio -p PRESET && sbupdate`  

###### NOTE: the equivalent of `mkinitcpio` on Debian based distros would be `initramfs`

###### NOTE: You'll see an error about a missing splash image - ignore this.

Create a boot option for the signed EFI file.  Where `EFI_FILE` is the filename created by sbupdate. Mine was `5.4-x86_64-signed.efi`.

12. `efibootmgr -c -d /dev/sdb1 -p 1 -L "Manjaro Linux" -l "EFI\Manjaro\EFI_FILE"`
###### NOTE: "Manjaro Linux" can be any label you want, this is what you would see at the UEFI boot menu when starting your machine.

13. close your chroot shell, `umount -R /mnt` and reboot! Remove your install media and you should be greeted with a passphrase prompt for both your boot partition and your keyfile.  

14. Enjoy your newly encrypted machine!



  







	



	
	
