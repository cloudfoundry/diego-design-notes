# Docker Support in CF + Diego

This document discusses Diego's support for running Docker images and outlines how CF uses Diego to run Docker images.

- [How Diego launches Docker images](#how-diego-launches-docker-images)
- [How CC tells Diego to launch Docker images](#how-cc-tells-diego-to-launch-docker-images)
- [Docker in a multi-tenant world](#docker-in-a-multi-tenant-world)
- [Docker Deltas](#docker-deltas)
- [Pushing a Docker image with the Diego CLI plugin](#pushing-a-docker-image-with-the-diego-cli-plugin)

> Some of this is adapted from the [Lattice documentation](http://lattice.cf/docs/troubleshooting/#how-does-lattice-work-with-docker-images)

## How Diego launches Docker images

A Docker image consists of two things: a collection of layers to download and mount (the raw bits that form the file system) and metadata that describes what command should be launched (the ENTRYPOINT and CMD directives, among others, specified in the Dockerfile).

Diego uses [Garden-Linux](https://github.com/cloudfoundry-incubator/garden-linux) to construct Linux containers. These containers are built on the same Linux kernel technologies that power all Linux containers: namespaces and cgroups. When a container is created a file system must be mounted as the root file system of the container. Garden-Linux supports mounting Docker images as root file systems for the containers it constructs. Garden-Linux takes care of fetching and caching the individual layers associated with the Docker image and combining and mounting them as the root file system - it does this using the same libraries that power Docker.

This yields a container with contents that exactly match the contents of the associated Docker image.

Once a container is created Diego is responsible for running and monitoring processes in the container. The [Receptor API](https://github.com/cloudfoundry-incubator/receptor/blob/master/doc/README.md) allows the Diego consumer to define exactly which commands to run within the container; in particular, it is possible to run, monitor, and route to multiple processes within a single container.

## How CC tells Diego to launch Docker images

To reiterate: Diego's LRPs and Tasks can reference a Docker image as the source for the container root filesystem.  In order to then *run* the appropriate process(es) within the container the LRP/Task must be configured appropriately.  Docker images include metadata that describe what should be run in the container.

To run a Docker image on Diego the Cloud Controller first performs a staging step.  This spins up a Diego task that fetches the metadata associated with the Docker image and returns it to the CC.  Once this completes the CC uses the Docker metadata to construct an appropriate LRP and submits the LRP to Diego.  When constructing the CC takes into account any user-specified overrides (e.g. a custom start command or custom environment variables).

> Technically the responsibilities ascribed to the CC are shared between the CC and CC-Bridged, but this detail is not particularly important.

## Docker in a multi-tenant world

Since Docker allows users to fully specify the contents of their root filesystems the attack surface area for a Docker-based container running on Diego is somewhat higher than that of a buildpack application (which runs on a trusted root filesystem).

The Garden-Linux team is working on a host of features to allow us to more securely run Docker images in a multi-tenant context.  In particular, we leverage the user namespacing feature found on modern Kernels to ensure that even if a user manages to escalate privileges within the container (via a setuid executable, for example) that they do not actually gain escalated privileges on the host.

CC always runs Docker containers on Diego with user namespaces enabled (it is not possible to disable this funcitonality at this point in time).  This does imply that certain features (e.g. mounting FuseFS devices) will not work in Docker containers.

There is more work to be done to mitigate our security concerns around running Docker containers in multi-tenant environments.  Until we have a greater degree of confidence we recommend only running *trusted* Docker containers on the platform.

## Docker Deltas

There are two major remaining areas of Docker compatbility that we are working on:

1. Removing assumptions about container contents. Currently, Garden-Linux makes some assumptions about what is available inside the container. Some Docker images do not satisfy these assumptions though most do (the liteweight busybox base image, for example).
2. Supporting arbitrary UIDs and GIDs. Currently Garden-Linux runs applications as the vcap user (a historical holdover).  We intend to fully support the USER directive which should make bringing arbitrary Docker images to the platform more seamless.

Finally, note that Diego runs and manages Docker applications just like it runs and manages build-pack based applications.  In particular, it is assumed that the application is a [12-factor app](http://12factor.net).

## Pushing a Docker image with the Diego CLI plugin

The [Diego CLI Plugin](https://github.com/cloudfoundry-incubator/diego-cli-plugin) includes support for pushing a Docker image via CC.  For example:

```
cf docker-push my-app cloudfoundry/lattice-app
```

will push the image located at `cloudfoundry/lattice-app`.

Note that you will need to `cf enable-feature-flag diego_docker` to enable Docker support in the CC.
