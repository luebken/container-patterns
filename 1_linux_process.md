# Linux Process

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
