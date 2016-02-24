# Module container

TODO write something about platform agnostic with special care about docker

## Defintion

The term "module container" builds upon the term "application container" coined by Docker. An application container focuses on running a single process inside a container in contrast to multiple processes per container. Different processes are spread out to different containers. A module container focuses on the aspect on being a good building block. In addition it suggests an even smaller granularity. And thinks of building an application of multiple containers.

## Related work

TODO

* cluster aware images
* cloud native

## Properties

A module container consists of the following properties:

1. [Proper Linux process](#1-proper-linux-process)
2. Explicit interfaces
3. Disposable
4. Immutable
5. Self-Contained
6. Small

### 1. Proper Linux process
We should acknowledge the fact that a container is foremost a Linux process (isolated by namepaces and controlled by cgroups). Therefor we should apply common Unix best practices.

#### a) React to signals
Don't daemonize your process and keep it in the foreground. Catch signals in you program and react appropriate. 

_Catch signals:_
Important signals: 
* `SIGINT`: E.g. send by Ctrl-C to interrupt / stop a running container.  
* `SIGTERM`: Signals process termination e.g. send by `docker stop`. If necessary the container can do some final steps. After an usual grace period of 10 seconds the container is killed if it hasn't returned anymore

Example catching `SIGTERM` in Node.JS:
```javascript
process.on('SIGTERM', function () {
  console.log("Received SIGTERM. Exiting.")
  server.close(function () {
    process.exit(0);
  });
});
```
https://github.com/luebken/currentweather/blob/master/server.js#L47

_Use the exec form:_
For Docker prefer the [exec form](https://docs.docker.com/engine/reference/builder/#run) which doesn't invoke a shell.

Further reading:
* The [man page](http://man7.org/linux/man-pages/man7/signal.7.html) contains a good overview of signals.
* [What makes an awesome CLI Application](https://pragprog.com/magazines/2012-05/what-makes-an-awesome-commandline-application) give some inspiration.
* [Signal handlers must be reentrant](http://blog.rubybestpractices.com/posts/ewong/016-Implementing-Signal-Handlers.html#fn1) What happens when another signal arrives while this handler is still running?
* [Self pipe trick](http://cr.yp.to/docs/selfpipe.html) Maintain a pipe for signals. 

#### b) Return proper exit codes
We should return proper exit codes when exiting the container. This gives us a better overview of what happened and the scheduler / init process better means of scheduling. E.g. in Kubernetes you can define that only failed containers [should be restarted](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/pod-states.md#restartpolicy).

We generally just differ between the exit code `0` as a successful termination and something `>0` as a failure. But other exit codes are conceivable. Some inspiration might give [glibc](https://github.molgen.mpg.de/git-mirror/glibc/blob/master/misc/sysexits.h) or [bash](http://tldp.org/LDP/abs/html/exitcodes.html).

Example return a non-failure exit code in Node.JS:
```javascript
process.exit(0);
```
https://github.com/luebken/currentweather/blob/master/server.js#L50

[//]: # (TODO better example with exit code 1)

#### c) Use standard streams

Linux processes use standard streams as a means of communication. There are `stdin`: standard input, `stdout`: standard output and `stderr` standard error. Let's talk about each:

_stdout_
For all logging activities use stdout let the infrastructure take care of handling this stream or forwarding it to some log aggregator. 

[//]: # (TODO point to log side-car and write an example)

_stderr_
TODO: Not sure what to write about

_stdin_
Our container could be a Unix tool and accept data from stdin.

Tips:
* If your app writes to a file link that file to the device file: 
`RUN ln -sf /dev/stdout /var/log/nginx/access.log` 
https://github.com/nginxinc/docker-nginx/blob/master/stable/jessie/Dockerfile#L14

Further reading:
* 12 Factor apps: [Treat logs as event streams](http://12factor.net/logs).

### 2. Explicit interfaces

TODO

Builds upon 1. Proper Linux Process
What can we do more? 
  * Hooks
  * Labels
  * 
Unix args http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html

### 3. Disposable
### 4. Immutable
### 5. Self-Contained
### 6. Small