# 7. Small
A container should have the minimum amount of code / libraries as possible to fulfil it's single purpose.

Reasons for a smaller image:

  * Faster in the network (deploy, reschedule, update)
  * Increased I/O performance
  * Smaller attack surface. Easier to audit.

Many containers are based of `debian/buildessentials` which is often unnecessary for runtime. Use different containers for building and running.

You don't have to use `Dockerfile`. Maybe creating a tar with something like [buildroot](https://buildroot.org/) and importing it via `docker import`. See the talk from Redbeard: [Best Practices For Containerized Environments](https://www.youtube.com/watch?v=gMpldbcMHuI).

Also have a look at Alpine. A minimalist Linux distribution based on busybox, musl-libc, a new package manager called apk (not the Android one) and OpenRC as init system. [Some Thoughts on the Use of Alpine Linux in Docker Images](http://www.skippbox.com/thoughts-on-the-use-of-alpine-linux-in-docker-images/).

## Further reading on image building guidelines

* [OpenShift Guideline](https://docs.openshift.org/latest/creating_images/guidelines.html) 
* [Docker best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/)
* [Project Atomic Guidance for Docker Image Authors](http://www.projectatomic.io/docs/docker-image-author-guidance/)