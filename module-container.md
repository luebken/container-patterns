# Module container

Developing container based applications is still a fairly new topic. Even more are general ideas such as using containers as modules to architect applictions. To assit further discussions this document tries to gather current best practices and tries to suggests some new ideas to the community. Most of these ideas should be container runtime agnostic but still practical with relevant and concrete examples.

Many of these ideas are formalised in the [Open Containers Spec](https://github.com/opencontainers/specs) but we want to give guidance for tools in use today. We will try to link to Docker, rkt and OCI examples where applicable.

> Note: This document is highly Work-In-Progress. Please get involved. Comment discuss and add your own ideas.  

## Definition

The term "module container" builds upon the term "application container" coined by Docker. An application container focuses on running a single process in contrast to multiple processes per container. If the application consists of multiple processes they may be spread out to different containers. A module container refines an application container and focuses on being a modular isolated building block in the provision of a service. In addition it suggests an even smaller level of granularity.

## Related work

In this field there is much prior and contemporary art:

* [The 12 Factor App](http://12factor.net/) a widely cited site describing how to write good PaaS applications. Initiated by Heroku. Many of these principles apply for containers as well.

* Cloud native: In the past year the term "cloud native applications" has gained popularity. A good introduction can be found in the free ebook [Migrating to Cloud Native Application Architectures](http://pivotal.io/platform/migrating-to-cloud-native-application-architectures-ebook)

The folks from [Deis](http://deis.io/) have coined the term [cluster aware image](https://twitter.com/gabrtv/status/676218070357032961).

## Properties of a module container

There is no canonical definition of a "module container". Instead we gather a set of guiding properties and associated best practices:

1. [Linux process](#1-linux-process)
2. [Container API](#2-container-api)
3. [Explicit interfaces](#3-explicit-interfaces)
4. [Disposable](#4-disposable)
5. [Immutable](#5-immutable)
6. [Self-Contained](#6-self-contained)
7. [Small](#small)

### 1. Linux process

Before we try to introduce too many new ideas, we should acknowledge the fact that a container is foremost a Linux process. Therefore we should apply common standards and best practices for writing Unix tools, that now happen to be running in containers. These are _React to signals_, _Use standard streams_, _Handle arguments_:

#### React to signals
A container should react to the [signals](https://en.wikipedia.org/wiki/Unix_signal) which are sent to it. So our applications in the container should run in the foreground and catch signals and react appropriately. 

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
https://github.com/luebken/currentweather/blob/master/server.js#L49

**Docker best practices:**
* Use the [exec form](https://docs.docker.com/engine/reference/builder/#run) to start you processes. Since it doesn't invoke a shell.

#### Return exit codes
When a process exits it should return a proper exit code. The same is true for our container. This gives us a better overview of the context and the scheduler / init process better means of scheduling. E.g. in Kubernetes you can define that only failed containers [should be restarted](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/pod-states.md#restartpolicy).

We generally just differ between the exit code `0` as a successful termination and something `>0` as a failure. But other exit codes are also conceivable. For some inspiration, check out [glibc](https://github.molgen.mpg.de/git-mirror/glibc/blob/master/misc/sysexits.h) or [bash](http://tldp.org/LDP/abs/html/exitcodes.html).

**Example for exit codes:**
Return a non-failure exit code in Node.JS:

```javascript
process.exit(0);
```
https://github.com/luebken/currentweather/blob/master/server.js#L52

#### Use standard streams

Linux processes use standard streams as a means of communication. There are `stdin`: standard input, `stdout`: standard output and `stderr` standard error:

* `stdout`: For all logging activities use stdout and let the infrastructure take care of handling this stream or forwarding it to some log aggregator. 

* `stdin`: If our container can be be conceived as a Unix tool we should accept data from stdin. This would allow [piping between containers](http://matthewkwilliams.com/index.php/2015/04/21/piping-hot-docker-containers/).

**General best practices:**

* If you do have specific logging requirements you can use a side-car container to adapt your logs. See the [composite patterns](composite-patterns.md) for details.

* If your app writes to a file you could link that file to the device file:

  ```Dockerfile
  RUN ln -sf /dev/stdout /var/log/nginx/access.log
  ```
  https://github.com/nginxinc/docker-nginx/blob/master/stable/jessie/Dockerfile#L14

#### Handle arguments

Arguments are a straightforward way to configure and send options to a container. We propose that you should handle these in a prescribed manner. And get your team to discuss and agree on a standard.

Luckily CLI arguments are a well defined topic so we refer to existing standards and libraries instead of re-inventing the wheel:

* [POSIX.1-2008 Utility conventions](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html)
* In conjunction with [getopt](http://pubs.opengroup.org/onlinepubs/9699919799/functions/getopt.html) as utility to parse / validate.
* Libcs [Program Argument Syntax Conventions](http://www.gnu.org/software/libc/manual/html_node/Argument-Syntax.html)


#### Further reading on Linux process:

* The [man page](http://man7.org/linux/man-pages/man7/signal.7.html) contains a good overview of signals.
* [What makes an awesome CLI Application](https://pragprog.com/magazines/2012-05/what-makes-an-awesome-commandline-application) gives some inspiration.
* [Signal handlers must be reentrant](http://blog.rubybestpractices.com/posts/ewong/016-Implementing-Signal-Handlers.html#fn1) What happens when another signal arrives while this handler is still running?
* [Self pipe trick](http://cr.yp.to/docs/selfpipe.html) Maintain a pipe for signals. 
* In 12 factor apps: [Treat logs as event streams](http://12factor.net/logs).
* Also be aware of [the PID 1 zombie reaping problem](https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/) in Docker


### 2. API

Although our container is foremost a Linux process it is also contained in it's own environment. This gives our "module container" more capabilities in defining APIs to it's environment and clients. These are _Use environment variables_, _Declare available ports_, _Volume mounts_ and _Hooks_.

#### Use environment variables

In addition or as an alternative to command line arguments we can use environment variables (short envs) to inject information into our container. Especially for configuration this has been made popular by the 12 Factor App: [III. Config Store config in the environment](https://12factor.net/config). They are easy to change between deploys and there's no danger of checking them into a VCS.

These envs can be set when creating the image: [Docker](https://docs.docker.com/engine/reference/builder/#env), [rkt](https://github.com/appc/acbuild/blob/master/Documentation/subcommands/environment.md), [OCI](https://github.com/opencontainers/specs/blob/master/config.md#process-configuration). And let the user overwrite it when the container is started ([Docker](https://docs.docker.com/engine/reference/run/#env-environment-variables), [rkt](https://github.com/coreos/rkt/blob/master/Documentation/subcommands/run.md#influencing-environment-variables). 

#### Declare available ports

One of the most important interface for distributed systems and the container are the network ports our container potentially listens on. 

In Docker with the [EXPOSE](https://docs.docker.com/engine/reference/builder/#expose) directive the ports will then show up in `docker ps` and `docker inspect`, and they can be [enforced](https://docs.docker.com/engine/userguide/networking/default_network/container-communication/#communication-between-containers) with setting `iptables=true` && `icc=false`.

In rkt with acbuild you can add a name to the [port declaration](https://github.com/appc/acbuild/blob/master/Documentation/subcommands/port.md). 

OCI [currently doesn't](https://github.com/opencontainers/specs/issues/178) offer a way to declare them.

#### Volume mounts

In Docker you can define volumes when starting the container or when defining the image. We advise to declare volumes in the image so it can be examined before hand. 


#### Hooks

Sometimes a container needs to react to a range of events during it's lifetime. Before termination we could check for the termination signal but that is a rather limited options since no context information can be given. In addition, we might want to react to additional events such as a post startup event.

Good examples are [container hooks in Kubernetes](http://kubernetes.io/docs/user-guide/container-environment/#container-hooks), and the [hooks in the opencontainer spec](https://github.com/opencontainers/specs/blob/master/config.md#hooks).

Docker currently doesn't natively support hooks but there is a proposal about adding them: [#6982](https://github.com/docker/docker/issues/6982) and an Fedora/Atomic [patch](https://github.com/projectatomic/docker/tree/fedora-1.10#add-dockerhooks-exec-custom-hooks-for-prestart-postspatch). Docker does submit [events](https://docs.docker.com/engine/reference/commandline/events/) which can be leveraged to implement non-blocking hooks. Jeff Lindsay happened to implement exactly this with [dockerhook](https://github.com/progrium/dockerhook). Another project to have look at is entrykit with it's [prehook](https://github.com/progrium/entrykit#prehook---run-pre-commands-on-start).

#### Health endpoints

A major part of running containers in a cluster is checking whether they are healthy. A good example again can be found in [Kubernetes](http://kubernetes.io/docs/user-guide/production-pods/#liveness-and-readiness-probes-aka-health-checks) or with [Consul](https://www.consul.io/intro/getting-started/checks.html). We see these health endpoints as part of the container definition and will propose a way of documenting in the future.

#### Further reading on API

* A discussion about the usage of envs: https://gist.github.com/telent/9742059
* "Parameterized Docker Containers": http://blog.james-carr.org/2013/09/04/parameterized-docker-containers
* A dynamic configuration file generation tool: https://github.com/markround/tiller
* A simple tool for leveraging environment variables in jinja2 templates: https://github.com/andreasjansson/envtpl



### 3. Descriptive

A good module has an explicit, or as wikipedia says [well defined](https://en.wikipedia.org/wiki/Modular_programming#Key_aspects) interface. So with all the ways declaring an API for our container we also need a way to document this. As this is not always possible within the declaration we want to expand on that and use labels to document our API.

#### Document With Labels

We like to propose to document the API with labels as all container engines have the means of doing so.

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

A module container should always strive for being exchanged with a copy at any point of time. This especially true in an cluster environment where there can be many reasons for a particular container be stopped:
 
 * Rescheduling because of limit or bad resources
 * Down-scaling
 * Errors within the container
 * Migration to new hardware / locality of services

This concept is so widely accepted in the container space that developers use the `--rm` with Docker as a default which always remove the container after they have stopped. We have chosen the term "disposable" from the [12Factor app](http://12factor.net/disposability).

**Best practices on being dispoable:**

* Be robust against sudden death.  
  If the container gets interrupted pass on your current job. (See ["React to signals"](#react-to-signals))
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

Sometimes developers put application code into a volume to cut the container build time. If you do so please do it only for yourself and include all application code once the container leaves your computer.

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
Which is blatantly copied from the great blogpost by Kelsey on [12 Fractured Apps](https://medium.com/@kelseyhightower/12-fractured-apps-1080c73d481c#.g3ydhs5zw).


[//]: # (TODO write something about config files http://www.mricho.com/confd-and-docker-separating-config-and-code-for-containers/)
[//]: # (TODO https://twitter.com/kelseyhightower/status/657761769570594816)


#### Further reading on configuration

* Configuration of applications running in containers: https://github.com/joyent/containerbuddy
* A dynamic configuration file generation tool: https://github.com/markround/tiller


### 7. Small
A container should have the least amount of code / libraries as possible to fulfil it's job. 

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
