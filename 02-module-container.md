# WIP !!!

Please note this document lives in a [WIP PR](https://github.com/luebken/container-patterns/pull/1)

# Module container

Developing container based applications is still a fairly new topic. This document tries to gather some best practices and suggests some new ideas from the community. These should be container runtime agnostic but still practical relevant with concrete examples. 

> Note: Many of these ideas are formalised in the [Open Containers Spec](https://github.com/opencontainers/specs) but we want to give guidance for todays tool chain.

> Note: This document is highly Work-In-Progress. Please get involved. Comment discuss and add your own ideas.  

## Definition

The term "module container" builds upon "application container" coined by Docker. An application container focuses on running a single process in contrast to multiple processes per container. If the application consists of multiple processes they are spread out to different containers. A module container refines application container with the focuses on being a good building block. In addition it suggests an even smaller granularity.

## Related work

In this field there is much prior and contemporary art.

* [The 12 Factor App](http://12factor.net/) a widely cited site describing how to write good PaaS applications. Initiated by Heroku. Many of these principles apply for containers as well.

* Cloud native: In the past year the term "cloud native applications" has gained popularity. A good introduction can be found in the free ebook [Migrating to Cloud Native Application Architectures](http://pivotal.io/platform/migrating-to-cloud-native-application-architectures-ebook)

[//]: # (TODO include cluster aware images once a post is up)


## Properties of a module container

There is no clear definition of a "module container". Instead we gathered  a set of guiding properties and associated best practices:

1. [Proper Linux process](#1-proper-linux-process)
2. [Container API](#2-container-api)
3. [Explicit interfaces](#3-explicit-interfaces)
4. [Disposable](#4-disposable)
5. [Immutable](#5-immutable)
6. [Self-Contained](#6-self-contained)
7. [Small](#small)

### 1. Proper Linux process

Before we come up with to many new ideas we should acknowledge the fact that a container is foremost a Linux process. Therefor we should apply common standards and best practices for writing Unix tools which happen to be containers.

#### React to signals
A container should react to [signals](https://en.wikipedia.org/wiki/Unix_signal) which are being send to it. This starts with that we don't daemonize our process and keep it in the foreground. We catch signals in the program and react appropriate. 

**Use the exec form**:  
For Docker prefer the [exec form](https://docs.docker.com/engine/reference/builder/#run) which doesn't invoke a shell.

**Catch signals**:  
Important signals to catch are:

* `SIGINT`: E.g. send by Ctrl-C to interrupt / stop a running container.  
* `SIGTERM`: Signals process termination e.g. send by `docker stop`. If necessary the container can do some final steps. After an usual grace period of 10 seconds the container is killed if it hasn't returned anymore

**Example:**  
Catching `SIGTERM` in Node.JS:  

```javascript
process.on('SIGTERM', function () {
  console.log("Received SIGTERM. Exiting.")
  server.close(function () {
    process.exit(0);
  });
});
```
https://github.com/luebken/currentweather/blob/master/server.js#L47


**Further reading:**  

* The [man page](http://man7.org/linux/man-pages/man7/signal.7.html) contains a good overview of signals.
* [What makes an awesome CLI Application](https://pragprog.com/magazines/2012-05/what-makes-an-awesome-commandline-application) give some inspiration.
* [Signal handlers must be reentrant](http://blog.rubybestpractices.com/posts/ewong/016-Implementing-Signal-Handlers.html#fn1) What happens when another signal arrives while this handler is still running?
* [Self pipe trick](http://cr.yp.to/docs/selfpipe.html) Maintain a pipe for signals. 

#### Return proper exit codes
We should return proper exit codes when exiting the container. This gives us a better overview of what happened and the scheduler / init process better means of scheduling. E.g. in Kubernetes you can define that only failed containers [should be restarted](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/pod-states.md#restartpolicy).

We generally just differ between the exit code `0` as a successful termination and something `>0` as a failure. But other exit codes are conceivable. Some inspiration might give [glibc](https://github.molgen.mpg.de/git-mirror/glibc/blob/master/misc/sysexits.h) or [bash](http://tldp.org/LDP/abs/html/exitcodes.html).

**Example:**  
Return a non-failure exit code in Node.JS:

```javascript
process.exit(0);
```
https://github.com/luebken/currentweather/blob/master/server.js#L50

[//]: # (TODO better example with exit code 1)

#### Use standard streams

Linux processes use standard streams as a means of communication. There are `stdin`: standard input, `stdout`: standard output and `stderr` standard error:

* `stdout`: For all logging activities use stdout let the infrastructure take care of handling this stream or forwarding it to some log aggregator. 

[//]: # (TODO point to log side-car and write an example)

* `stdin`: If our container can be be conceived as a Unix tool we should accept data from stdin. This would allow [piping between containers](http://matthewkwilliams.com/index.php/2015/04/21/piping-hot-docker-containers/).

Tips / Further reading:

* If your app writes to a file link that file to the device file:

  ```Dockerfile
  RUN ln -sf /dev/stdout /var/log/nginx/access.log
  ```
  https://github.com/nginxinc/docker-nginx/blob/master/stable/jessie/Dockerfile#L14

* In 12 factor apps: [Treat logs as event streams](http://12factor.net/logs).

#### Handle Arguments

Arguments are a straight forward way to configure and send options to a container. We propose that you should handle these in a comprehensible way.

Luckily CLI arguments are a very old topic so we refer to existing standards and libraries:

* [POSIX.1-2008 Utility conventions](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html)
* In conjunction with [getopt](http://pubs.opengroup.org/onlinepubs/9699919799/functions/getopt.html) as utility to parse / validate.
* Libcs [Program Argument Syntax Conventions](http://www.gnu.org/software/libc/manual/html_node/Argument-Syntax.html)


### 2. Container API

Although our container is foremost a Linux process it is also contained in it's own environment. This gives our "module container" more capabilities in defining APIs to it's environment and clients.

#### Use Environment Variables

In addition or as an alternative to command line arguments we can use environment variables to inject information into our container. Especially for configuration this has been made popular by the 12 Factor App: [III. Config Store config in the environment](Store config in the environment)

A common best practice is to set the [defaults envs](https://docs.docker.com/engine/reference/builder/#env) in the image and let the user overwrite it [](https://docs.docker.com/engine/reference/run/#env-environment-variables). See the section with labels below as an idea how to document these. 

#### Declare Available Ports

Another interface are the network ports our container potentially listens on. With the [EXPOSE](https://docs.docker.com/engine/reference/builder/#expose) directive. The ports will then show up in `docker ps` and `docker inspect`. And they can be [enforced](https://docs.docker.com/engine/userguide/networking/default_network/container-communication/#communication-between-containers) with setting `ipables=true` && `icc=false`.

#### Volume mounts

In Docker you can define volumes when starting the container or when defining the image. We advise to declare volumes in the image so it can be examined before hand.


#### Hooks

Sometimes a container needs to react to different events during it's life time. Before termination we can check for the termination signal but that is limited since no context information can be given. In addition we might want to react to more events like something like an event post startup.

A good example are [container hooks in Kubernetes](http://kubernetes.io/v1.1/docs/user-guide/container-environment.html#container-hooks). Another  example are the [hooks in the opencontainer spec](https://github.com/opencontainers/specs/blob/master/config.md#hooks). 

Docker currently doesn't natively support hooks but there is a proposal about adding them: [#6982](https://github.com/docker/docker/issues/6982).

Docker does submit [events](https://docs.docker.com/engine/reference/commandline/events/) which can be leveraged to implement non-blocking hooks. Jeff Lindsay happened to implement exactly this with [dockerhook](https://github.com/progrium/dockerhook).		

[//]: # (TODO write a proposal for adding this as a label)
[//]: # (TODO look at native docker plugins )
[//]: # (TODO maybe add https://github.com/progrium/entrykit#prehook---run-pre-commands-on-start )


### 3. Explicit interfaces

A good module has an explicit or as wikipedia says [well defined](https://en.wikipedia.org/wiki/Modular_programming#Key_aspects) interface. With all the ways of creating an API for our container we also need a way to expose this. Which is what we do with the EXPOSE or the VOLUME declaration. We want to expand on that and use Labels to document more of our API.


#### Document With Labels

Docker has the ability to add arbitrary metadata to images via [labels](https://docs.docker.com/engine/userguide/labels-custom-metadata/) which can be even [JSON](https://docs.docker.com/engine/userguide/labels-custom-metadata/#store-structured-data-in-labels).

Gareth Rushgrove is evangelizing the idea of creating standards on this data and creating ["shipping manifests"](https://speakerdeck.com/garethr/shipping-manifests-bill-of-lading-and-docker-metadata-and-container
). He has even written a tool that validates the schema of this data: [docker-label-inspector](https://github.com/garethr/docker-label-inspector). See his Dockercon [presentation](https://www.youtube.com/watch?v=j4SZ1qoR8Hs) and [slides](https://speakerdeck.com/garethr/shipping-manifests-bill-of-lading-and-docker-metadata-and-container) for details.

There is also an initiative on [standard labels](https://github.com/projectatomic/ContainerApplicationGenericLabels).

As an example we want to document an environment variable that our container needs:

```json
{
  "key": "OPENWEATHERMAP_APIKEY",
  "description": "APIKEY to access the OpenWeatherMap. Get one at tp://openweathermap.org/appid",
  "mandatory": true
}
```
https://github.com/luebken/currentweather/blob/master/Dockerfile#L13

To inspect the labels use:

```
$ docker inspect -f "{{json .Config.Labels }}" <container image>
```
https://github.com/luebken/currentweather/blob/master/Makefile#L9

#### View the API contract of a module container

https://github.com/luebken/container-api

[//]: # (TODO comment on Feature request: https://github.com/docker/docker/issues/20360)

### 4. Disposable

The [Pets vs Cattle](https://blog.engineyard.com/2014/pets-vs-cattle) is the infamous article about an analogy which differ two different server types. There are pets which you give names and want to hold on and catles which you give numbers and can be exchanged easily.

A module container should always strive for being exchanged with a copy at any point of time. This especially true in an cluster environment where there can be many reasons for a particular container be stopped:
 
 * Rescheduling because of limit or bad resources
 * Down-scaling
 * Errors within the container
 * Migration to new hardware / locality of services

This concept widely accepted in the container space that developers use the `--rm` with Docker as a default which always remove the container after they have stopped. We have http://12factor.net/disposabilitychosen the term "disposable" from the [12Factor app](http://12factor.net/disposability).

[//]: # (Most argued property so far. Would need to put some  Ops perspective on this.) 

#### Best practices

* Be robust against sudden death.  
  If the container gets interrupted pass on your current job. (See ["React to signals"](#react-to-signals))
* Minimal setup  
  If more setup needed let the scheduler know and use [hooks](#hooks).

### 5. Immutable

The container image contains the OS, libraries, configurations, files and application code. Once a container image is built it shouldn’t be changed especially not between different staging environtments like dev, qa and production. State should be extracted and changes to the container should be applied by rebuilding the container.

#### Best practices
* Have a [dev / prod parity](http://12factor.net/dev-prod-parity) with the container image
* Externalize configuration with defaults in the image
[//]: # (TODO move down?)
* Extract runtime state in volumes
* Anti-pattern to go into the container and change configuration. Risk of a [SnowflakeServer](http://martinfowler.com/bliki/SnowflakeServer.html)
* Create a final file layout on build

### 6. Self-Contained
The container should only rely on the Linux kernel. All other dependencies should be made explicit and added dynamically.

#### Best practices
* Add dependencies at build time
  * E.g. Build Uber-Jar and include webserver
* Zero-config deployment. Use sensible defaults.
* Generate dynamic config files on the fly.

[//]: # (TODO find good example)
[//]: # (TODO https://github.com/markround/tiller)
[//]: # (TODO https://github.com/kelseyhightower/confd)
[//]: # (TODO https://github.com/joyent/containerbuddy)
[//]: # (TODO https://github.com/progrium/entrykit#render---template-rendering)

#### Best practices
* Anti-Patterns: 
  * Put config into a volume
  * Put code into a volume.  
    The exception might be a development environment.


### 7. Small
A container should have the least amount of code possible to fulfil its job. The smaller, the less dependencies, the less interference. Easier to redristibute. Better bug hunting, auditing. Smaller interface generally means better security.

Alpine is a minimalist linux distribution based on busybox, musl-libc, a new package manager called apk (not the Android one) and OpenRC as init system. [Some Thoughts on the Use of Alpine Linux in Docker Images](http://www.skippbox.com/thoughts-on-the-use-of-alpine-linux-in-docker-images/).


See image building guidelines like:

* [Open Shift Guideline](https://docs.openshift.org/latest/creating_images/guidelines.html) 
* [Docker best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/)
* [Project Atomic Guidance for Docker Image Authors](http://www.projectatomic.io/docs/docker-image-author-guidance/)

#### Best practices
* Build from scratch 
* Use small base-image
  alpine is the shiny new kid in town
* Reuse custom base image
Anti-Pattern: VM Container