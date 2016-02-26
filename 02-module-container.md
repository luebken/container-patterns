# Module container

TODO write something about platform agnostic with special care about docker

## Description

The term "module container" builds upon "application container" coined by Docker. An application container focuses on running a single process in contrast to multiple processes per container. If the application consists of multiple processes they are spread out to different containers. A module container refines application container with the focuses on being a good building block. In addition it suggests an even smaller granularity.

## Related work

TODO

* cluster aware images
* cloud native
* 12factor apps

## Definition

We currently don't have a clear definition. Instead we want to guide with a set of properties and associated best practices.

A module container consists of the following properties:

1. [Proper Linux process](#1-proper-linux-process)
2. [Explicit interfaces](#2-explicit-interfaces)
3. Disposable
4. Immutable
5. Self-Contained
6. Small

### 1. Proper Linux process
We should acknowledge the fact that a container is foremost a Linux process (isolated by namespaces and controlled by cgroups). Therefor we should apply common standards and best practices for writing Unix tools.

#### a) React to signals
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

#### b) Return proper exit codes
We should return proper exit codes when exiting the container. This gives us a better overview of what happened and the scheduler / init process better means of scheduling. E.g. in Kubernetes you can define that only failed containers [should be restarted](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/pod-states.md#restartpolicy).

We generally just differ between the exit code `0` as a successful termination and something `>0` as a failure. But other exit codes are conceivable. Some inspiration might give [glibc](https://github.molgen.mpg.de/git-mirror/glibc/blob/master/misc/sysexits.h) or [bash](http://tldp.org/LDP/abs/html/exitcodes.html).

**Example:**  
Return a non-failure exit code in Node.JS:

```javascript
process.exit(0);
```
https://github.com/luebken/currentweather/blob/master/server.js#L50

[//]: # (TODO better example with exit code 1)

#### c) Use standard streams

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

### 2. Explicit interfaces

TODO better intro

A good module has an explicit interface. We have touched on those in [Proper Linux process](#1-proper-linux-process) and want to further expand what is possible for containers.

**Handle Arguments**

TODO / TBD:  

* should this be moved to linux process?
* why do we describe this an not HTTP APIs?
* idea: move this up since it is "linux process.

Arguments are a straight forward way to configure and send options to a container. We propose that you should handle these in a comprehensible way.

Luckily CLI arguments are a very old topic so we refer to existing standards and libraries:

* [POSIX.1-2008 Utility conventions](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html)
* In conjunction with [getopt](http://pubs.opengroup.org/onlinepubs/9699919799/functions/getopt.html) as utility to parse / validate.
* Libcs [Program Argument Syntax Conventions](http://www.gnu.org/software/libc/manual/html_node/Argument-Syntax.html)


**Use Environment Variables**

In addition or as an alternative to command line arguments we can use environment variables to inject information into our container. Especially for configuration this has been made popular by the 12 Factor App: [III. Config Store config in the environment](Store config in the environment)

A common best practice is to set the [defaults envs](https://docs.docker.com/engine/reference/builder/#env) in the image and let the user overwrite it [](https://docs.docker.com/engine/reference/run/#env-environment-variables). See the section with labels below as an idea how to document these. 

**Declare Available Ports**

Another interface are the network ports our container potentially listens on. With the [EXPOSE](https://docs.docker.com/engine/reference/builder/#expose) directive. The ports will then show up in `docker ps` and `docker inspect`. And they can be [enforced](https://docs.docker.com/engine/userguide/networking/default_network/container-communication/#communication-between-containers) with setting `ipables=true` && `icc=false`.

**Mounts?**

TODO

**Document With Labels**



```json
{
  "key": "OPENWEATHERMAP_APIKEY",
  "description": "APIKEY to access the OpenWeatherMap. Get one at tp://openweathermap.org/appid",
  "mandatory": true
}
```
https://github.com/luebken/currentweather/blob/master/Dockerfile#L13

$ docker inspect -f "{{json .Config.Labels }}" <container image>
https://github.com/luebken/currentweather/blob/master/Makefile#L9



* https://github.com/progrium/entrykit

https://docs.docker.com/engine/userguide/labels-custom-metadata/
https://github.com/projectatomic/ContainerApplicationGenericLabels
https://github.com/garethr/docker-label-inspector
https://www.youtube.com/watch?v=j4SZ1qoR8Hs
https://speakerdeck.com/garethr/shipping-manifests-bill-of-lading-and-docker-metadata-and-container

Feature request: https://github.com/docker/docker/issues/20360
add label in one statement
access labels via inspect

**Use Hooks**



**Further reading / Related issues:**

* 



### 3. Disposable
### 4. Immutable
### 5. Self-Contained
### 6. Small