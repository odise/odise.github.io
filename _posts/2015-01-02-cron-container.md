---
layout: post
category: 
tagline: "Handling periodical tasks with Docker containers"
tags : [Docker, Elasticsearch, Etcd, cronjob]
---
{% include JB/setup %}

Maintaining an [Elasticsearch](http://www.elasticsearch.org/) cluster with the help of Docker containers is a challenging task. One might say it is relatively easy to cage Elasticsearch itself inside a container. It is true: there are [ready to use Dockerfiles](https://registry.hub.docker.com/u/dockerfile/elasticsearch/) available which can be extended or used as is. However managing hardware resources depending on the load or adding and removing Elasticsearch nodes can be tricky. Lets say we gathered some experiences with our ES cluster and at a certain point in time everybody agrees on the fact that regular backups of all indices is a must. 

Honestly Elasticsearch offers a variety of options to backup your data. You can play around with data only nodes or take filesystem snapshots of the volume container ES is storying the indices in. This all works out and if it suites your environment you can use it - Elasticsearch tends to forgive smaller errors. One of the mechanisms ES offers integrates very nicely with Amazon Web Services: [AWS Cloud Plugin](https://github.com/elasticsearch/elasticsearch-cloud-aws#s3-repository). The plugin is using the [build-in snapshot/restore mechanism](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-snapshots.html#modules-snapshots) of Elasticsearch and stores the backup on a S3 bucket of your choice. 

For the sake of convenience (beauty sleep) we wanted to snapshotting on a regular base. A cronjob would do such a job in the non-containerized world. Configuring a cron job on the Docker host is problematic as ES containers can be moved dynamically among the Fleet cluster we are running. Putting the cronjob inside of one of the container being part of the Elasticsearch cluster doesn't feel right as well. Building a tiny Docker container that is taking snapshots on a regular bases sounds like a much better idea. Such a `cron-container` can be bound to a specific Fleet container based on metadata information to make sure only one of those will exist within the Elasticsearch cluster. 

For the [first version](https://github.com/odise/cron-container) of the `cron-container` I focused on minimal container size and short implementation time. Furthermore the solution should be flexible enough to handle other periodical tasks like an `Etcd` cluster backup (stored on S3 as well). Minimal size can be achieved by using a [Busybox base image](https://github.com/progrium/busybox) and standalone, statically liked tools based on [Golang](http://golang.org/). 

What I find important is the possibility to monitor the state of the last run (e.g. with [Datadog](https://www.datadoghq.com/) or [Sensu](http://sensuapp.org/)). Thats why I added some extra logic to make this available via HTTP. By firing a GET request on the container port 18080 one gets the state of the last finished run and currently running process (if any) in return. Here is an example that return HTTP 503 with the following JSON response:

	{
		Running: { },
		Last: {
			Exit_status: 2,
			Stdout: "... Checking the repository ... ... Taking snapshot ... ... Couldn't create the snapshot on repository es_backup_s3_repo at http://192.168.59.103:9200 ",
			Stderr: "{ "error" : "SnapshotCreationException[[es_backup_s3_repo:2015-01-03-13-36-30_my_fancy_production_snapshot] failed to create snapshot]; nested: IOException[Failed to get s]napshot-2015-01-03-13-36-30_my_fancy_production_snapshot]]; nested: AmazonS3Exception[Access Denied (Service: Amazon S3; Status Code: 403; Error Code: AccessDenied; Request ID: XYZ)]; ", "status" : 500 } ",
			ExitTime: "2015-01-03T13:36:30Z",
		Pid: 22,
		StartingTime: "2015-01-03T13:36:30Z"
		},
		Schedule: "0/10 * * * * *"
	}

So whats in it to cover all this points?

1. [go-cron](http://godoc.org/github.com/robfig/cron): a small wrapper around a [lightweight cron spec parser and runner](http://godoc.org/github.com/robfig/cron) containing the extra logic for monitor running jobs via a HTTP. 

2. [s3gof3r](https://github.com/rlmcpherson/s3gof3r) is a small S3 client written in Go. It is focusing on transferring large objects. Thats perfect and has a minimal footprint compared to [S3cmd](http://s3tools.org/s3cmd).

3. For the Etcd backup functionalty I added [etcd-backup](https://github.com/odise/etcd-backup). 

4. Configuration file templates are handled by [envplate](https://github.com/kreuzwerker/envplate). 

5. Elasticsearch snapshot logic as well as Etcd backups are handled via shell scripts for the time being. There is definitely room for improvements as Elasticsearch cluster handling should be done with the help of an ES library. There are a few implemented in Golang but they all missing snapshot/restore functionality. 

 What you get in the end is a small container that starts a backup task (for Elasticsearch or Etcd) and stores the result on AWS S3. The current version can be extended. Here are some things that can be added later on to make the solution more complete:
 
* Better integration with monitoring tools like Datadag. This can be done by integrating [Dogstatsd](http://docs.datadoghq.com/guides/dogstatsd/) or using the Datadog API directly.
* Better Elasticsearch cluster interaction by using an Elasticsearch library like [elasticgo](https://github.com/mattbaird/elastigo), [elastic](https://github.com/olivere/elastic) or [goes](https://github.com/belogik/goes).
* Adding an backup data lifecycle mechanism. Depending on the task old Elasticsearch snapshots should be deleted after storing the latest snapshot or setting [object expiration](http://docs.aws.amazon.com/AmazonS3/latest/dev/object-lifecycle-mgmt.html) rules on S3 buckets.




