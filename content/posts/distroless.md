---
title: "Exploring distroless docker images"
date: 2023-08-27T20:53:10+05:30
draft: false
type: post
tags: [Containers, Linux]
showTableOfContents: true
---

I recently discovered the concept of [distroless](https://github.com/GoogleContainerTools/distroless) docker images. However, this project is quite a few years old.[^old]

The name may be quite misleading (much like "serverless") - as distroless images are actually based on Debian (currently Debian 11, Bullseye). However, they don't contain most of the stuff which is there in standard debian image, which such as all the command line executables (`coreutils` etc..)

### What's there in a standard Debian image?

Let's pull and inspect a standard debian docker image:

```bash
docker run -it debian:11 bash
```

Let me just list the packages installed in a standard debian image. Which is quite a lot.

```
root@abccaea3d8d3:/# apt list --installed
Listing... Done
adduser/now 3.118 all [installed,local]
apt/now 2.2.4 amd64 [installed,local]
base-files/now 11.1+deb11u7 amd64 [installed,local]
base-passwd/now 3.5.51 amd64 [installed,local]
bash/now 5.1-2+deb11u1 amd64 [installed,local]
// ----------------------------------------------------------------------
// ----- AROUND 80 PACKAGES REDACTED FOR READABILITY OF THIS POST -------
// ----------------------------------------------------------------------
tar/now 1.34+dfsg-1 amd64 [installed,local]
tzdata/now 2021a-1+deb11u10 all [installed,local]
util-linux/now 2.36.1-8+deb11u1 amd64 [installed,local]
zlib1g/now 1:1.2.11.dfsg-2+deb11u2 amd64 [installed,local]
```

Now most of this will not be used by your production image. However, existence of all these packages creates two disadvantages:

1. The final size of the image will be very high. Debian is the base image for most popular containerized applications, and since they're usually pinned to some point version and never updated again, almost every such image you pull will end up having slightly different layers, causing 100s of MBs of pulls. 

2. The existence of tools like `sh` creates an attack surface. Many security exploits leverage the relative simplicity of invoking a shell command like `sh` and getting arbitrary command execution on the remote machine.

### What are distroless images?

Distroless images are a very stripped down version of Debian base system, consisting of only few essential libraries and files.

For language like Go which can carry its own dependencies, we can use the most barebones version called `static-debian11`, which is smaller than the alpine image! There's also `base-debian11` which contains minimum `glibc` for compiled languages like Rust which still depend on libc.

For languages like Java and NodeJS, there are variant images which consist of the minimal language runtime - such as `gcr.io/distroless/java11-debian11` and `gcr.io/distroless/java11-debian11`. (__Note that you have to pull these by specifying the full registry URL__, since these images are hosted in GCR instead of docker hub. I will use shorter names in the remainder of this article).

We can compare the sizes of `alpine:latest` which is also known for small storage size and widely used, vs the distroless `static-debian11`.

```bash
$ docker image ls --format '{{.Size}}\t{{.Repository}}' | grep -E 'distroless|alpine'
5.54MB  alpine
2.45MB  gcr.io/distroless/static-debian11
```

We all know that Alpine Linux image is very small compared to standard debian etc.. because it uses `musl` and `busybox` over its GNU counterparts, even at the cost of some incompatibility. But the smallest distroless debian-based image is smaller than alpine!

There's a catch though, it doesn't contain anything like `sh` to explore what's inside it. Because there's almost no executables, our normal run / exec commands won't be much helpful.

```
$ docker run -it gcr.io/distroless/static-debian11 sh
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: exec: "sh": executable file not found in $PATH: unknown.
```

### Exploring the distroless image

So we have two ways to explore it.

1. We can export the image to a `tar` using `docker export` command, then unpack the tarball and inspect it.
2. Trickier, but we can bind mount a busybox binary into the image and run it.

I have a particular affinity towards complicating things, so let's go with 2.

I have busybox-static installed which I will just bind-mount.

```
$ docker run -v $(which busybox):/bin/busybox -it gcr.io/distroless/static-debian11 busybox sh


BusyBox v1.30.1 (Ubuntu 1:1.30.1-7ubuntu3) built-in shell (ash)
Enter 'help' for a list of built-in commands.

/ # ls
bin   boot  dev   etc   home  lib   proc  root  run   sbin  sys   tmp   usr   var
```

Let's run `du` to ~~find out~~ guess which directories have larger amount of files.

```
/ # du -hd 1
0       ./sys
0       ./dev
0       ./proc
4.0K    ./tmp
4.0K    ./sbin
156.0K  ./var
2.1M    ./bin
5.5M    ./usr
4.0K    ./boot
4.0K    ./root
308.0K  ./etc
8.0K    ./home
4.0K    ./run
4.0K    ./lib
8.1M    .
```

Ignore `/bin` because we mounted it. Most stuff comes from `/usr/share`. In that most of it comes from timezone data.

I would leave further analysis from here to the reader. The point is that distroless trims down most stuff from the base distribution.

### When is it useful?
Small storage size makes it compelling for variety of use cases. Many projects including kubernetes have standardized on distroless base images.

When I had the impulse to containerize some of my small side projects, I have used distroless base image for both [rrip](https://github.com/mahesh-hegde/rrip) and [zserv](https://github.com/mahesh-hegde/zserv). 

The size difference is however negligible when the application has dependencies like Node JS. Official alpine-based `node:20-alpine` image is 181MB while distroless `nodejs20-debian11` image is 170MB - not much of a difference.

However, it must be noted that there are many tradeoffs besides size.

* Debian based images will be more compatible with C libraries than Alpine based images - which can be a reason to pick distroless.

* Distroless images lack shell (in release configuration at least) and thus are harder to debug especially in remote environments like Kubernetes.

* Distroless images have, theoretically, lesser attack surface due to lack of standard distro executables.

* If your application is mostly static but still depends on C runtime (like rust applications, or Go applications linked with CGO), you need `base-debian11` instead of `static-debian11`, which is 4 times larger than `alpine` at the time of writing. (Because `glibc` vs `musl`). So the choice depends on whether the application works well with `musl` and maybe other Alpine quirks, or expects a more standard environment.

* If your application requires installing anything in the production image (not the build image in 2-stage build) through package manager, then distroless isn't a choice as of today.

There's also one other candidate which has similar value proposition - that is RedHat UBI micro. I haven't really gotten an opportunity to try it yet. UBI micro is considerably larger than distroless / alpine, but it's based on RHEL which is preferred by some people / organizations.

### FAQ: Why not use `FROM scratch` ?
`FROM scratch` is technically the most minimal image you can have, but it doesn't contain SSL certificates, DNS resolv.conf or timezone data, one or the other of which end up being required for real world applications.

[^old]: The repo was created in 2017 which is only 4 years after the initial release of Docker
