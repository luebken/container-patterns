# Container Patterns

Building and designing applications with container technologies like [Docker](https://www.docker.com/) or [rkt](https://github.com/coreos/rkt) require new approaches. In this document we would like to start a discussion around this and share experiences.

## Context

We are mainly talking about "cloud aware applications". The [cloud native foundations](https://cncf.io/about/our-mission) summarises these as "container packed", "dynamically managed" and "microservices oriented".

## Goal

Find and describe general applicaple patterns for building "cloud aware applictions". The patterns should be container runtime agnostic but with concrete examples. 

## Overview

* [Module Container](module-container.md)
* [Composite Patterns](composite-patterns.md)

## State

The current version is a discusssion starter mainly taken from various feedback from the talks [https://github.com/luebken](@luebken) and [https://github.com/denderello](@denderello) have given. The latest version of the talk is from the [QCon London](http://www.slideshare.net/luebken/container-patterns).

For concrete issues please see TODOs within the markdowns or refer to the Github [issues](https://github.com/luebken/container-patterns/issues). 

## Resources

We are collecting links which we found valuable at [resources.md](resources.md)

## Discussion

Currently most of the discussions are done in other channels. The first idea to consolidate this is to [use issues](https://github.com/luebken/container-patterns/issues). But feel free to suggest other channels.
