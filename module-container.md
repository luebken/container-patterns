# WIP !!!

Please note this document lives in a [WIP PR](https://github.com/luebken/container-patterns/pull/1)

# Module container

Developing container based applications is still a fairly new topic. This document tries to gather some best practices and suggests some new ideas from the community. These should be container runtime agnostic but still practically relevant with concrete examples.

> Note: Many of these ideas are formalised in the [Open Containers Spec](https://github.com/opencontainers/specs) but we want to give guidance for todays tool chain.

> Note: This document is highly Work-In-Progress. Please get involved. Comment discuss and add your own ideas.

## Definition

The term "module container" builds upon "application container" coined by Docker. An application container focuses on running a single process in contrast to multiple processes per container. If the application consists of multiple processes they are spread out to different containers. A module container refines application container with the focus on being a good building block. In addition it suggests an even smaller granularity.

## Related work

In this field there is much prior and contemporary art.

* [The 12 Factor App](http://12factor.net/) a widely cited site describing how to write good PaaS applications. Initiated by Heroku. Many of these principles apply for containers as well.

* Cloud native: In the past year the term "cloud native applications" has gained popularity. A good introduction can be found in the free ebook [Migrating to Cloud Native Application Architectures](http://pivotal.io/platform/migrating-to-cloud-native-application-architectures-ebook)

[//]: # (TODO include cluster aware images once a post is up)


## Properties of a module container

There is no clear definition of a "module container". Instead we gathered a set of guiding properties and associated best practices:

1. [Linux process](#1-linux-process)
2. [Container API](#2-container-api)
3. [Explicit interfaces](#3-explicit-interfaces)
4. [Disposable](#4-disposable)
5. [Immutable](#5-immutable)
6. [Self-Contained](#6-self-contained)
7. [Small](#small)

### 1. Linux process

Before we come up with too many new ideas we should acknowledge the fact that a container is foremost a Linux process. Therefore we should apply common standards and best practices for writing Unix tools which happen to be containers.

#### React to signals
A container should react to [signals](https://en.wikipedia.org/wiki/Unix_signal) which are sent to it. This starts with that we don't daemonize our process and keep it in the foreground. We catch signals in the program and react appropriately.

**Use the exec form**:
For Docker prefer the [exec form](https://docs.docker.com/engine/reference/builder/#run) which doesn't invoke a shell.

**Catch signals**:
Important signals to catch are:

* `SIGINT`: E.g. send by Ctrl-C to interrupt / stop a running container.
* `SIGTERM`: Signals process termination e.g. send by `docker stop`. If necessary the container can do some final steps. After a usual grace period of 10 seconds the container is killed if it still hasn't returned.

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
* [What makes an awesome CLI Application](https://pragprog.com/magazines/2012-05/what-makes-an-awesome-commandline-application) gives some inspiration.
* [Signal handlers must be reentrant](http://blog.rubybestpractices.com/posts/ewong/016-Implementing-Signal-Handlers.html#fn1) What happens when another signal arrives while this handler is still running?
* [Self pipe trick](http://cr.yp.to/docs/selfpipe.html) Maintain a pipe for signals.

#### Return exit codes
We should return proper exit codes when exiting the container. This gives us a better overview of what happened and the scheduler / init process better means of scheduling. E.g. in Kubernetes you can define that only failed containers [should be restarted](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/pod-states.md#restartpolicy).

We generally just differ between the exit code `0` as a successful termination and something `>0` as a failure. But other exit codes are conceivable. For some inspiration, check out [glibc](https://github.molgen.mpg.de/git-mirror/glibc/blob/master/misc/sysexits.h) or [bash](http://tldp.org/LDP/abs/html/exitcodes.html).

**Example:**
Return a non-failure exit code in Node.JS:

```javascript
process.exit(0);
```
https://github.com/luebken/currentweather/blob/master/server.js#L50

[//]: # (TODO better example with exit code 1)

#### Use standard streams

Linux processes use standard streams as a means of communication. There are `stdin`: standard input, `stdout`: standard output and `stderr` standard error:

* `stdout`: For all logging activities use stdout and let the infrastructure take care of handling this stream or forwarding it to some log aggregator.

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
* In conjunction with [getopt](http://pubs.opengroup.org/onlinepubs/9699919799/functions/getopt.html) as a utility to parse / validate.
* Libcs [Program Argument Syntax Conventions](http://www.gnu.org/software/libc/manual/html_node/Argument-Syntax.html)


### 2. API

Although our container is foremost a Linux process it is also contained in its own environment. This gives our "module container" more capabilities in defining APIs to its environment and clients.

#### Use Environment Variables

In addition or as an alternative to command line arguments we can use environment variables to inject information into our container. Especially for configuration this has been made popular by the 12 Factor App: [III. Config Store config in the environment](https://12factor.net/config). They are easy to change between deploys and there's no danger of checking them into a VCS.

A common best practice is to set the [defaults envs](https://docs.docker.com/engine/reference/builder/#env) in the image and [let the user overwrite it](https://docs.docker.com/engine/reference/run/#env-environment-variables). See the section with labels below as an idea how to document these.


[//]: # (TODO compare with rkt env handling https://coreos.com/rkt/docs/latest/subcommands/run.html#influencing-environment-variables)

Links:
  * A discussion about the usage of envs: https://gist.github.com/telent/9742059
  * "Parameterized Docker Containers": http://blog.james-carr.org/2013/09/04/parameterized-docker-containers/
  * A dynamic configuration file generation tool https://github.com/markround/tiller

[//]: # (TODO example with https://github.com/andreasjansson/envtpl)


[//]: # (TODO write something about secret handling)

#### Declare Available Ports

Another interface are the network ports our container potentially listens on. With the [EXPOSE](https://docs.docker.com/engine/reference/builder/#expose) directive. The ports will then show up in `docker ps` and `docker inspect`, and they can be [enforced](https://docs.docker.com/engine/userguide/networking/default_network/container-communication/#communication-between-containers) with setting `iptables=true` && `icc=false`.

#### Volume mounts

In Docker you can define volumes when starting the container or when defining the image. We advise to declare volumes in the image so it can be examined before hand.


#### Hooks

Sometimes a container needs to react to different events during it's lifetime. Before termination we can check for the termination signal but that is limited since no context information can be given. In addition we might want to react to more events like something like an event post startup.

Good examples are [container hooks in Kubernetes](http://kubernetes.io/v1.1/docs/user-guide/container-environment.html#container-hooks), and the [hooks in the opencontainer spec](https://github.com/opencontainers/specs/blob/master/config.md#hooks).

Docker currently doesn't natively support hooks but there is a proposal about adding them: [#6982](https://github.com/docker/docker/issues/6982).

Docker does submit [events](https://docs.docker.com/engine/reference/commandline/events/) which can be leveraged to implement non-blocking hooks. Jeff Lindsay happened to implement exactly this with [dockerhook](https://github.com/progrium/dockerhook).

[//]: # (TODO write a proposal for adding this as a label)
[//]: # (TODO look at native docker plugins )
[//]: # (TODO maybe add https://github.com/progrium/entrykit#prehook---run-pre-commands-on-start )


### 3. Descriptive

A good module has an explicit, or as wikipedia says [well defined](https://en.wikipedia.org/wiki/Modular_programming#Key_aspects), interface. So with all the ways of creating an API for our container we also need a way to expose this. Which is what we do with the EXPOSE or the VOLUME declaration. We want to expand on that and use labels to document more of our API.


#### Document With Labels

Docker has the ability to add arbitrary metadata to images via [labels](https://docs.docker.com/engine/userguide/labels-custom-metadata/) which can be even [JSON](https://docs.docker.com/engine/userguide/labels-custom-metadata/#store-structured-data-in-labels).

Gareth Rushgrove is evangelizing the idea of creating standards on this data and creating ["shipping manifests"](https://speakerdeck.com/garethr/shipping-manifests-bill-of-lading-and-docker-metadata-and-container
). He has even written a tool that validates the schema of this data: [docker-label-inspector](https://github.com/garethr/docker-label-inspector). See his Dockercon [presentation](https://www.youtube.com/watch?v=j4SZ1qoR8Hs) and [slides](https://speakerdeck.com/garethr/shipping-manifests-bill-of-lading-and-docker-metadata-and-container) for details.

There is also an initiative on [standard labels](https://github.com/projectatomic/ContainerApplicationGenericLabels).

As an example we want to document an environment variable that our container needs:

```json
{
  "key": "OPENWEATHERMAP_APIKEY",
  "description": "APIKEY to access the OpenWeatherMap. Get one at http://openweathermap.org/appid",
  "mandatory": true
}
```
https://github.com/luebken/currentweather/blob/master/Dockerfile#L30

To inspect the labels use:

```
$ docker inspect -f "{{json .Config.Labels }}" <container image>
```
https://github.com/luebken/currentweather/blob/master/Makefile#L9

#### View the API contract of a module container

If you take this a step futher and agree on some standard labels and schemas you could write tools that introspect the API contract of a container. e.g.


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


[//]: # (TODO think about a depends /expect label: https://github.com/docker/docker/issues/7247)
[//]: # (TODO think about a  https://github.com/docker/docker/issues/12142)
[//]: # (TODO think about Does this goes along: Dependency Injects / Service Locator patterns?)
[//]: # (TODO comment on Feature request: https://github.com/docker/docker/issues/20360)

### 4. Disposable

The [Pets vs Cattle](https://blog.engineyard.com/2014/pets-vs-cattle) is the infamous article about an analogy which compares two different server types. There are pets which you give names and want to hold on to, and cattle which you give numbers to and can be exchanged easily.

A module container should always expect to be exchanged with a copy at any point of time. This is especially true in a cluster environment where there can be many reasons for a particular container be stopped:

 * Rescheduling because of limit or bad resources
 * Down-scaling
 * Errors within the container
 * Migration to new hardware / locality of services

This concept is so widely accepted in the container space that developers use the `--rm` with Docker as a default which always remove the container after they have stopped. We have chosen the term "disposable" from the [12Factor app](http://12factor.net/disposability).

[//]: # (Most argued property so far. Would should put some Ops perspective on this which want to keep as much as possible in the case of a failure.)

#### Best practices

* Be robust against sudden death.
  If the container gets interrupted pass on your current job. (See ["React to signals"](#react-to-signals))
* Minimal setup
  If more setup needed let the scheduler know and use [hooks](#hooks).

### 5. Immutable

The container image contains the OS, libraries, configurations, files and application code. Once a container image is built it shouldn’t be changed, especially not between different staging environtments like dev, qa and production. State should be extracted and changes to the container should be applied by rebuilding the container.

#### Best practices
* Have a [dev / prod parity](http://12factor.net/dev-prod-parity) with the container image
* Extract runtime state in volumes
* Anti-pattern to go into the container and change configuration. Risk of a [SnowflakeServer](http://martinfowler.com/bliki/SnowflakeServer.html)
* Create a final file layout on build

### 6. Self-Contained
The container should only rely on the Linux kernel. All dependencies should be added at build time. E.g. Build an Uber-Jar which includes a webserver.

#### Configuration
If your container relies on configuration files generate them on the fly. For the parameters use sensible defaults so a simple zero-config deployment is possible.

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
Which is blatently copied from the great blogpost by Kelsey on [12 Fractured Apps](https://medium.com/@kelseyhightower/12-fractured-apps-1080c73d481c#.g3ydhs5zw).

Also try to put as much as possible into the container. E.g. putting the application code into a volume should only be used for dev environments. If the container is deployed as much as possible should be in the container.

[//]: # (TODO write something about config files http://www.mricho.com/confd-and-docker-separating-config-and-code-for-containers/)
[//]: # (TODO https://twitter.com/kelseyhightower/status/657761769570594816)


Links:
* Configuration of applications running in containers: https://github.com/joyent/containerbuddy
* A dynamic configuration file generation tool:  https://github.com/markround/tiller


### 7. Small
A container should have the least amount of code / libraries as possible to fulfil its job.

Reasons for a smaller image:
  * Faster in the network (deploy, reschedule, update)
  * Increased I/O performance
  * Smaller attack service. Easier to audit.

Use different containers for building and running. Note many containers are based on `debian/buildessentials` which is mostly unnecessary for runtime.

You don't have to use `Dockerfile`. Maybe creating a tar with something like [buildroot](https://buildroot.org/) and importing it via `docker import`. See the talk from Redbeard: [Best Practices For Containerized Environments](https://www.youtube.com/watch?v=gMpldbcMHuI).

Alpine is a minimalist linux distribution based on busybox, musl-libc, a new package manager called apk (not the Android one) and OpenRC as init system. [Some Thoughts on the Use of Alpine Linux in Docker Images](http://www.skippbox.com/thoughts-on-the-use-of-alpine-linux-in-docker-images/).

See image building guidelines like:

* [OpenShift Guidelines](https://docs.openshift.org/latest/creating_images/guidelines.html)
* [Docker best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/)
* [Project Atomic Guidance for Docker Image Authors](http://www.projectatomic.io/docs/docker-image-author-guidance/)
