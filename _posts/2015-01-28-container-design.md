---
layout: post
title: "State of the art web-application deployments"
tagline: "from a consultant perspective"
category: 
tags: [ AWS, Docker ]
---
{% include JB/setup %}

The world of web-application deployments has changed significantly over the last year. Containerization of applications has reached production at scale. Projects that started as Docker-prototype survived a couple of updates and handling deployments based on containers feels nearly normal. However the market of tools and services for container orchestration is still moving. There is still no killer application to handle your container stack - at least I couldn't find it so far. Is it still worth searching? Is the stack your container ecosystem is running at future proof or at least stable enough for the next year? Nobody knows. For my daily work I found some solid patterns and best practices keeping the projects alive I taking care of. 

# Container design matters

Packaging applications into containers lays the foundation of the container environment a DevOp engineer (formerly known as system administrator resp. operations engineer) will handle as a day-by-day job. This basically means that the container handling (deployment, update and monitoring) massively depends on the decisions you take. Here is a list of things I find important:

## Container size

Size matters for two reasons: build time and deployment/update time depend on container size. Of cause it is difficult to define a meaningful threshold for that. The size of the container is bound to the importance of the application that it contains and on the expected number of update cycles. Two [Fat Jar](http://www.javacodegeeks.com/2012/11/packing-your-java-application-as-one-or-fat-jar.html) application examples:

1. A backend system that processes data from other backends and won't change offen due to the fact that it isn't under active development.
2. A Tomcat application that changes very frequently. More exactly the Tomcat container WAR file will be released very frequently.

There is not so much you can do about the first application beside providing a minimal base image to build your container. Java is a good example as the JVM doesn't need to much to run. 

**XXX: JAVA base image comparison**

The Tomcat container adds the update size that should be minimized. So beside a minimal base image containing the JVM stack, the WAR file update can cause a certain delay during the deployment. In order to minimize the version delta I would split the application into JVM/Tomcat base container and a second one only providing the WAR file. The later can be linked into the first. The point I'm trying to make here is: check wether it is worth putting effort into the container size. For a bunch of cases it is but there are always components that give only little room for optimization.

## Running one application per container

Putting only one application per container differentiates the cgroups container concept from being a micro VM approach. If you containerize an application from scratch this can turn out to be complicated. However there are a couple of good reasons to do so. 

Starting an application as a blocking process and redirecting log output to STDOUT give you mainly two things:

1. Process monitoring: the container dies when your application dies. Only one component will be effected (leaving the collateral damage aside). 
2. `docker logs <containerID>` provides you with a dedicated view on this application without handling log files on the host system.








Not so long ago using the [phusion/baseimage-docker](https://github.com/phusion/baseimage-docker) base image was best pratice when developing Docker containers. Thanks to [runit](http://smarden.org/runit/) it was possible to start (and restart) multiple applications inside one container. SSH opened the container for shell access which was painful before `docker exec` appeared in Docker 1.3. 



