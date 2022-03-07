---
layout: post
title: "Init RAMFS"
date: 2022-02-25
categories: jekyll blogging
published: false
---

The initial RAM disk (initrd) is an initial root file system that is mounted prior to when the real root file system is available. The initrd image contans a minimal set of directories and executables to support the second stage boot. But on some small system, it can be also the permanent root file system.

## How to create initramfs image

The old initrd in 2.4 Linux kernel is a gzipped file system image, e.g. ext2 format. It is usually constructed using the loop device, and needs the filesystem driver built into the kernel.

From 2.6 kernel, gzipped cpio format initramfs archive is used to create the initrd. The initramfs is always built-in kernel, so it does not require any external driver to load the initrd.

(If you're curious why cpio format is used rather than tar, you can find the background from the reference [ramfs, rootfs and initramfs] below)

```bash
wget https://busybox.net/downloads/busybox-1.26.2.tar.bz2
tar -xvf busybox-1.26.2.tar.bz2

cd busybox-1.26.2
make defconfig
make menuconfig

make
make CONFIG_PREFIX=./../busybox_rootfs install

mkdir -p initramfs/{bin,dev,etc,home,mnt,proc,sys,usr}
cd initramfs/dev
sudo mknod sda b 8 0
sudo mknod console c 5 1

echo "#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
exec /bin/sh" > init

chmod +x init

find . -print0 | cpio --null -ov --format=newc > initramfs.cpio
gzip ./initramfs.cpio
```

## How initramfs is loaded by kernel

## Links

- [ramfs, rootfs and initramfs](https://www.kernel.org/doc/Documentation/filesystems/ramfs-rootfs-initramfs.txt)
- [Using the initial RAM disk (initrd)](https://www.kernel.org/doc/html/latest/admin-guide/initrd.html)

