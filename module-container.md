# Module container

Developing container based applications is still a fairly new topic. Even more are general ideas such as using containers as modules to architect applictions. To assit further discussions this document tries to gather current best practices and tries to suggests some new ideas to the community. 

These ideas should be container runtime and orchestration engine agnostic. And at the same time still practical with relevant and concrete examples.

Many of these ideas are formalised in the [Open Containers Spec](https://github.com/opencontainers/specs) but we want to give guidance for tools in use today. We will try to link to Docker, rkt and OCI examples where applicable.

> Note: This document is highly Work-In-Progress. Please get involved. Comment discuss and add your own ideas.  

## Definition

The term "module container" builds upon the term "application container" coined by Docker. An application container focuses on running a single process in contrast to multiple processes per container. If the application consists of multiple processes they may be spread out to different containers. A module container refines an application container and focuses on being a modular isolated building block in the provision of a service. In addition it suggests an even smaller level of granularity.

[//]: # (TODO mention system container)

## Related work

In this field there is much prior and contemporary art:

* [The 12 Factor App](http://12factor.net/) a widely cited site describing how to write good PaaS applications. Initiated by Heroku. Many of these principles apply for containers as well.

* Cloud native: In the past year the term "cloud native applications" has gained popularity. A good introduction can be found in the free ebook [Migrating to Cloud Native Application Architectures](http://pivotal.io/platform/migrating-to-cloud-native-application-architectures-ebook)

The folks from [Deis](http://deis.io/) have coined the term [cluster aware image](https://twitter.com/gabrtv/status/676218070357032961).

## Properties of a module container

There is no canonical definition of a "module container". Instead we gather a set of guiding properties and associated best practices:

1. [Linux process](1_linux_process.md)
2. [Container API](#2-container-api)
3. [Self-Describing](#3-self-describing)
4. [Disposable](#4-disposable)
5. [Immutable](#5-immutable)
6. [Self-Contained](#6-self-contained)
7. [Small](#small)




### 3. Self-Describing

A good module has an explicit, or as wikipedia says [well defined](https://en.wikipedia.org/wiki/Modular_programming#Key_aspects) interface. So with all the ways declaring an API for our container we also need a way to document this. Unfortunatel, as this is not always possible within the declaration; we want to expand on the definition and use labels to document our API.

#### Document With Labels

We like to propose to document the API with labels, as all container engines have the means of doing so.

Docker has the ability to add arbitrary metadata to images via [labels](https://docs.docker.com/engine/userguide/labels-custom-metadata/) which can be even [JSON](https://docs.docker.com/engine/userguide/labels-custom-metadata/#store-structured-data-in-labels).

Gareth Rushgrove has been evangelising the idea of creating standards on this data and creating ["shipping manifests"](https://speakerdeck.com/garethr/shipping-manifests-bill-of-lading-and-docker-metadata-and-container
). He has even written a tool that validates the schema of this data: [docker-label-inspector](https://github.com/garethr/docker-label-inspector). See his Dockercon [presentation](https://www.youtube.com/watch?v=j4SZ1qoR8Hs) and [slides](https://speakerdeck.com/garethr/shipping-manifests-bill-of-lading-and-docker-metadata-and-container) for details.

From Red Hat there is an initiative on [standardizing certain labels](https://github.com/projectatomic/ContainerApplicationGenericLabels).

Example:

As a simple example we want to document that our container expects a certain environment variable. Here a suggestion using the prefix `api.ENV`:

```
LABEL api.ENV.OPENWEATHERMAP_APIKEY="" \
      api.ENV.OPENWEATHERMAP_APIKEY.description="Access key for OpenWeatherMap. See http://openweathermap.org/appid for details." \
      api.ENV.OPENWEATHERMAP_APIKEY.mandatory="false"
```
See https://github.com/luebken/currentweather/blob/master/Dockerfile

We can than use tools like `docker inspect` to read these labels:

```
$ docker inspect -f "{{json .Config.Labels }}" <container image>
```

#### View the API contract of a module container

If you take this a step further and agree on some standard labels and schemas you could write tools that introspect the API contract of a container. e.g.


```
$ container-api luebken/currentweather-nodejs
 -----------------------------------------------------------
| Image:  luebken/currentweather-nodejs
|-----------------------------------------------------------
| Author:   Matthias Luebken, matthias@luebken.com
| Size:     158 MB
| Created:  2016-03-01 15:59
|-----------------------------------------------------------
| Container API:
| * Mandatory ENVs to configure:
|   - ENV:           OPENWEATHERMAP_APIKEY
|   - Description:   APIKEY to access the OpenWeatherMap. Get one at http://openweathermap.org/appid
|   - Mandatory:     true
| * Optional ENVs to configure:
|     - < empty >
| * Available ports:
|     -  1337/tcp {}
| * Volumes:
 -----------------------------------------------------------
```

For more info see: https://github.com/luebken/container-api


### 4. Disposable

The [Pets vs Cattle](https://blog.engineyard.com/2014/pets-vs-cattle) is the infamous article about an analogy which differs two different server types. There are pets which you give names and want to hold on and cattle which you give numbers and can be exchanged easily.

A module container should always strive to be able to be exchanged with a fresh instance at any point of time. This is especially true in a cluster environment, where there are many reasons that a particular container can be stopped:
 
 * Rescheduling because of limit or bad resources
 * Down-scaling
 * Errors within the container
 * Migration to new hardware / locality of services

This concept is so widely accepted in the container space that developers use the `--rm` with Docker as a default which always removes the container after it has stopped. We have chosen the term "disposable" from the [12Factor app](http://12factor.net/disposability).

**Best practices on being dispoable:**

* Be robust against sudden death.  
  If the container gets interrupted, pass your current job on to another instance of possible. (See ["React to signals"](#react-to-signals))
* Minimal setup  
  If more setup needed let the scheduler know and use [hooks](#hooks).

### 5. Immutable

The container image contains the OS, libraries, configurations, files and application code. Once a container image is built it shouldn’t be changed, especially not between different staging environments like dev, qa and production. State should be extracted and changes to the container should be applied by rebuilding the container.

**Best practices on being immutable:**
* Have a [dev / prod parity](http://12factor.net/dev-prod-parity) with the container image
* Extract runtime state in volumes
* Create a final file layout on build
* And don't go into the container and change configuration with the risk of creating a [SnowflakeServer](http://martinfowler.com/bliki/SnowflakeServer.html)

### 6. Self-Contained
The container should only rely on the Linux kernel. All dependencies should be added at build time. E.g. In the Java world build an Uber-Jar which includes a webserver.

Sometimes developers put application code into a volume to cut the container build time. If you do this, please ensure that you do it only for yourself. And remember to include all application code once the container leaves your computer.

The exception to this rule might sensible data like secretes or god forbid passwords. A detailed writeup about those will follow with [issue 7](https://github.com/luebken/container-patterns/issues/7).

#### Configuration
If your container relies on configuration files generate them on the fly. Also ensure that you use sensible defaults for parameters which enables simple zero-config deployments.

There are several tools out there to help generate config files. E.g. [progrium/entrykit](https://github.com/progrium/entrykit#render---template-rendering) or [kelseyhightower/confd](https://github.com/kelseyhightower/confd). But also a simple shell script might do the job e.g.:

```
#!/bin/sh
set -e
datadir=${APP_DATADIR:="/var/lib/data"}
host=${APP_HOST:="127.0.0.1"}
port=${APP_PORT:="3306"}
cat <<EOF > /etc/config.json
{
  "datadir": "${datadir}",
  "host": "${host}",
  "port": "${port}",
}
EOF
mkdir -p ${APP_DATADIR}
exec "/app"
```
Which is blatantly copied from the great blogpost by Kelsey on [12 Fractured Apps](https://medium.com/@kelseyhightower/12-fractured-apps-1080c73d481c#.g3ydhs5zw).


[//]: # (TODO write something about config files http://www.mricho.com/confd-and-docker-separating-config-and-code-for-containers/)
[//]: # (TODO https://twitter.com/kelseyhightower/status/657761769570594816)


#### Further reading on configuration

* Configuration of applications running in containers: https://github.com/joyent/containerbuddy
* A dynamic configuration file generation tool: https://github.com/markround/tiller


### 7. Small
A container should have the minimum amount of code / libraries as possible to fulfil it's single purpose.

Reasons for a smaller image:

  * Faster in the network (deploy, reschedule, update)
  * Increased I/O performance
  * Smaller attack surface. Easier to audit.

Many containers are based of `debian/buildessentials` which is often unnecessary for runtime. Use different containers for building and running.

You don't have to use `Dockerfile`. Maybe creating a tar with something like [buildroot](https://buildroot.org/) and importing it via `docker import`. See the talk from Redbeard: [Best Practices For Containerized Environments](https://www.youtube.com/watch?v=gMpldbcMHuI).

Also have a look at Alpine. A minimalist Linux distribution based on busybox, musl-libc, a new package manager called apk (not the Android one) and OpenRC as init system. [Some Thoughts on the Use of Alpine Linux in Docker Images](http://www.skippbox.com/thoughts-on-the-use-of-alpine-linux-in-docker-images/).

#### Further reading on image building guidelines

* [OpenShift Guideline](https://docs.openshift.org/latest/creating_images/guidelines.html) 
* [Docker best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/)
* [Project Atomic Guidance for Docker Image Authors](http://www.projectatomic.io/docs/docker-image-author-guidance/)
