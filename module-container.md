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
