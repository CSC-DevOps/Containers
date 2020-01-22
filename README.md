# Containers

## Setup

### Preqs

* Ensure you have `node --version >= 12.14`.
* Ensure you have `bakerx --version >= 0.6.3`.
* Ensure you have VirtualBox installed.

## Creating a simple container. 

### Create a headless micro VM with bakerx

Pull an 3.9 alpine image and create a micro VM called, `con0`.

```bash
bakerx pull ottomatica/slim#images alpine3.9-simple

bakerx run con0 alpine3.9-simple
```

Prepare a simple rootfs with busybox.

```bash
mkdir -p rootfs

wget https://www.busybox.net/downloads/binaries/1.31.0-defconfig-multiarch-musl/busybox-i686 -O rootfs/busybox
chmod +x rootfs/busybox
```



nanobox:/tmp# mkdir -p lower/usr/sbin
nanobox:/tmp# mount -t overlay -o lowerdir=/tmp/lower,upperdir=/tmp/upper,workdir=/tmp/workdir none /tmp/overlay
nanobox:/tmp# chroot overlay/ /bin/busybox --install -s



