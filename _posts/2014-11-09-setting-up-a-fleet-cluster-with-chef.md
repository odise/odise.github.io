---
layout: post
title: "Setting up a fleet cluster with Chef and without CoreOS"
category: 
tags: [etcd, fleet, Chef, Docker, CoreOS]
---
{% include JB/setup %}

Moving your software to [Docker](https://www.docker.com/) is like running a long distance. Putting your software inside containers is only the first two kilometers. Then you start to realize that moving your underlaying infrastructure to handle all your new packaged software is a much longer road. I was used to deploy complex systems with [Chef](https://www.getchef.com/) to a VM build with [Foreman](http://theforeman.org/) using the [Phoenix Principals](//kreuzwerker.de/en/blog/posts/the-phoenix-principle-a-devops-concept-explained/). Moving the components to containers opened the battle field of Docker orchestration. [fleet](https://coreos.com/using-coreos/clustering/) can handle such things and I believe that [CoreOS](https://coreos.com/) is the right technology to run Docker hosts. Unfortunately Foreman currently can not handle CoreOS in a sufficient manner so I was forced to make `fleet` running on CentOS 7. This article describes how I did this.

# Preconditions

In the environment I have at hand, Foreman can be used to (re)build a VM. After spinning up a VM, the bootstrap process provisions it automatically using Chef (server or standalone). The Docker host system I'm going to set up runs CentOS 7 (or RHEL 7 if needed - why?). In order to prepare a `fleet` cluster node I need to write a cookbook that deploys `etcd` version 0.5.0alpha, `fleet` version 0.8.3, `systemd` and of course Docker.

Basically I need to serve two environments: one is a testing cluster running inside a corporate network, the other one is located in a [DMZ](http://en.wikipedia.org/wiki/DMZ_(computing)) like network segment. The later needs to handle HTTP proxy settings which is the reason for choosing `etcd` 0.5. The `ETCD_DISCOVERY_PROXY` option is worth trying the alpha.

# The Cookbook

The cookbook I will referring to can be found on [Github](https://github.com/odise/fleet-cluster-cookbook). It demonstrates the solution based on Vagrant boxes build with [Chef kitchen](https://docs.getchef.com/kitchen.html).

`etcd` can handle cluster setup based on Etcd Discovery Service (discovery.etcd.io or as self-made discovery service) or based on a static cluster setup. I implemented both methods based on [Chef roles](https://docs.getchef.com/essentials_roles.html). Proxy settings can be set based on [Chef environments](https://docs.getchef.com/essentials_environments.html).

## Etcd recipe

Etcd releases can be easily downloaded as [binary packages](https://github.com/coreos/etcd/releases) from GitHub. The server will be started using `systemd` so we need to add a `etcd.service` unit file which is implemented as template. The unit file will contain all configuration we need for the `etcd` cluster node. Storing it at `/etc/systemd/system/` forces a `systemd` daemon reload. All together we end up with the following recipe:


```
# systemd configuration reload
execute 'systemctl-daemon-reload' do
  command '/bin/systemctl --system daemon-reload'
  action :nothing
end

# the service resource to be triggered whenever the unit file changes
service "etcd" do
  action :nothing
  provider Chef::Provider::Service::Systemd
  supports :status => true, :restart => true, :reload => true
  action :nothing
  #action [:start, :enable]
end

# deployment of the release package and linking to /usr/local/bin
ark 'etcd' do
  url node[:etcd][:download_url]
  has_binaries ['etcd', 'etcdctl']
end

# the unit file including the etcd configuration
template "/etc/systemd/system/etcd.service" do
  source "etcd.service.erb"
  variables({
    :environment => node[:etcd][:config][:environment],
  })
  notifies :run, 'execute[systemctl-daemon-reload]', :immediately
  notifies :enable, 'service[etcd]', :immediately
  notifies :restart, 'service[etcd]', :immediately
end

```

The code above demonstrates the main aspects of the `etcd` recipes. Full recipe can be found [here](https://github.com/odise/fleet-cluster-cookbook). The unit file for `etcd` containing the daemons configuration depends on the role set. The `discovery` role adds additional attributes to `default["etcd"]["config"]["environment"]` that configures the server to start using the discovery mode. This will create a file like this:

```
[Unit]
Description=etcd

[Service]
Environment=ETCD_DISCOVERY=https://discovery.etcd.io/ad716088a794a4f6a904e7c4eb1786b0
Environment=ETCD_DATA_DIR=/var/lib/etcd/
Environment=ETCD_NAME=infra0
Environment=ETCD_INITIAL_ADVERTISE_PEER_URLS=http://192.168.33.9:2380
Environment=ETCD_LISTEN_CLIENT_URLS=http://192.168.33.9:2379,http://192.168.33.9:4001,http://127.0.0.1:4001
Environment=ETCD_LISTEN_PEER_URLS=http://192.168.33.9:2380,http://192.168.33.9:7001
Environment=ETCD_ADVERTISE_CLIENT_URLS=http://192.168.33.9:2380,http://192.168.33.9:7001
ExecStart=/usr/local/bin/etcd

[Install]
WantedBy=multi-user.target
```

One more thing about the `discovery.etcd.io`: This is a CoreOS service making it easy to initially setting up a `etcd` cluster. However it has limitations like the load balancer in front of it which is cutting connections after a while (you get a 504 in case not all nodes appearing in time - check this [issue](https://github.com/coreos/etcd/issues/1292)).

## Fleet recipe

Fleet will be deployed using the same procedure as `etcd` above. There are two more things for `fleet`:

1. In order to use `fleetctl` efficiently every cluster node needs to be able to connect to each other using SSH based on authentication keys. In order to do this I implemented SSH key distribution using [Chef data bags](https://docs.getchef.com/essentials_data_bags.html).
2. For some reason `fleetctl` relies on the existence of a `core` user. It took me a while to find out why it wasn't able to connect to other cluster nodes throwing the following error: *Failed building SSH client: ssh: handshake failed: ssh: unable to authenticate, attempted methods [none publickey], no supported methods remain*. But in the end the answer was [in the code](https://github.com/coreos/fleet/blob/a7a4845662d76a8ca31d58dcc7fb68554055405a/fleetctl/ssh.go#L107).

Creating the `core` user and distributing ssh keys can be found in the `ssh` [recipe](https://github.com/odise/fleet-cluster-cookbook/blob/master/recipes/ssh.rb).

## Docker recipe

This one is easy: just using the [docker-cookbook](https://supermarket.getchef.com/cookbooks/docker).

## Test Service

To proof the cluster is working a [example unit](https://github.com/odise/fleet-cluster-cookbook/blob/master/recipes/test_service.rb) file will be deployed to every node. Once everything is set we can start the container to see the cluster working. Attention: simple example!

# And Action ...

Now lets start the two boxes and give it a try. Clone the repository and start the two vagrant boxes:

```
$ kitchen converge
```
Log in to one of them and check the status of the cluster.

```
$ kitchen login default
$ systemctl --failed
```

You should see no failed services here. Now for the cluster:

```
$ fleetctl list-machines -l
MACHINE					IP		METADATA
2993aec374654ef39d8b408f74716c0e	192.168.33.10 az=eu-central-1b,region=eu-central-1
37568b8d74794355b936643f790b070a	192.168.33.9  az=eu-central-1a,region=eu-central-1
```

We see two machines working as a fleet cluster. Lets have a look at etcd:

```
$ etcdctl member list
53d215869e907e0c: name=infra1 peerURLs=http://192.168.33.10:2380 clientURLs=http://192.168.33.10:2380,http://192.168.33.10:7001
a7e311a6b89e188f: name=infra0 peerURLs=http://192.168.33.9:2380 clientURLs=http://192.168.33.9:2380,http://192.168.33.9:7001
```

Some more details:

```
$ curl http://192.168.33.9:2379/v2/stats/leader
{
    "leader": "a7e311a6b89e188f",
    "followers": {
        "53d215869e907e0c": {
            "latency": {
                "current": 0.441689,
                "average": 84.62743093358489,
                "standardDeviation": 1296.8138483894625,
                "minimum": 0.197766,
                "maximum": 29738.208068
            },
            "counts": {
                "fail": 216,
                "success": 6068
            }
        }
    }
}
```

Now lets start the test service. `fleetctl` is using `ssh-agent` to communicate to the other nodes. We distributed the SSH key earlier so lets use it.


```
$ sudo -i -u core
$ exec /usr/bin/ssh-agent $SHELL
$ ssh-add
Identity added: /home/core/.ssh/id_rsa (/home/core/.ssh/id_rsa)
$ /usr/local/bin/fleetctl start /root/test.service
Unit test.service launched on 2993aec3.../192.168.33.10
```

The example service is deployed on the other box. Checking the status of the container can be done with `fleetctl`. Furthermore it is possible to print the journal of the unit running on a remote node.


```
$ /usr/local/bin/fleetctl status test.service
The authenticity of host '192.168.33.10' can't be established.
ECDSA key fingerprint is 14:1a:d6:19:4f:ad:89:fd:99:37:0c:f7:60:1e:ec:de.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.33.10' (ECDSA) to the list of known hosts.
test.service - MyTestApp
   Loaded: loaded (/run/fleet/units/test.service; linked-runtime)
   Active: active (running) since Tue 2014-11-04 11:40:22 EST; 1min 25s ago
  Process: 4485 ExecStartPre=/usr/bin/docker pull busybox (code=exited, status=0/SUCCESS)
  Process: 4478 ExecStartPre=/usr/bin/docker rm busybox1 (code=exited, status=1/FAILURE)
  Process: 4468 ExecStartPre=/usr/bin/docker kill busybox1 (code=exited, status=1/FAILURE)
 Main PID: 4639 (docker)
   CGroup: /system.slice/test.service
           └─4639 /usr/bin/docker run --name busybox1 busybox /bin/sh -c while true; do echo Hello World; sleep 1; done

Nov 04 11:41:37 peer-centos7.vagrantup.com docker[4639]: Hello World
Nov 04 11:41:38 peer-centos7.vagrantup.com docker[4639]: Hello World
Nov 04 11:41:39 peer-centos7.vagrantup.com docker[4639]: Hello World
Nov 04 11:41:40 peer-centos7.vagrantup.com docker[4639]: Hello World
Nov 04 11:41:41 peer-centos7.vagrantup.com docker[4639]: Hello World
Nov 04 11:41:42 peer-centos7.vagrantup.com docker[4639]: Hello World
Nov 04 11:41:43 peer-centos7.vagrantup.com docker[4639]: Hello World
Nov 04 11:41:44 peer-centos7.vagrantup.com docker[4639]: Hello World
Nov 04 11:41:45 peer-centos7.vagrantup.com docker[4639]: Hello World
Nov 04 11:41:46 peer-centos7.vagrantup.com docker[4639]: Hello World
```

Everything looks nice. Mission accomplished.

# What's next?

Running a more complex piece of software is clearly the next step. Elasticsearch is a pretty good candidate. The `fleet` metadata capability can be used and there is no need for dynamic scaling (horizontally) in the first place.

Maybe in the near future we can waste all this code and use CoreOS directly. [Working as a consultant](//kreuzwerker.de/en/devops/) you always depend on the customers speed. So lets concentrate on things that can be reused. E.g. organizing `systemd` unit files in a meaningful manner because this is the next challenge if you want to stay with `fleet`.
