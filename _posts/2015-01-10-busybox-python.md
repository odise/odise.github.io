---
layout: post
title: "busybox-python Docker image"
tagline: "Minimal Docker base image for aws-cli"
category: 
tags: [Docker, busybox, AWS, aws-cli]
---
{% include JB/setup %}

In my last post I demonstrated how to build a minimal Docker images for the [cron-container](). All in all this approach works pretty well for the described use case. Following this road I wanted to use [aws-cli]() for periodical [AWS]() infrastructure updates. The `aws` command line tool is implemented in Python which needs to be installed inside the container. I found the [python base images]() which provides the interpreter but comes with a huge overhead. The container became kind of huge for a simple script (the `python:2-slim` image has a size of approx. 200mb). 

Searching around [pointed](http://yasermartinez.com/blog/posts/creating-super-small-docker-images.html) me to [Pyrun]() - a static linked Python binary of 11mb size. In combination with [progrium/busybox](https://ithub.com:progrium/busybox) the container becomes much smaller. Missing dependencies needed by Pyrun (Openssl, zlib, SQlite, bzip2) can be installed with the help of the `opkg`.

So far so good. Using `curl` and `aws` inside the container reveals SSL/TLS problems. I was able to tweak `curl` to run without the `--insecure` option by adding `--capath` pointing to the CA certificates provided by the `ca-certificates` packages installed with `opkg`. However `aws` and especially [botocore](https://github.com/boto/botocore) refused to run without the `--no-verify-ssl` option (got some weird `SSL error in _ssl.c:510`). 

I tried hard but wasn't able to overcome the SSL/TLS termination problems with the `progrium/busybox` base image. So I started to build my own base image providing all the necessary CA certificates and libraries needed for Pyrun. 

Busybox is the best approach to implement small Docker base images I know (please point me to alternatives). Building a `rootfs.tar` with the help of [buildroot](http://buildroot.net) has been done by many people in the past and a lot of boilerplate projects exists out there. [My solution](https://registry.hub.docker.com/u/odise/busybox-python/) is based on the `tarmaker` component of the [radial/busyboxplus](https://github.com/radial/core-busyboxplus) project. 

Lets have a look on how to build this base image by looking at the Dockerfile:

```bash
FROM scratch
ADD rootfs.tar /
```

It all starts with the creation of the root filesystem. Busy box can be configured and compiled with the help of the Buildroot toolchain but how do you do this inside a container? Here are the steps:

1. Create a Ubuntu based container and bring it up-to-date.
2. Install some basic tools we will need for the compilation of Busybox (wget,                    build-essential, libncurses-dev, rsync, unzip etc).
3. Get the Buildroot archive of your choice, put it somewhere inside the container.
4. Configure the build. This works very much the same like a Linux kernel build configuration controlled by `make config` or `make menueconfig`. 
 