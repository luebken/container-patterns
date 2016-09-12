# 2. Container API

Although our container is foremost a Linux process it is also embedded in some kind of container runtime. This gives our "module container" more capabilities in defining APIs to it's environment and clients. These are _Use environment variables_, _Declare available ports_, _Volume mounts_ and _Hooks_.

## Use environment variables

In addition or as an alternative to command line arguments we can use environment variables (short envs) to inject information into our container. Especially for configuration this has been made popular by the 12 Factor App: [III. Config Store config in the environment](https://12factor.net/config). They are easy to change between deploys and there's no danger of checking them into a VCS.

These envs can be set when creating the image: [Docker](https://docs.docker.com/engine/reference/builder/#env), [rkt](https://github.com/appc/acbuild/blob/master/Documentation/subcommands/environment.md), [OCI](https://github.com/opencontainers/specs/blob/master/config.md#process-configuration). And let the user overwrite it when the container is started ([Docker](https://docs.docker.com/engine/reference/run/#env-environment-variables), [rkt](https://github.com/coreos/rkt/blob/master/Documentation/subcommands/run.md#influencing-environment-variables). 

## Declare available ports

One of the most important interface for distributed systems and the container are the network ports our container potentially listens on. 

In Docker with the [EXPOSE](https://docs.docker.com/engine/reference/builder/#expose) directive the ports will then show up in `docker ps` and `docker inspect`, and they can be [enforced](https://docs.docker.com/engine/userguide/networking/default_network/container-communication/#communication-between-containers) with setting `iptables=true` && `icc=false`.

In rkt with acbuild you can add a name to the [port declaration](https://github.com/appc/acbuild/blob/master/Documentation/subcommands/port.md). 

OCI [currently doesn't](https://github.com/opencontainers/specs/issues/178) offer a way to declare them.

## Volume mounts

In Docker you can define volumes when starting the container or when defining the image. We advise to declare volumes in the image so it can be examined before hand. 


## Hooks

Sometimes a container needs to react to a range of events during it's lifetime. Before termination we could check for the termination signal but that is a rather limited options since no context information can be given. In addition, we might want to react to additional events such as a post startup event.

Good examples are [container hooks in Kubernetes](http://kubernetes.io/docs/user-guide/container-environment/#container-hooks), and the [hooks in the opencontainer spec](https://github.com/opencontainers/specs/blob/master/config.md#hooks).

Docker currently doesn't natively support hooks but there is a proposal about adding them: [#6982](https://github.com/docker/docker/issues/6982) and an Fedora/Atomic [patch](https://github.com/projectatomic/docker/tree/fedora-1.10#add-dockerhooks-exec-custom-hooks-for-prestart-postspatch). Docker does submit [events](https://docs.docker.com/engine/reference/commandline/events/) which can be leveraged to implement non-blocking hooks. Jeff Lindsay happened to implement exactly this with [dockerhook](https://github.com/progrium/dockerhook). Another project to have look at is entrykit with it's [prehook](https://github.com/progrium/entrykit#prehook---run-pre-commands-on-start).

## Health endpoints

A major part of running containers in a cluster is checking whether they are healthy. A good example again can be found in [Kubernetes](http://kubernetes.io/docs/user-guide/production-pods/#liveness-and-readiness-probes-aka-health-checks) or with [Consul](https://www.consul.io/intro/getting-started/checks.html). We see these health endpoints as part of the container definition and will propose a way of documenting in the future.

## Further reading on API

* A discussion about the usage of envs: https://gist.github.com/telent/9742059
* "Parameterized Docker Containers": http://blog.james-carr.org/2013/09/04/parameterized-docker-containers
* A dynamic configuration file generation tool: https://github.com/markround/tiller
* A simple tool for inserting environment variables in templates is https://github.com/andreasjansson/envtpl or [envsubst from gettext](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html)
* Kelsey Hightower - healthz: Stop reverse engineering applications and start monitoring from the inside https://vimeo.com/173610242 & https://github.com/kelseyhightower/app-healthz
