---
layout: post
title: "Fleet cluster recovery"
description: "How do you restore a broken Fleet cluster?"
category: 
tags: [fleet, etcd, docker]
---
{% include JB/setup %}

I've played around quite a while with Etcd now and it turned out to be essential to backup your data frequently. This guide describes how I usually recover crashed Fleet cluster without following the [lately documented way](https://github.com/coreos/etcd/blob/master/Documentation/2.0/admin_guide.md#disaster-recovery) of Etcd backup.

# Assumption

There is (or was) an Etcd cluster running on more then one node which for some reason is not operating as expected anymore. To bring the cluster back to life one needs at least one running node with valid data (to be recovered) or a valid Etcd backup. Furthermore the procedure only makes sense if the docker container Fleet has been managing before are still (part wise) running.
Creating a Etcd backup
Fleet is storing all necessary data in the hidden directory `/_coreos.com`. In order to to list the data directory with `etcdctl` you need to fire the following command:

    $ etcdctl ls /_coreos.com --recursive

To make a dump of the data stored on the Etcd node use the tool [etc-backup](https://github.com/odise/etcd-backup). The tool needs two configuration files (`etcd-configuration.json`, `backup-configuration.json`). Here is an example for the production environment:

etcd-configuration.json

    {
      "cluster": {
          "leader": "http://phost-dh01.test:2379",
            "machines": [
                "http://phost-dh01.test:2379",
                "http://phost-dh02.test:2379",
                "http://phost-dh03.test:2379"
              ]
        },
      "config": {
          "certFile": "",
          "keyFile": "",
          "caCertFiles": [],
          "timeout": 10000000000000,
          "consistency": "STRONG"
        }
    }

backup-configuration.json

    {
      "concurrentRequests": 50,
      "retries": 5,
      "dumpFilePath": "dump.json",
      "backupStrategy": {
        "keys": ["/_coreos.com"],
        "sorted": false,
        "recursive": true
      }
    }

Creating a backup by hand can be done the following way:

    $ etcd-backup dump

Make sure you have the two config file (see above) in the same directory. The dump will be created as file dump.json.  Store it on safe place.

# Recreate the Etcd cluster

Once the configuration of Fleet has been saved as a backup the existing Etcd cluster can be setup from scratch. It is also possible to repair the cluster by hand adding and/or removing nodes, however the current state of the implementation (0.5.0-alpha4) is not very stable. 

To operate on different VMs at the same time use csshx on OSX (can be installed with brew). Connect to the hosts the Etcd cluster is running and do the following:


    $ systemctl stop fleet
    $ systemctl stop etcd
    $ rm -rf /data/etcd


Now fix the configuration on all nodes that are intended to joining the party. Make sure the `ETCD_INITIAL_CLUSTER_STATE=new` is set in `/etc/systemd/system/etcd.service`. Restart the Etcd server on all nodes and check that it is operating as expected (use `systemctl start etcd` and `etcdctl member list`).  **Don't start Fleet for now!**
Now restore the backup to the cluster using the dump file from the backup procedure:

    $ etcd-backup restore

You should see the restored data on every Etcd node now:

    $ etcdctl ls /_coreos.com/fleet/machines
    /_coreos.com/fleet/machines/3f058f7c80d348fb9a01595878404277
    /_coreos.com/fleet/machines/8a235a5e09e54223a79c287655fb88f8
    /_coreos.com/fleet/machines/91e82cd59c2d4f3aac0d37203b2e17bb

Now start Fleet node by node. Don't start all at the same time! Check the following after each fleet node started:

    $ # you should see each started node here:
    $ fleetctl list-machines
    MACHINE		IP		METADATA
    2df4dc82...	10.201.225.25	es=true,kafka=true,zk_node_id=2
    $ # check the status of the units bound to the fleet node
    $ fleetctl list-units
    UNIT			MACHINE				ACTIVE	SUB
    elasticsearch@1.service	2df4dc82.../10.201.225.25	active	running

# How to improve

* Establish automated backup of the etcd cluster content. I started this by implementing a [cron-like container](https://github.com/odise/etcd-backup-cron) which is dumping Etcd on a regular base and uploading it to AWS S3.
* Move Etcd cluster away form the docker hosts. Set up a small dedicated pool of 3 machines that run the Etcd cluster and reconfigure Fleet to use this instead of having it on the same machines as Fleet. Preferably don't touch it ;). 

