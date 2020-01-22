# Containers

## Setup

### Preqs

* Ensure you have `node --version >= 12.14`.
* Ensure you have `bakerx --version >= 0.6.3`.
* Ensure you have VirtualBox installed.

## Creating a simple container with chroot

### Create a headless micro VM with bakerx

Pull an 3.9 alpine image and create a micro VM called, `con0`.

```bash
bakerx pull ottomatica/slim#images alpine3.9-simple

bakerx run con0 alpine3.9-simple
```

### Prepare a simple rootfs with busybox.

```bash
mkdir -p rootfs/bin rootfs/sbin rootfs/usr/bin rootfs/usr/sbin

wget https://www.busybox.net/downloads/binaries/1.31.0-defconfig-multiarch-musl/busybox-i686 -O rootfs/bin/busybox
chmod +x rootfs/bin/busybox
```

Install symlinks inside the rootfs
```bash
chroot rootfs /bin/busybox --install -s
```

### Playing with container

Let's try it out!

```bash
PS1="C-$ " chroot rootfs /bin/busybox sh
C-$ touch HELLO.txt
C-$ ls
C-$ exit
```

There is a minor problem. We have not perserved the isolation property for our filesystem, as the filesystem is still mutable.

You can see new file exists in the file system, and worse, we can delete and mess up things.
```bash
PS1="C-$ " chroot rootfs /bin/busybox sh
ls
rm HELLO.txt
```

### Introducing overlay filesystem

Create a new [container.sh](container.sh) file inside your micro VM and make it exectuable.
This script will create a new snapshot of the filesystem everytime is it launched in order to run the container.

```bash
#!/bin/ash

# Expect rootfs as script argument
ROOTFS=$1

# Create a random unique string
nonce=$(</dev/urandom tr -dc A-Za-z0-9-_ | head -c 10)

# Prepare our filesystem for our container.
mkdir -p /tmp/$nonce/upper /tmp/$nonce/workdir /tmp/$nonce/overlay
# Create an overlay filesystem based on our read-only root filesystem.
mount -t overlay -o lowerdir=$ROOTFS,upperdir=/tmp/$nonce/upper,workdir=/tmp/$nonce/workdir none /tmp/$nonce/overlay

# Create symlinks in our container for convience.
chroot /tmp/$nonce/overlay/ /bin/busybox --install -s

# Launch container with custom prompt in ash shell.
PS1="$nonce-# " chroot /tmp/$nonce/overlay /bin/busybox sh
```

Using the overlay filesystem, we can keep our rootfs "read-only", while allowing new changes to be made. The read-only portion is denotated by the "lower" directory. The change states are maintained in the "upper" and "work" directories, and the merged/unified filesystem is available in the "overlay" directory.

A short demo of the script illustrates our ability to perserve the rootfs each time we create a new container.

![demo](imgs/simple-chroot.png)


### Summary

While we have demonstrated a very simple way to implement containers---however, there are several limitations with our implementation.

* Preparing root file systems can be problematic. Can we do better than random wget scripts?
* Keeping multiple root filesystems can be inefficient --- We are not taking advantage of common shared files.
* Other devices of the host OS are not isolated, such as networking.
* We may want to share and isolate other resources with the host. For example, if we mount the proc filesystem, we can kill any process on the system. Not good! A rogue container could interfere with other containers or bring down the whole system.

  ```bash
  mkdir -p /tmp/$nonce/overlay/proc
  mount -t proc none /tmp/$nonce/overlay/proc
  ```

## Introducing Docker containers.

If we worked on our script for several more years and took advantage of other capabilities of the Linux kernel, such as cgroups and namespaces, we could overcome many of the limitations we saw with our homegrown containers. Thankfully, someone has done this work for us.

### Setup 

#### Create a headless micro VM with bakerx

Pull an ubuntu bionic image and create a micro VM called, `docker0`.

```bash
bakerx pull cloud-images.ubuntu.com bionic
bakerx run docker0 bionic
```

#### Install docker inside your VM.

The recommended method for installing docker on ubuntu can be [found here](https://docs.docker.com/install/linux/docker-ee/ubuntu/#install-docker-ee-1).  But in the interest of time: :grimacing:

```
curl -sSL https://get.docker.com/ | sh
```

After installation, it is recommended you allow docker to be run without needing sudo. **IMPORTANT** You'd need to restart your shell (exit and log back in) to see the changes reflected.

```
sudo usermod -aG docker $(whoami)
```

Verify you can run docker:
```
docker run hello-world
```

### Playing with Docker

Pull an ubuntu image.

```
$ docker pull ubuntu:18.04
```

Check the size of the images.

```bash
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              18.04               ccc6e87d482b        6 days ago          64.2MB
hello-world         latest              fce289e99eb9        12 months ago       1.84kB
```

Let's create a simple container.

```bash
docker run ubuntu:18.04 sh
```

Unfortunately, that doesn't quite do what we want, because a pseudo-tty allocated to the container, so we can use the terminal.

```
$ docker run -t ubuntu:18.04 sh
# ls

exit
.exit
^C
```

Unfortunately, that's still not enough. Docker isn't necessarily the most intuitive system, you see. You also need to allow input (`-i` Keep STDIN open even if not attached).

```
$ docker run -it ubuntu:18.04 sh
```

Finally, we can get to a terminal, and see we can run shell commands in our own private snapshot of the ubuntu image.


```bash
$ docker run -it ubuntu:18.04 sh
# ls
# rm -rf --no-preserve-root /
# exit
```

Oh no, what have we done? Is everything okay?

```bash
$ docker run -it ubuntu:18.04 sh
```

### Building Images

We can create our own images.
Create a "Dockerfile" and place this content inside:

```
FROM ubuntu:18.04

RUN apt-get -y update

# update packages and install
RUN apt-get install -y openjdk-11-jre-headless wget curl unzip

RUN apt-get -y install git
RUN apt-get -y install maven

ENV JAVA_HOME /usr/lib/jvm/java-11-openjdk-amd64
```

Build the docker image, and name it "java11".

```bash
docker build -t java11 .
```

We can see our image listed, now:

```bash
docker images
```

Now, let's use it to run `mvn`.

```bash
docker run java11 mvn
```



