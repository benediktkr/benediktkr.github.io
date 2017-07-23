---
layout: post
title:  "Remote full disk encryption on Linux"
date:   2015-05-01 03:32:17 +0000
categories: ops
---

Using full disk encrytion means that you need to enter your passphrase
before booting the computer. If the computer in question is in a
remote location (such as a datacenter), this means that you don't have
access to the console and thus cannot enter the passphrase needed to
unlock the disks. However, Linux offers the possibility of running a
lightweight SSH server before fully booting the system so that you can
enter the key.

We will be setting up LUKS (with `dm-crypt`) on Debian
Wheezy. [According to the cryptsetup Google Code page][0], the cipher
mode `aes-xts-plain` should not be used for encrytped container size
larger than 2 TiB, `aes-xts-plain64` should be used.

# Drawbacks

* The ssh server keys for the small pre-boot ssh daemon need to be
  stored unencrypted. This means that an attacker could possibly
  access them and fake our server, tricking us into giving up the
  passphrase.

# Preparation

Before proceeding, install the needed packages to run the small ssh
server during the boot process[2].

    # apt-get install busybox dropbear

This is the default on Debian, but check if
`/etc/initramfs-tools/initramfs.conf` contains `BUSYBOX=y`. It's also
important that it doesn't contain `DROPBEAR=n` (the default is
`y`). Also verify that the keys for dropbear were generated, located
in `/etc/initramfs-tools/etc/dropbear/`. if not, use `dropbearkey` to
create RSA and DSS keys.

As the initramfs will not be encrypted, publickey authentication is
now assumed. Keys have probably been configured for you in
`/etc/initramfs-tools/root/.ssh/`. Following are the commands to
create the keys manually.

First we have to create a key in the format used by dropbear. We then
convert it to the format used by OpenSSH and extract the public part,
before adding it to `authorized_keys`.

    # dropbearkey -t rsa -f /etc/initramfs-tools/root/.ssh/id_rsa.dropbear
    # dropbearconvert dropbear openssh  /etc/initramfs-tools/root/.ssh/id_rsa.dropbear /etc/initramfs-tools/root/.ssh/id_rsa
    # dropbearkey -y -f /etc/initramfs-tools/root/.ssh/id_rsa.dropbear |  grep "^ssh-rsa " > /etc/initramfs-tools/root/.ssh/id_rsa.pub
    # cat /etc/initramfs-tools/root/.ssh/id_rsa.pub >> /etc/initramfs-tools/root/.ssh/

Later on, we will generate a new initramfs that includes dropbear and
these keys. You should copy the ssh keys to your computer.

If your disks are already encrypted, you can stop here and just
generate a new initramfs with `update-initramfs -u`. Otherwise it's
time to reboot into your particular variation of a rescue
OS to encrypt the disks. E.g. connecting via ssh:

# Encrypting the disks remotely

Usually when renting dedicated servers, you won't get to do the OS
install yourself so you will have a server with unencrypted
disks. Other hosting providers will let you have a remote console so
you can set up your disks with encrypted disks.

This section deals with setting up FDE post-install. We need to do
this as soon after install as possible, because we will have to keep
the current OS setup in memeory while encrypting the disks.

    localhost:~$ ssh -o "UserKnownHostsFile=~/.ssh/known_hosts.rescue" root@server
    root@server's password: 
    root@rescue ~ #

Before encrypting the drives, we need to copy our old installation
into memory since the disks will be wiped clean:

    # mkdir /oldroot
    # mount /dev/mapper/vg0-root /mnt/
    # mount /dev/mapper/vg0-home /mnt/home/
    # mount /dev/mapper/vg0-srv /mnt/srv/
    # rsync -a /mnt/ /oldroot/
    # umount /mnt/home/
    # umount /mnt/srv/
    # umount /oldroot

In order to encrypt the partition we need to remove the current LVM
setup. We also backup the vg config to make life easier.

    # vgcfgbackup vg0 -f vg0.freespace
    # vgremove vg0

Now we can encrypt the disk drive. Since this is a server that will
boot relatively seldom, we specify `--iter-time 6000` to determine the
number of iterations needed for the server to spend 6 seconds on the
key derivation function.

    # cryptsetup luksFormat --cipher aes-xts-plain64 -s 256 --iter-time 6000 /dev/md1
    # cryptsetup luksDump /dev/md1
    # cryptsetup luksOpen /dev/md1 cryptroot
    # ls /dev/mapper/cryptroot

Now we will re-create the LVM volume group, but we need to edit the
backup. Instead of the volume group using `/dev/md1` as it's physical
volume, it will now use `/dev/mapper/cryptroot`. We need to
[find UID for `cryptroot`][1] and edit the backup accordingly. The
settings are `id` and `device` under `vg0.physical_volumes.pv0`.

    # blkid /dev/mapper/cryptroot
    /dev/mapper/cryptroot: UUID="HEZqC9-zqfG-HTFC-PK1b-Qd2I-YxVa-QJt7xQ"
    # cp vg0.freespace /etc/lvm/backup/vg0
    # nano /etc/lvm/backup/vg0
    [Edit file]
    # vgcfgrestore vg0
    Restored volume group vg0
    # vgchange -a y vg0

The next task is to create filesystems on the logical volumes and then restore the system

    # mkfs.ext4 /dev/vg0/root
    # mkfs.ext4 /dev/vg0/home
    # mkfs.ext4 /dev/vg0/srv
    # mount /dev/vg0/root /mnt
    # mkdir /mnt/home /mnt/srv
    # mount /dev/vg0/home /mnt/home
    # mount /dev/vg0/srv /mnt/srv
    # rsync -a /oldroot/ /mnt/

Before we can change root into our system we need the special filesystems. We need to change root to fix some things

    # mount /dev/md0 /mnt/boot
    # mount --bind /dev /mnt/dev
    # mount --bind /sys /mnt/sys
    # mount --bind /proc /mnt/proc
    # chroot /mnt

We need a crypttab to be able to boot from the encrypted filesystem. Since we installed LVM from the image, we can leave the fstab as-is.

`/etc/crypttab`:
    # <target name> <source device>         <key file>      <options>
    cryptroot /dev/md1 none luks

`/etc/fstab:`
    proc           /proc   proc defaults 0 0
    /dev/md/0      /boot   ext3 defaults 0 0
    /dev/vg0/root  /       ext4  defaults 0 0
    /dev/vg0/home  /home   ext4  defaults 0 0
    /dev/vg0/srv   /srv    ext4  defaults 0 0
    /dev/vg0/swap  swap    swap  defaults 0 0

We just need to tell grub about the new root device
and regenerate the initramfs to include dropbear and busybox. This
will use the keys we generated for `/etc/initramfs-tools/root/.ssh/`.

    # update-initramfs -u
    # update-grub
    # grub-install /dev/sda
    # grub-install /dev/sdb
    # echo "/sbin/ifdown --force eth0" >> /etc/rc.local
    # echo "/sbin/ifup --force eth0" >> /etc/rc.local


And now, reboot and hope for the best

    # exit
    # umount /mnt/boot /mnt/home /mnt/srv /mnt/proc /mnt/sys /mnt/dev
    # umount /mnt
    # sync
    # reboot

# Unlocking remotely

    localhost:~$ ssh -o "UserKnownHostsFile=~/.ssh/known_hosts.initramfs" -i ~/.ssh/id_rsa.initramfs root@server
    ~ # echo -ne $PASSPHRASE > /lib/cryptsetup/passfifo


[0]: https://code.google.com/p/cryptsetup/wiki/FrequentlyAskedQuestions
[1]: https://unix.stackexchange.com/questions/107810/why-my-encrypted-lvm-volume-luks-device-wont-mount-at-boot-time
[2]: https://unix.stackexchange.com/questions/5017/ssh-to-decrypt-encrypted-lvm-during-headless-server-boot
