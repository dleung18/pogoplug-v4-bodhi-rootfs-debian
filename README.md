# pogoplug-v4-bodhi-rootfs-debian
bodhi's Debian rootfs, prepared specifically for Pogoplug V4 devices (Pogoplug Mobile and Pogoplug Series 4).

* [Debian-jessie-3.18.5-pogoplug-v4-20151110-rootfs.tar.gz](https://github.com/pepaslabs/pogoplug-v4-bodhi-rootfs-debian/releases/download/jessie-3.18.5-20151110/Debian-jessie-3.18.5-pogoplug-v4-20151110-rootfs.tar.gz)

Also packaged as a raw disk image (see below instructions about growing the partition after you write the image to an SD card):

* [Debian-jessie-3.18.5-pogoplug-v4-20151110-disk-image.dd.gz](https://github.com/pepaslabs/pogoplug-v4-bodhi-rootfs-debian/releases/download/jessie-3.18.5-20151110/Debian-jessie-3.18.5-pogoplug-v4-20151110-disk-image.dd.gz)

If you can't grow the partition after writing the SD card (e.g. Windows and Mac users), here's the above packaged as a 4GB raw disk image:

* [Debian-jessie-3.18.5-pogoplug-v4-20151110-disk-image.4GB.img.zip](https://github.com/pepaslabs/pogoplug-v4-bodhi-rootfs-debian/releases/download/jessie-3.18.5-20151110/Debian-jessie-3.18.5-pogoplug-v4-20151110-disk-image.4GB.img.zip)

* [md5sums](https://github.com/pepaslabs/pogoplug-v4-bodhi-rootfs-debian/releases/download/jessie-3.18.5-20151110/md5sums)

## Rationale

User 'bodhi' at http://forum.doozan.com has prepared a Debian rootfs tarball for use with many small ARM machines (see this thread: http://forum.doozan.com/read.php?2,12096).

However, his release still requires a few additional steps in order to prepare it for the Pogoplug.

This github repository aims to provide versions of his rootfs (and disk images) which are fully ready to be used with Pogoplug V4 devices (Pogoplug Mobile and Pogoplug Series 4).

## Linux users: Using the disk image

* [Debian-jessie-3.18.5-pogoplug-v4-20151110-disk-image.dd.gz](https://github.com/pepaslabs/pogoplug-v4-bodhi-rootfs-debian/releases/download/jessie-3.18.5-20151110/Debian-jessie-3.18.5-pogoplug-v4-20151110-disk-image.dd.gz)

Write the disk image to the SD card at `/dev/sdf`:

```
dev=sdf
cat ~/Debian-jessie-3.18.5-pogoplug-v4-*-disk-image.dd.gz | gunzip > /dev/${dev}
```

Grow the SD card partition to take up the entire disk:

```
sfdisk /dev/${dev} << 'EOF'
,,,*
EOF
```

Grow the filesystem:

```
e2fsck -f /dev/${dev}1
resize2fs /dev/${dev}1
sync
```

Now stick it in a pogoplug and boot it!

`root`'s password is set to the empty string.  `ssh` shouldn't prompt you for a password.

**Don't forget to set a root password after logging in!**

```
ssh root@192.168.X.XXX
passwd
```

**Note**: it may take a minute or two for `ssh` to start (it has to regenerate a new set of host keys).

## Windows and Mac users: Using the disk image

For those without access to `sfdisk` and `resize2fs`, I have also provided a 4GB disk image packaged in a `.zip` file.

* [Debian-jessie-3.18.5-pogoplug-v4-20151110-disk-image.4GB.img.zip](https://github.com/pepaslabs/pogoplug-v4-bodhi-rootfs-debian/releases/download/jessie-3.18.5-20151110/Debian-jessie-3.18.5-pogoplug-v4-20151110-disk-image.4GB.img.zip)

See this page for some tips on writing the disk image to an SD card on Windows or Mac: http://elinux.org/RPi_Easy_SD_Card_Setup

## Producedure I Used to Modify bodhi's rootfs

I performed the following steps (on a machine running Debian `jessie`) to prepare the rootfs and disk images.

All of these commands should be run as **root**.

Set the following env vars (adjust as needed for your situation):

```
# SD card device:
dev=sdf

# temporary mount point:
mnt=/mnt/tmp
```

### Install the rootfs onto an SD card

Partition and format the SD card:

```
sfdisk /dev/${dev} << 'EOF'
,,,*
EOF
```

```
mke2fs -j -L rootfs /dev/${dev}1
```

Mount the SD card:

```
mount /dev/${dev}1 ${mnt}
```

Download bodhi's tarball (if you haven't already):

```
if [ ! -e ~/Debian-3.18.5-kirkwood-tld-1-rootfs-bodhi.tar.bz2 ]
then
    wget -O ~/Debian-3.18.5-kirkwood-tld-1-rootfs-bodhi.tar.bz2 https://bitly.com/1ELvRaw
    md5sum -c - << EOF
b5057448e7e08c747793f205e7027395  ${HOME}/Debian-3.18.5-kirkwood-tld-1-rootfs-bodhi.tar.bz2
EOF
fi
```

Unpack the tarball:

```
cd ${mnt}
cat ~/Debian-3.18.5-kirkwood-tld-1-rootfs-bodhi.tar.bz2 | bunzip2 | tar x
```

Create a basic `${mnt}/etc/fstab` which uses `ext3` as the rootfs filesystem:

```
cat > ${mnt}/etc/fstab << 'EOF'
# /etc/fstab: static file system information.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/root       /               ext3    noatime,errors=remount-ro 0 1
EOF
```

Setup the kernel image for the bootloader:

```
cd ${mnt}/boot
cp -a zImage-3.18.5-kirkwood-tld-1  zImage.fdt
cat dts/kirkwood-pogoplug_v4.dtb  >> zImage.fdt
mv uImage uImage.orig
mkimage -A arm -O linux -T kernel -C none -a 0x00008000 -e 0x00008000 -n Linux-3.18.5-kirkwood-tld-1 -d zImage.fdt uImage
```

Unmount the SD card:

```
cd
umount ${mnt}
sync
```

### Preparing the rootfs for distribution

Insert the SD card into a Pogoplug, **boot it**, and ssh into it:

```
ssh root@192.168.X.XXX
```

(The root password for bodhi's rootfs is **root**).

Comment out the `deb-src` entry in `/etc/sources.list` (speeds up apt a bit):

```
sed -i'' 's/^deb-src/#deb-src/' /etc/apt/sources.list
```

Install the latest security updates:

```
apt-get update
apt-get dist-upgrade
```

Remove unneeded packages:

```
apt-get autoremove
```

Remove cached packages:

```
apt-get clean
```

Restore root's `~/.profile`:

```
cp -a /usr/share/base-files/profile /root/.profile
```

Clear out `/etc/udev/rules.d/70-persistent-net.rules`:

```
echo -n > /etc/udev/rules.d/70-persistent-net.rules
```

Set root's password to be the empty string.  Edit `/etc/shadow` so that root's password hash is `s8OebdUUcHNvg`.

```
nano /etc/shadow
```

`root`'s entry should look something like this:

```
root:s8OebdUUcHNvg:15910:0:99999:7:::
```

Edit `/etc/ssh/sshd_config` to allow root login with no password:

```
PermitRootLogin yes
PermitEmptyPasswords yes
```

Remove the ssh host keys:

```
rm -f /etc/ssh/ssh_host*
```

Add the following to `/etc/rc.local` to have the ssh host keys regenerated upon next boot.  **NOTE**: Make sure you add this section **before** the line which says `exit 0`:

(thanks to https://forums.opensuse.org/showthread.php/477727-Regenerate-SSH-config-after-virtual-machine-cloning)

```
if [ ! -e /etc/ssh/ssh_host_rsa_key ]
then
    dpkg-reconfigure openssh-server
    /etc/init.d/ssh restart || true
fi
```

Don't forget to make sure `/etc/rc.local` is set executable:

```
chmod +x /etc/rc.local
```

Shutdown the pogoplug and remove the SD card:

```
sync
poweroff
```

### Create the redistributable rootfs tarball

Insert the SD card back into your Debian `jessie` PC.

Double-check that your `${dev}` env var is still correct:

```
fdisk -l /dev/${dev}
```

Mount the SD card:

```
mount /dev/${dev}1 ${mnt}
```

Remove root's `.bash_history` file:

```
rm ${mnt}/root/.bash_history
```

Create the rootfs tarball:

```
cd ${mnt}
tarball_date=$(date +%Y%m%d)
tar c . | gzip > ~/Debian-jessie-3.18.5-pogoplug-v4-${tarball_date}-rootfs.tar.gz
```

Unmount the SD card:

```
cd
umount ${mnt}
sync
```

### Create the redistributable (minimal) disk image

Zero out the first 512,000,000 bytes of the SD card:

```
dd if=/dev/zero of=/dev/${dev} bs=1MB count=512
sync
```

Create a bootable partition of (at least) 480MiB on the SD card (`sfdisk` will round up to the nearest cylinder):

```
sfdisk -uM /dev/${dev} << 'EOF'
,480,,*
EOF
```

Format the partition:

```
mke2fs -j -L rootfs /dev/${dev}1
```

Mount the SD card:

```
mount /dev/${dev}1 ${mnt}
```

Unpack the tarball:

```
cd ${mnt}
cat ~/Debian-jessie-3.18.5-pogoplug-v4-${tarball_date}-rootfs.tar.gz | gunzip | tar x
```

Unmount the SD card:

```
cd
umount ${mnt}
sync
```

Determine the exact size of the disk image:

```
sfdisk -l -uS /dev/${dev}
```

Example output:

```
# sfdisk -l -uS /dev/${dev}

Disk /dev/sdf: 1021 cylinders, 245 heads, 62 sectors/track
Units: sectors of 512 bytes, counting from 0

   Device Boot    Start       End   #sectors  Id  System
/dev/sdf1   *         1    987349     987349  83  Linux
/dev/sdf2             0         -          0   0  Empty
/dev/sdf3             0         -          0   0  Empty
/dev/sdf4             0         -          0   0  Empty
```

Here, we need to copy exactly `(1 + 987349) * 512` bytes of the disk in order to make a usable disk image.  We will tell `dd` to copy 987350 blocks of 512 bytes.

Create the disk image:

```
dd if=/dev/${dev} bs=512 count=987350 | gzip > ~/Debian-jessie-3.18.5-pogoplug-v4-${tarball_date}-disk-image.dd.gz
```

### Create the 4GB disk image

Zero out the first 4,000,000,000 bytes of the SD card:

```
dd if=/dev/zero of=/dev/${dev} bs=1MB count=4000
sync
```

Create a bootable partition of (at least) 3800MiB on the SD card (`sfdisk` will round up to the nearest cylinder):

```
sfdisk -uM /dev/${dev} << 'EOF'
,3800,,*
EOF
```

Format the partition, mount the SD card, unpack the tarball, then unmount the SD card:

```
mke2fs -j -L rootfs /dev/${dev}1
mount /dev/${dev}1 ${mnt}
cd ${mnt}
cat ~/Debian-jessie-3.18.5-pogoplug-v4-${tarball_date}-rootfs.tar.gz | gunzip | tar x
cd
umount ${mnt}
sync
```

Determine the exact size of the disk image:

```
sfdisk -l -uS /dev/${dev}
```

Example output:

```
# sfdisk -l -uS /dev/${dev}

Disk /dev/sdf: 1021 cylinders, 245 heads, 62 sectors/track
Units: sectors of 512 bytes, counting from 0

   Device Boot    Start       End   #sectors  Id  System
/dev/sdf1   *         1   7792469    7792469  83  Linux
/dev/sdf2             0         -          0   0  Empty
/dev/sdf3             0         -          0   0  Empty
/dev/sdf4             0         -          0   0  Empty
```

Here, we need to copy exactly `(1 + 7792469) * 512` bytes of the disk in order to make a usable disk image.  We will tell `dd` to copy 7792470 blocks of 512 bytes.

Create the disk image:

```
cd
dd if=/dev/${dev} bs=512 count=7792470 > Debian-jessie-3.18.5-pogoplug-v4-${tarball_date}-disk-image.4GB.img
zip Debian-jessie-3.18.5-pogoplug-v4-${tarball_date}-disk-image.4GB.img.zip Debian-jessie-3.18.5-pogoplug-v4-${tarball_date}-disk-image.4GB.img
rm Debian-jessie-3.18.5-pogoplug-v4-${tarball_date}-disk-image.4GB.img
```

### Generate an `md5sums` file:

```
cd
md5sum *rootfs.tar.gz *disk-image.dd.gz *disk-image.4GB.dd.zip > md5sums
```

## Related resources:

You may also be interested in my other Pogoplug github projects:

* [pogoplug_mobile_uboot_installer](https://github.com/pepaslabs/pogoplug_mobile_uboot_installer)
* [pogoplug_static_binaries](https://github.com/pepaslabs/pogoplug_static_binaries)
