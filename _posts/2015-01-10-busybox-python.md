---
layout: post
title: "busybox-python Docker image"
tagline: "Minimal Docker base image for aws-cli"
category: 
tags: [Docker, busybox, AWS, aws-cli]
---
{% include JB/setup %}

In my last post I demonstrated how to build a minimal Docker image for the [cron-container](https://github.com/odise/cron-container). All in all this approach works pretty well for the described use case. Following this road I wanted to use [aws-cli](https://github.com/aws/aws-cli) for periodical AWS infrastructure updates. The `aws` command line tool is implemented in Python which needs to be installed inside the container. I found the [python base images](https://registry.hub.docker.com/_/python/) which provides the interpreter but comes with a huge overhead. The container became kind of huge for a simple script (the `python:2-slim` image has a size of approx. 200mb). 

Searching around [pointed](http://yasermartinez.com/blog/posts/creating-super-small-docker-images.html) me to [Pyrun](https://www.egenix.com/products/python/PyRun/) - a static linked Python binary of 11mb size. In combination with [progrium/busybox](https://github.com/progrium/busybox) the container becomes much smaller. Missing dependencies needed by Pyrun (Openssl, zlib, SQlite, bzip2) can be installed with the help of the `opkg`.

So far so good. Sadly I realized that using `curl` and `aws` inside the container reveals SSL/TLS problems. I was able to tweak `curl` to run without the `--insecure` parameter by adding `--capath` pointing to the CA certificates (those are provided by the `ca-certificates` packages that can be installed with `opkg`). However `aws` and especially [botocore](https://github.com/boto/botocore) Python package refused to run without the `--no-verify-ssl` option (got some weird `SSL error in _ssl.c:510`). 

I tried hard but wasn't able to overcome the SSL/TLS termination problems with the `progrium/busybox` base image. So I started to build my own base image providing all the necessary CA certificates and libraries needed for Pyrun. 

Busybox is the best approach to implement small Docker base images I know (please point me to alternatives). Building a `rootfs.tar` with the help of [Buildroot](http://buildroot.net) has been done by many people in the past and a lot of boilerplate projects exists out there. [My solution](https://registry.hub.docker.com/u/odise/busybox-python/) is based on the `tarmaker` component of the [radial/busyboxplus](https://github.com/radial/core-busyboxplus) project. 

How to build a Docker base image from scratch? Lets have a look on a Dockerfile that does nothing but setup the basics:

```bash
FROM scratch
ADD rootfs.tar /
```

It all starts with the creation of the root filesystem. Busybox can be configured and compiled with the help of the Buildroot toolchain. Ideally this will be done as part of a container build. Here are the steps to create the filesystem as part of a `docker build` command:

1. Create a Ubuntu based container with `docker build` and bring it up-to-date.
2. Install some basic tools needed for the compilation of Busybox (wget,                    build-essential, libncurses-dev, rsync, unzip etc).
3. Get the Buildroot archive of your choice, put it somewhere inside the container.
4. Configure the build. This works very much the same as a Linux kernel build.
5. Build the whole Busybox toolchain. This will create a tar archive inside the Docker container that can be used as root filesystem with Docker.
6. Run the created Docker image and copy the root filesystem tar archive out of the Docker container.

Please have a look at the [`run.sh`](https://github.com/odise/busybox-python/blob/master/run.sh) script and the tarmaker [`Dockerfile`](https://github.com/odise/busybox-python/blob/master/tarmaker/Dockerfile) for further details.

The rest is a piece of cake. Here is the [odise/busybox-python](https://registry.hub.docker.com/u/odise/busybox-python/) base image Dockerfile:

```
FROM scratch

ADD rootfs.tar /

# install standalone python binary
ADD pyrun2.7 /bin/python2.7
RUN ln -s /bin/python2.7 /bin/python2
RUN ln -s /bin/python2.7 /bin/python
RUN ln -s /usr/lib/libbz2.so.1.0 /usr/lib/libbz2.so.1

RUN mkdir -p /lib/python2.7/site-packages/
RUN mkdir -p /include/python2.7/

# install easy_install
RUN curl https://bitbucket.org/pypa/setuptools/raw/0.7.4/ez_setup.py -o - | python - --user
RUN rm -f /setuptools-0.7.4.tar.gz

CMD [ "/bin/sh" ]
```
 
Using the `aws-cli` tool inside your own container of small size becomes easy now:  

```
FROM odise/busybox-python:stable

RUN easy_install awscli
```

There might be some [limitations](https://www.egenix.com/products/python/PyRun/doc/#_Toc390548380) due to the nature of Pyrun. I didn't tested tools other then `aws` so far.  