---
layout: post
title: "Docker hosts deployment pattern"
tagline: "using images, systemd and cloud-init"
category: 
tags: [ AWS, cloud-init, systemd, Docker, Packer ]
---
{% include JB/setup %}

**THIS IS WORK IN PROGRESS**

The deployment pattern for setting up services in a cloud like environment has somewhat changed over the last weeks. The reason is mainly container based application deployments. This happened for AWS as well as for (more or less static) data centers that provides you with bare VM resources which can be created and destroyed up on demand. A few month ago deployments have done with tools like Chef, Ansible and friends to provision a whole machine with everything thats needed for a specific application. Software containers (and namely Docker) has changed that quite a bit. 

Setting up Docker host machines as easy and flexible as possible is the name of the game. The easiest approach for that is: _running one container per node_. Of cause this is not the most efficient way to use VM resources but the best in terms of simplicity.

Let us have a look on some basic concepts how to do this for a virtual machine inside cooperate data-center and on an AWS EC2 instance.

# Basic tasks 

Here comes our check list including the major tasks that needs to be done to setup a Docker host:

* Store public SSH keys for administrators. Disable `root` login. Set the correct timezone etc.
* Update the system, enable `yum` or `apt` repositories to install further applications.
* Install basic tools like `curl`, `tmux`, `nmap`, `awscli` or what ever else you need in order to check your system is running as expected. 
* Install `docker` and add user to the `docker` group.
* Install monitoring tools and agents like [Datadog](http://docs.datadoghq.com/guides/basic_agent_usage) or [NewRelic](https://docs.newrelic.com/docs/apm/new-relic-apm/installation-configuration/installing-agent).
* At the end of this chain you want to install you container and start them. 

# Deploy your stuff

Following the list above you have a couple of options depending on the toolchain and infrastructure you have at hand. An obvious choice to do this is putting everything in an Ansible playbook or Chef cookbook and run it on VM creation. Done! This works perfectly fine on your cooperate data-center but has one disadvantage: it takes quite some time  and needs to be done on every VM creation. Looking at the list again some of the tasks can be done beforehand by moving them to a VM base image (e.g. [VMware image](https://solutionexchange.vmware.com/store/category_groups/virtual-appliances) or [AWS AMI](https://aws.amazon.com/marketplace/ref=brs_navhdr_header)). 

# Base image creation

I found [Packer](https://www.packer.io/docs) being an fantastic tool for building all kind of images. It supports all kind of provisioning tools so that you can use your cookbooks and playbook with it. Even difficult tasks like the installation of a Nvidia graphic driver which depends on system reboots in the middle of it are supported. Here is an example snippet to install the graphic driver on RHEL7 for a `g2.2xlarge` EC2 instance on AWS:

```
{
  "variables": {
    "aws_access_key": "",
    "aws_secret_key": ""
  },
  "builders": [{
    "type": "amazon-ebs",
    "access_key": "{{user `aws_access_key`}}",
    "secret_key": "{{user `aws_secret_key`}}",
    "region": "eu-west-1",
    "source_ami": "ami-8cff51fb",
    "instance_type": "g2.2xlarge",
    "ssh_username": "ec2-user",
    "ssh_timeout" : "15m",
    "ami_name": "nvidiaserver-rhel7 {{timestamp}}"
  }],
  "provisioners": [
    {
      "type": "shell",
      "scripts": [ 
        "scripts/install-base-tools.sh",
        "scripts/remove-nouveau-driver.sh"
      ]
    },
    {
      "type": "shell",
      "inline": [
        "echo 'rebooting the system'",
        "sudo /usr/sbin/reboot",
        "sleep 60"
      ]
    },
    {
      "type": "shell",
      "script": "scripts/install-nvidia-driver.sh"
    }
  ]
}

``` 

Starting from a base image, that has all the tools you need installed and running a OS update procedure on VM startup, can speed up things dramatically. For automatic instance up- and downscaling things needs to be fast. If OS updates take to long in this context, the base images can be build on a regular base as part of your CI lifecycle.

# Handling dynamic instance data 

Truly not everything can be handled within base image. There is a need for OS system updates and container deployment at the initial instance launch. Generally Ansible and friends can help here as long as this procedure is kept as short as possible:

* deploy system updates
* deploy container images and configuration

For AWS EC2 instances (and a [bunch of others](https://cloudinit.readthedocs.org/en/latest/topics/datasources.html) including OpenStack, vSphere and even [Vagrant](http://davemartorana.com/logs/software/cloud-init-in-vagrant-with-ubuntu-12-10-13-04/)) dynamic configuration can be done with [user data](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html#instancedata-user-data-retrieval) based on [cloudinit](http://cloudinit.readthedocs.org/). Using it gives you the opportunity to set some last bits on a freshly launched VM. 

A lot of things can be told about it but what I found most important is how to test cloud-init user data without firing an instance a dozen times before your code works as expected. Testing can be done on a running instance by creating a `seed` directory for the [NoCloudNet](http://cloudinit.readthedocs.org/en/latest/topics/datasources.html#no-cloud) datasource, putting the user data in there and restarting the `cloud-init` service:


```
$ mkdir /var/lib/cloud/seed/nocloud-net/
$ cat /var/lib/cloud/seed/nocloud-net/meta-data
ami-id: ami-4be0633c
local-hostname: ip-172-31-10-159.eu-west-1.compute.internal

$ cat /var/lib/cloud/seed/nocloud-net/user-data
#cloud-config

runcmd:
 - [ sh, -c, 'echo ==== $(date) ====; echo HI WORLD; echo =======' ]

$ # remove the data from /var/lib/cloud/instance/ to mimic a freshly created instance
$ rm -rf /var/lib/cloud/instance/*
$ # restart the cloud-init service 
$ sudo systemctl restart cloud-init
$ # check the logs
$ sudo journalctl -u cloud-init
...
$ less /var/log/cloud-init.log
```
BTW: [CoreOS](https://github.com/coreos/coreos-cloudinit) machine customization is heavily using the same mechanism. Even more interesting the guys reimplemented the cloudinit specification (originally in Python) in Golang. The project is available on Github: [coreos-cloudinit](https://github.com/coreos/coreos-cloudinit).

# Starting Docker containers

The last piece in the puzzle. Again nearly all configuration management software too;chains support Docker container deployments out of the box now. However this being the most volatile part of the provisioning we always want to have the latest version of the container available. Moreover preferably we want to update the containers without destroying the whole instance. 

Using [systemd](http://www.freedesktop.org/wiki/Software/systemd/) unit files to `pull` and `run` containers is a very interesting approach. On the [CoreOS how-to pages](https://coreos.com/docs/launching-containers/launching/getting-started-with-systemd/)  some examples can be found. Here is a snippet:

```
[Unit]
Description=MyApp
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill nginx
ExecStartPre=-/usr/bin/docker rm nginx
ExecStartPre=/usr/bin/docker pull nginx
ExecStart=/usr/bin/docker run --name nginx -p 443:443 nginx

[Install]
WantedBy=multi-user.target
```

With this unit file, installed on `/etc/systemd/system/nginx.service` and enabled, Docker will download the missing Nginx image from the configured Docker registry and run it on system startup. Via the _user data_ mechanism described above you can easily control which container should be started on which host. Additional unit files can handle container updates and image housekeeping (e.g. controlled via system reboot or SSH interaction etc.). 

Docker registries authentication can be handled with the help of `.dockercfg` and the [`User=`](http://www.freedesktop.org/software/systemd/man/systemd.exec.html) option of systemd.

Using this quite a while makes you maybe halting. Things sometimes do not work as expected. Container don't get killed or removed properly etc. Systemd does not actually supervise the Docker container you are starting in the unit file but instead the Docker client. This makes systemd incapable of reliably managing Docker containers. Luckily there is a quite neat workaround for that: [systemd-docker](https://github.com/ibuildthecloud/systemd-docker). Please check the Github page to get more background information about the problem. Using this wrapper your unit file looks a bit different though:

```
[Unit]
Description=MyApp
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
# put your environment options in a file 
EnvironmentFile=/etc/sysconfig/mycontainer
ExecStart=/usr/local/bin/systemd-docker --cgroups name=all --env run -it --rm --name %n  -p 443:443 mycontainer
Restart=always
RestartSec=10s
Type=notify
NotifyAccess=all
TimeoutStartSec=600
TimeoutStopSec=15

[Install]
WantedBy=multi-user.target
```

I think for the time being this little tool improves the way Docker containers and systemd collaborate. There are other container implementations rising so it's going to be an interesting 2015.
