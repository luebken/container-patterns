# Container Patterns

Building applications with container technologies like [Docker](https://www.docker.com/) or [rkt](https://github.com/coreos/rkt) allows for new architecture and design decisions. In this document we would like to start a discussion around this and share experiences.

### Context

We are mainly talking about **cloud aware applications**. Or as the [cloud native foundations](https://cncf.io/about/our-mission) summarises: systems architected being _container packed_, _dynamically managed_ and _microservices oriented_.

The goal is to find and describe general applicable patterns for building these applications. The patterns should be container runtime agnostic but with concrete examples. 

### State

The current version is a discussion starter mainly taken from various feedback from the talks [@luebken](https://github.com/luebken) and [@denderello](https://github.com/denderello) have given. The latest version of the talk is from [QCon London](http://www.slideshare.net/luebken/container-patterns). For concrete issues please refer to the Github [issues](https://github.com/luebken/container-patterns/issues). 

### Feedback

Currently most of the discussions are done in other channels. The first idea to consolidate this is to [use issues](https://github.com/luebken/container-patterns/issues). But feel free to suggest other channels.

## Overview Content

* [Module Container](module-container.md)
* [Composite Patterns](composite-patterns.md)

We are collecting links which we found valuable at [resources.md](resources.md)

