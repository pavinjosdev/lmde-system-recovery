# LMDE system recovery using btrfs and snapshots - from broken OS or hardware failure to a fully functional system.

This guide is written for [Linux Mint Debian Edition](https://linuxmint.com/download_lmde.php) but can be easily adapted for any Debian/Ubuntu based system, and less easily for any other Linux based system. The concepts are the same irrespective of distribution.

A similar guide written for [EndeavourOS](https://endeavouros.com/) can be found [here](https://github.com/pavinjosdev/eos-system-recovery/).

---

# Goals

- Encrypted root and swap (for hibernation) partitions.
- Fast incremental snapshots of whole system that can be restored easily in the event of a broken OS.
- Offsite backup of snapshots that can be restored easily in the event of hardware failure.

> This setup should not be used in production, it was originally intended for a personal laptop without RAID and that means without the self healing feature offered by the Btrfs filesystem.

---

# Tools / technologies

- [Btrfs](https://btrfs.wiki.kernel.org/index.php/Main_Page) ("Butter" FS): CoW (copy-on-write) filesystem with super fast snapshots
- [snapper](http://snapper.io/): automatic creation & management of btrfs snapshots
- [snap-apt](https://github.com/pavinjosdev/snap-apt): automatic creation of pre/post btrfs snapshots on package changes when using APT (Debian package manager)
- [grub-btrfs](https://github.com/Antynea/grub-btrfs): include btrfs snapshots in grub (bootloader) menu options
- [borg](https://borgbackup.readthedocs.io): backup program that supports deduplication, compression, and client side encryption - used for offsite backup and restore
- [borgmatic](https://torsion.org/borgmatic/): automatic creation & management of borg backups

---

# LMDE installation

> For ease of setup, this guide assumes you're `root`. Be careful and read twice before executing any commands!

This is optional, but provides the same disk/partition layout used in this article.

## Automated install

1. Boot into LMDE live installer
2. Patch files that haven't been upstreamed yet (open terminal using Ctrl+Alt+T)

```
apt update
apt install git
git clone https://github.com/pavinjosdev/lmde-live-installer.git
cp -a lmde-live-installer/usr/* /usr
```

3. Run the installer in normal mode from the desktop icon or by running the command `/usr/bin/live-installer`

4. In the "Installation type" window, choose "Automated installation" and check the options:

- Use LVM (Logical Volume Management)
- Encrypt the operating system
- Format root partition using btrfs

## Manual install

- Boot into LMDE live installer

- Fire up the terminal using Ctrl+Alt+T.

```
# Setup partitions on physical disk
fdisk /dev/nvme0n1
# Partition 1 of size 286M and type EFI system
# Partition 2 of size 944M and type Linux filesystem
# Partition 3 of size (rest of disk space) and type Linux filesystem

# Format partition 1 as vfat for /boot/efi (ESP)
mkfs.vfat -F32 /dev/nvme0n1p1

# Format partition 2 as ext4 for /boot
mkfs.ext4 /dev/nvme0n1p2

# Setup LUKS encryption on partition 3
cryptsetup luksFormat /dev/nvme0n1p3
cryptsetup open /dev/nvme0n1p3 lvmlmde
cryptsetup status lvmlmde
ls /dev/mapper

# Setup logical volumes (LVs) to hold root and swap partitions
pvcreate /dev/mapper/lvmlmde
pvs
vgcreate lvmlmde /dev/mapper/lvmlmde
vgs
lvcreate -L 6G -n swap lvmlmde
lvcreate -l 100%FREE -n root lvmlmde
lvs
ls /dev/mapper
ls /dev/lvmlmde
lsblk -f /dev/nvme0n1

# Format new LVs
mkswap /dev/lvmlmde/swap
mkfs.btrfs /dev/lvmlmde/root

# Create btrfs subvolumes
mount /dev/lvmlmde/root /mnt
cd /mnt
btrfs sub cr @
btrfs sub cr @home
btrfs sub cr @var-log
btrfs sub cr @var-cache
btrfs sub list /mnt
cd ~
umount /mnt
```

- Start LMDE GUI installer in expert mode with the command `/usr/bin/live-installer-expert-mode` and proceed with installation.

- In the "Installation type" window, choose "Manual Partitioning".

- In the "Partitioning" window, choose "Expert mode".

- Prepare `/target` mountpoint:

```
mkdir /target
mount -o subvol=@,defaults,noatime,compress=zstd,discard=async /dev/lvmlmde/root /target

mkdir -p /target/home
mkdir -p /target/var/log
mkdir -p /target/var/cache
mount -o subvol=@home,defaults,noatime,compress=zstd,discard=async /dev/lvmlmde/root /target/home
mount -o subvol=@var-log,defaults,noatime,compress=zstd,discard=async /dev/lvmlmde/root /target/var/log
mount -o subvol=@var-cache,defaults,noatime,compress=zstd,discard=async /dev/lvmlmde/root /target/var/cache

mkdir -p /target/boot
mount /dev/nvme0n1p2 /target/boot

mkdir -p /target/boot/efi
mount /dev/nvme0n1p1 /target/boot/efi
```

- Proceed with installation.

- Chroot into new system:

```
for i in /dev /dev/pts /proc /run /sys; do mount --bind $i /target$i; done
chroot /target
```

- Setup fstab, crypttab, and a few other things:

```
root@mint:~# lsblk -f /dev/nvme0n1
NAME               FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINT
nvme0n1                                                                                             
├─nvme0n1p1        vfat        FAT32          4429-E99B                                 282M     1% 
├─nvme0n1p2        ext4        1.0            c2f96f90-db58-4700-abbd-7aff8dd07929    737.5M    15% 
└─nvme0n1p3        crypto_LUKS 2              a911c0df-4acf-4af4-aba1-376af138db8f                  
  └─lvmlmde        LVM2_member LVM2 001       9tglX3-L9FP-ASNE-nibs-T2Ev-Id7e-im2TZh                
    ├─lvmlmde-root btrfs                      992e4b8c-7de1-4a94-a2cc-079b69bbadd7      224G     2% 
    └─lvmlmde-swap swap        1              d8294c05-4570-4a78-9eaa-382c05d7040d                  
```

File: `/etc/fstab`

Contents:

```
# Static filesystem information
tmpfs  /tmp  tmpfs  defaults,noatime  0  0
# /dev/mapper/lvmlmde-root
UUID=992e4b8c-7de1-4a94-a2cc-079b69bbadd7  /  btrfs  subvol=@,defaults,noatime,compress=zstd,discard=async  0  1
UUID=992e4b8c-7de1-4a94-a2cc-079b69bbadd7  /home  btrfs  subvol=@home,defaults,noatime,compress=zstd,discard=async  0  2
UUID=992e4b8c-7de1-4a94-a2cc-079b69bbadd7  /var/log  btrfs  subvol=@var-log,defaults,noatime,compress=zstd,discard=async  0  2
UUID=992e4b8c-7de1-4a94-a2cc-079b69bbadd7  /var/cache  btrfs  subvol=@var-cache,defaults,noatime,compress=zstd,discard=async  0  2
# /dev/mapper/lvmlmde-swap
UUID=d8294c05-4570-4a78-9eaa-382c05d7040d none   swap sw 0 0
# /dev/nvme0n1p2
UUID=c2f96f90-db58-4700-abbd-7aff8dd07929 /boot  ext4 defaults 0 1
# /dev/nvme0n1p1
UUID=4429-E99B /boot/efi  vfat defaults 0 1
```

File: `/etc/crypttab`

Contents:

```
# <target name>	<source device>		<key file>	<options>
lvmlmde   UUID=a911c0df-4acf-4af4-aba1-376af138db8f   none   luks,discard,tries=3
```

Modify parameter in `/etc/default/grub`

```
GRUB_CMDLINE_LINUX="cryptdevice=UUID=a911c0df-4acf-4af4-aba1-376af138db8f:lvmlmde root=/dev/mapper/lvmlmde-root resume=/dev/mapper/lvmlmde-swap"
```

Add the following modules to `/etc/initramfs-tools/modules`

```
aes-i586
aes_x86_64
dm-crypt
dm-mod
xts
```

- Proceed to finish installation.

---

# Disk topology

Disk and partitions:

```
root@mint-laptop:~# lsblk -f
NAME               FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINT
nvme0n1                                                                                             
├─nvme0n1p1        vfat        FAT32          4429-E99B                                 282M     1% /boot/efi
├─nvme0n1p2        ext4        1.0            c2f96f90-db58-4700-abbd-7aff8dd07929    737.5M    15% /boot
└─nvme0n1p3        crypto_LUKS 2              a911c0df-4acf-4af4-aba1-376af138db8f                  
  └─lvmlmde        LVM2_member LVM2 001       9tglX3-L9FP-ASNE-nibs-T2Ev-Id7e-im2TZh                
    ├─lvmlmde-root btrfs                      992e4b8c-7de1-4a94-a2cc-079b69bbadd7      224G     2% /home/.snapshots
    └─lvmlmde-swap swap        1              d8294c05-4570-4a78-9eaa-382c05d7040d                  [SWAP]
```

It's not required to have the exact aforementioned layout, but knowing the disk/partition topology would make following this guide easier.

We're naturally interested in the btrfs partition: `/dev/lvmlmde/root`. Ignore the mountpoint for it as `lsblk` is showing the last mount configured in fstab, btrfs can mount each "subvolume" to different mountpoints. This is covered in the next section.

Note that since we've encrypted root and swap partitions, the entire physical partition `/dev/nvme0n1p3` is encrypted using [LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup).
The root and swap partitions residing in it are [logical volumes](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)).

---

# Btrfs

Btrfs ("Butter" FS) is a modern CoW (copy-on-write) filesystem with a lot of interesting features like checksumming for both data and metadata that can be used for finding and correcting errors.

Since we're not using RAID (with mirroring), our setup can only find errors and not fix it automatically. We can always restore from the offsite backups if the original ever gets corrupted beyond repair.

For the purpose of this guide it's important to understand Btrfs _subvolume_ and _CoW_ concepts.

![Btrfs logo](https://i.imgur.com/muX701g.png)

## Subvolumes

- A Btrfs subvolume is an independently mountable POSIX filetree and not a block device (and cannot be treated as one). Most other POSIX filesystems have a single mountable root, Btrfs has an independent mountable root for the volume (top level subvolume) and for each subvolume; a Btrfs volume can contain more than a single filetree, it can contain a forest of filetrees. A Btrfs subvolume can be thought of as a POSIX file namespace.
- The top-level subvolume (with Btrfs id 5) (which one can think of as the root of the volume) can be mounted, and the full filesystem structure will be seen at the mount point; alternatively any other subvolume can be mounted (with the mount options subvol or subvolid, for example subvol=@home) and only anything below that subvolume will be visible at the mount point. For example when we map the subvolume `@home` to `/home` directory, only the contents of subvolume `@home` will be visible at the mount point `/home`.

Here's the filesystem layout we'll be implementing:

```
<subvol>            <mountpoint>
@                   /
@home               /home
@var-log            /var/log
@var-cache          /var/cache
@snapshots          /.snapshots
@home-snapshots     /home/.snapshots
```

Listing current subvolumes:

```
root@mint-laptop:~# btrfs subvolume list /
ID 521 gen 17840 top level 5 path @
ID 257 gen 17841 top level 5 path @home
ID 258 gen 17841 top level 5 path @var-log
ID 259 gen 17635 top level 5 path @var-cache
```

Manually mount the btrfs partition to see what's inside:

```
# Mount btrfs partition at /mnt
root@mint-laptop:~# mount /dev/lvmlmde/root /mnt

root@mint-laptop:~# cd /mnt
root@mint-laptop:/mnt# ls
@  @home  @var-cache  @var-log
root@mint-laptop:/mnt# ls @
bin   dev  home        initrd.img.old  lib32  libx32  mnt  proc  run   srv  tmp  var	  vmlinuz.old
boot  etc  initrd.img  lib	       lib64  media   opt  root  sbin  sys  usr  vmlinuz
root@mint-laptop:/mnt# ls @home
pavin
root@mint-laptop:/mnt# ls @var-cache
apparmor  apt	    cups     dictionaries-common  fwupd     ldconfig  man	  pm-utils  samba
app-info  cracklib  debconf  fontconfig		  fwupdmgr  lightdm   PackageKit  private
root@mint-laptop:/mnt# ls @var-log
alternatives.log  cups						   faillog	   messages		  snap-apt.log	     Xorg.0.log
apt		  daemon.log					   fontconfig.log  mintsystem.log	  snapper.log	     Xorg.0.log.old
aptitude	  debian-system-adjustments-adjust-grub-title.log  hp		   mintsystem.timestamps  speech-dispatcher
auth.log	  debian-system-adjustments-start.log		   journal	   openvpn		  syslog
boot.log	  debian-system-adjustments-stop.log		   kern.log	   private		  ufw.log
bootstrap.log	  debug						   lastlog	   runit		  user.log
btmp		  dpkg.log					   lightdm	   samba		  wtmp

# Unmount
root@mint-laptop:~# umount /mnt
```

[Source](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Subvolumes)

## Copy on Write (CoW)

- The CoW operation is used on all writes to the filesystem.
- This makes it much easier to implement lazy copies, where the copy is initially just a reference to the original, but as the copy (or the original) is changed, the two versions diverge from each other in the expected way.

This is why snapshots can be created so fast and easily in btrfs. A snapshot subvolume is simply a reference to another subvolume.

[Source](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Copy_on_Write_.28CoW.29)

---

# Installing packages

Update repos and upgrade packages:

```
apt update && apt upgrade
```

Install snapper, borg, and borgmatic:

```
apt install snapper borgbackup borgmatic
```

Install grub-btrfs:

```
apt install git inotify-tools
git clone https://github.com/Antynea/grub-btrfs.git
cd grub-btrfs
make install
```

Install snap-apt:

```
git clone https://github.com/pavinjosdev/snap-apt.git
chmod 755 snap-apt/scripts/snap_apt.py
cp snap-apt/scripts/snap_apt.py /usr/bin/snap-apt
cp snap-apt/hooks/80snap-apt /etc/apt/apt.conf.d/
cp snap-apt/logrotate/snap-apt /etc/logrotate.d/
rm -f /etc/apt/apt.conf.d/80snapper
sed -i 's/DISABLE_APT_SNAPSHOT=\"no\"/DISABLE_APT_SNAPSHOT=\"yes\"/g' /etc/default/snapper
```

---

# Configure snapper

Create a snapper config for root subvolume mounted at `/`

```
snapper -c root create-config /
```

You may create other snapper configs as needed, for example for `/home` like so:

```
snapper -c home create-config /home
```

List snapper configs:

```
snapper list-configs
```

Modify the snapper configs, the defaults are probably fine.

```
nano /etc/snapper/configs/root
nano /etc/snapper/configs/home
```

Backup non-btrfs partition `/boot` on kernel/initramfs updates

```
mkdir -p /etc/initramfs/post-update.d
```

File: `/etc/initramfs/post-update.d/backup_boot`

Contents:

```
#!/usr/bin/env bash
/usr/bin/rsync -a --delete /boot/ /.bootbackup
```

Mark the file as executable:

```
chmod 755 /etc/initramfs/post-update.d/backup_boot
```

> The filename must contain only ASCII uppercase & lowercase letters, digits, underscore, and hyphen. See `man run-parts`.

---

# Configure btrfs snapshots

Snapper automatically creates a `.snapshots` directory inside the directory it's snapshotting.
For the above config for root it will be at `/.snapshots`.
But we don't need it as we'll create an empty directory of same name/path for our `@snapshots` subvolume to be mounted at.

Delete the `.snapshots` directory snapper created at `/` and at `/home`

```
rm -rI /.snapshots
rm -rI /home/.snapshots
```

Create the directories anew

```
mkdir /.snapshots
mkdir /home/.snapshots
```

Mount btrfs partition and create new subvolumes

```
mount -t btrfs /dev/lvmlmde/root /mnt
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@home-snapshots
btrfs subvolume list /mnt
umount /mnt
```

Automatically mount the snapshots subvolumes:

File: `/etc/fstab`

Contents:

```
# Static filesystem information
tmpfs  /tmp  tmpfs  defaults,noatime  0  0
# /dev/mapper/lvmlmde-root
UUID=992e4b8c-7de1-4a94-a2cc-079b69bbadd7  /  btrfs  subvol=@,defaults,noatime,compress=zstd,discard=async  0  1
UUID=992e4b8c-7de1-4a94-a2cc-079b69bbadd7  /home  btrfs  subvol=@home,defaults,noatime,compress=zstd,discard=async  0  2
UUID=992e4b8c-7de1-4a94-a2cc-079b69bbadd7  /var/log  btrfs  subvol=@var-log,defaults,noatime,compress=zstd,discard=async  0  2
UUID=992e4b8c-7de1-4a94-a2cc-079b69bbadd7  /var/cache  btrfs  subvol=@var-cache,defaults,noatime,compress=zstd,discard=async  0  2
# /dev/mapper/lvmlmde-swap
UUID=d8294c05-4570-4a78-9eaa-382c05d7040d none   swap sw 0 0
# /dev/nvme0n1p2
UUID=c2f96f90-db58-4700-abbd-7aff8dd07929 /boot  ext4 defaults 0 1
# /dev/nvme0n1p1
UUID=4429-E99B /boot/efi  vfat defaults 0 1

# Snapshots
UUID=992e4b8c-7de1-4a94-a2cc-079b69bbadd7  /.snapshots  btrfs  subvol=@snapshots,defaults,noatime,compress=zstd,discard=async  0  0
UUID=992e4b8c-7de1-4a94-a2cc-079b69bbadd7  /home/.snapshots  btrfs  subvol=@home-snapshots,defaults,noatime,compress=zstd,discard=async  0  0
```

Mount all partitions listed in `/etc/fstab`:

```
mount -av
```

---

# Create a manual snapshot

Check if everything works by creating a manual snapshot:

```
snapper -c root create -c number -d 'Test snap'
```

Flags (options):

- The first `-c` flag is for specifying the config name, which is `root`. To list all snapper configs: `snapper list-configs`
- The second `-c` flag is for specifying the cleanup algorithm to use, which is `number`. Snapshots older than the number limit specified in `/etc/snapper/configs/root` would be pruned
- The `-d` flag specifies a description for the snapshot

List the snapshots:

```
snapper -c root list
```

List all subvolumes:

```
root@mint-laptop:~# btrfs subvolume list /
ID 521 gen 17875 top level 5 path @
ID 257 gen 17875 top level 5 path @home
ID 258 gen 17874 top level 5 path @var-log
ID 259 gen 17851 top level 5 path @var-cache
ID 268 gen 17765 top level 5 path @snapshots
ID 269 gen 17763 top level 5 path @home-snapshots
ID 291 gen 349 top level 290 path @snapshots/1/snapshot
```

> We can see the snapshot we just created manually.

Get property of subvolume:

```
root@mint-laptop:~# btrfs property get /.snapshots/1/snapshot
ro=true

root@mint-laptop:~# btrfs property get /
ro=false
```

> By default, snapper creates read-only snapshots. This is preferred as it would prevent us from accidentally modifying any snapshots, one of which may be our last chance at restoring to a functional system state.
> To restore from a snapshot, we create a read-write snapshot from the read-only snapshot (shown later).

---

# Update grub menu with snapshot entries

Generate grub configuration manually once:

```
update-grub
/etc/grub.d/41_snapshots-btrfs
```

To automatically update grub menu entries on detecting changes in `/.snapshots` directory, the `grub-btrfs` package provides a systemd script:

```
systemctl enable grub-btrfsd
systemctl restart grub-btrfsd
systemctl status grub-btrfsd
```

---

# Restoring from a snapshot

Power on or reboot the system.

From the snapshot list, boot into the last functional snapshot. Make a mental note of the snapshot number.

> The system should boot without issues as `/tmp`, `/var/log`,  `/var/cache`, `/home` are all read-write, only `/` will be read-only.
> If the system does not boot from the snapshot, boot from a live image.

Make sure we have booted into a read-only snapshot:

```
root@mint-laptop:~# btrfs property get /
ro=true
```

> The aforementioned step is crucial, do not proceed if it's _not_ a read-only snapshot.

Optional: If you don't know the correct snapshot number to restore, read the `info.xml` file associated with each snapshot created by snapper:

```
nano /.snapshots/*/info.xml
```

As an example, we will proceed with restoration of root `/` (subvolume `@`) from snapshot number `33`

Mount the btrfs partition to `/mnt`:

```
mount -t btrfs -o defaults,noatime,compress=zstd,discard=async /dev/lvmlmde/root /mnt
```

Rename the `@` subvolume:

```
mv /mnt/@ /mnt/@.broken
```

Create a new `@` subvolume from our chosen snapshot:

```
btrfs subvolume snapshot /.snapshots/33/snapshot /mnt/@
```

> By default, btrfs creates read-write snapshot. To create a read-only snapshot, specify the `-r` flag.

Unmount and reboot into the restored system:

```
umount /mnt
reboot
```

Once rebooted, delete the `@.broken` subvolume:

```
mount -t btrfs -o defaults,noatime,compress=zstd,discard=async /dev/lvmlmde/root /mnt
btrfs subvolume delete /mnt/@.broken
umount /mnt
```

Optionally, scrub the filesystem (verifies all data & metadata checksums):

```
btrfs scrub start /
btrfs scrub status /
```

---

# Check and repair btrfs filesystem

Boot from a live image

List all block devices:

```
lsblk -f
```

Check the unmounted btrfs filesystem:

```
btrfs check /dev/lvmlmde/root
```

Repairing an unmounted btrfs filesystem (DANGEROUS OPTION, see man page for `btrfs-check`):

```
btrfs check --repair /dev/lvmlmde/root
```

---

# Setup offsite backups

Install borg and create user on *remote server* with SSH access (in this case it's a Debian server):

```
apt install borgbackup
adduser borg --disabled-password
```

From local machine:

```
mkdir -p /root/.ssh
cd /root/.ssh
ssh-keygen -t rsa
```

Copy the public key to *remote server* location: `/home/borg/.ssh/authorized_keys`.

From local machine:

```
ssh borg@remote-server.tld
mkdir mint-laptop
```

> In the previous command, `mint-laptop` refers to local machine's hostname and `remote-server.tld` refers to remote server's FQDN

Initialize borg repository from local machine:

```
borg init -e repokey borg@remote-server.tld:/home/borg/mint-laptop
```

> Make sure to securely backup the borg repo password as well as the private key created earlier.

Generate borgmatic config from local machine:

```
mkdir -p /etc/borgmatic
generate-borgmatic-config -d /etc/borgmatic/config.yaml
chmod 600 /etc/borgmatic/config.yaml
chown root:root /etc/borgmatic/config.yaml
```

Edit the borgmatic config file `/etc/borgmatic/config.yaml`:

```
location:
    source_directories:
        - /mnt/@.latest
        - /mnt/@home.latest
        - /mnt/@var-log.latest
        - /mnt/@var-cache.latest

    repositories:
        - borg@remote-server.tld:/home/borg/mint-laptop

storage:
    encryption_passphrase: "<your-borg-repo-password>"

retention:
    keep_daily: 7
    keep_weekly: 4

hooks:
    before_backup:
        - mount /dev/lvmlmde/root /mnt
        - btrfs subvolume snapshot -r /mnt/@ /mnt/@.latest
        - btrfs subvolume snapshot -r /mnt/@home /mnt/@home.latest
        - btrfs subvolume snapshot -r /mnt/@var-log /mnt/@var-log.latest
        - btrfs subvolume snapshot -r /mnt/@var-cache /mnt/@var-cache.latest

    after_backup:
        - btrfs subvolume delete /mnt/@.latest
        - btrfs subvolume delete /mnt/@home.latest
        - btrfs subvolume delete /mnt/@var-log.latest
        - btrfs subvolume delete /mnt/@var-cache.latest
        - umount /mnt

    on_error:
        - btrfs subvolume delete /mnt/@.latest
        - btrfs subvolume delete /mnt/@home.latest
        - btrfs subvolume delete /mnt/@var-log.latest
        - btrfs subvolume delete /mnt/@var-cache.latest
        - umount /mnt
```

Validate the borgmatic config file `/etc/borgmatic/config.yaml`:

```
validate-borgmatic-config
```

Perform a backup (this may take a while as it's a first time sync):

```
borgmatic --verbosity 1
```

Setup cron to automatically perform backups.

File: `/etc/cron.d/borgmatic`

Contents:

```
0 3 * * * root /usr/bin/borgmatic --verbosity 1 --syslog-verbosity 1
```

Secure cron file:

```
chmod 600 /etc/cron.d/borgmatic
chown root:root /etc/cron.d/borgmatic
```

List all backups:

```
borgmatic list
```

Get information about backups:

```
borgmatic info
```

Break lock if previous backup was interrupted and no borg processes are running:

```
borgmatic borg break-lock
```

---

# Restore from offsite backups

This section details restoring to a fresh installation of the OS.

Boot into freshly installed system and install borg and borgmatic as stated in the previous section.

List all backups:

```
borgmatic list
```

Mount the latest backup to check files:

```
borgmatic mount --archive latest --mount-point /mnt
```

Unmount:

```
borgmatic umount --mount-point /mnt
```

Extract files to `/root`:

```
borgmatic extract --archive latest --destination /root --progress
```

Rename backup:

```
cd /root
mv mnt backup
```

Reboot into live environment

Mount btrfs partition:

```
cryptsetup open /dev/nvme0n1p3 lvmlmde
mount -t btrfs -o defaults,noatime,compress=zstd,discard=async /dev/lvmlmde/root /mnt
```

Rename existing subvolumes:

```
cd /mnt
mv @ @.fresh
mv @home @home.fresh
mv @var-log @var-log.fresh
mv @var-cache @var-cache.fresh
```

Move backup data into its subvolumes:

```
mv /mnt/@.fresh/root/backup/* /mnt
btrfs subvolume create @
btrfs subvolume create @home
btrfs subvolume create @var-log
btrfs subvolume create @var-cache
btrfs subvolume create @snapshots
btrfs subvolume create @home-snapshots
rsync -aH @.latest/ @
rsync -aH @home.latest/ @home
rsync -aH @var-log.latest/ @var-log
rsync -aH @var-cache.latest/ @var-cache
rm -rI @.latest @home.latest @var-log.latest @var-cache.latest
```

- Edit `/mnt/@/etc/fstab` with UUID of new boot, EFI, btrfs and swap partitions
- Edit `/mnt/@/etc/crypttab` with UUID of new LUKS partition
- Edit `/mnt/@/etc/default/grub` or `/mnt/@/etc/default/grub.d/61_live-installer.cfg` with UUID of new LUKS partition

Update kernel (optional, recommended):

```
cd ~
umount /mnt

mkdir /target
mount -o subvol=@,defaults,noatime,compress=zstd,discard=async /dev/lvmlmde/root /target

mkdir -p /target/home
mkdir -p /target/var/log
mkdir -p /target/var/cache
mount -o subvol=@home,defaults,noatime,compress=zstd,discard=async /dev/lvmlmde/root /target/home
mount -o subvol=@var-log,defaults,noatime,compress=zstd,discard=async /dev/lvmlmde/root /target/var/log
mount -o subvol=@var-cache,defaults,noatime,compress=zstd,discard=async /dev/lvmlmde/root /target/var/cache

mkdir -p /target/boot
mount /dev/nvme0n1p2 /target/boot

mkdir -p /target/boot/efi
mount /dev/nvme0n1p1 /target/boot/efi

for i in /dev /dev/pts /proc /run /sys; do mount --bind $i /target$i; done
chroot /target

mount -av

rsync -a /boot/ /.boot.fresh
# if using resolvconf for DNS resolution
echo 'nameserver 8.8.8.8' > /run/resolvconf/resolv.conf

apt update
apt purge linux-image-* linux-headers-*
apt install linux-image-amd64 linux-headers-amd64
update-grub
update-initramfs -c -k all
```

Reboot back into restored OS.

Optional: Delete unneeded btrfs subvolumes.

---

Here's a cute picture of *Butters* from *South Park.*

![Butters from South Park looks pleased at hearing about btrfs](https://i.imgur.com/UvG9wCq.png)

<!-- THE END -->
