# Docker Support in CF + Diego

This document discusses Diego's support for running Docker images and outlines how CF uses Diego to run Docker images.

> **Docker v1 image manifest is scheduled to be deprecated as of Diego v3.0.0. It's recommended to use the v2 image manifest schema going forward.**

- [Pushing a Docker image with the CF CLI](#pushing-a-docker-image-with-the-cf-cli)
	- [CLI 6.13.0 and later](#cli-6130-and-later)
	- [CLI 6.12.4 and earlier](#cli-6124-and-earlier)
- [How Diego runs Docker images](#how-diego-runs-docker-images)
- [How CC tells Diego to launch Docker images](#how-cc-tells-diego-to-launch-docker-images)
- [Docker in a multi-tenant world](#docker-in-a-multi-tenant-world)
- [Docker Deltas](#docker-deltas)

## Pushing a Docker image with the CF CLI

### CLI 6.13.0 and later

Versions 6.13.0 and later of the CF CLI include native support for pushing a Docker image as a CF app, with the `cf push` command's `-o` or `--docker-image` flags. For example, running

```bash
cf push my-app -o cloudfoundry/lattice-app
```

will push the image located at `cloudfoundry/lattice-app`.


### CLI 6.12.4 and earlier

Versions 6.12.4 and earlier of the CF CLI do not natively support pushing Docker images, but the [Diego CLI Plugin](https://github.com/cloudfoundry-incubator/diego-cli-plugin) provides this functionality with its `docker-push` command. For example, running

```bash
cf docker-push my-app cloudfoundry/lattice-app
```

will push the image located at `cloudfoundry/lattice-app`.



## How Diego runs Docker images

A Docker image consists of two things: a collection of layers to download and mount (the raw bits that form the file system) and metadata that describes what command should be run, as what user, and in what environment (the ENTRYPOINT and CMD directives, among others, specified in the Dockerfile).

Diego uses [Garden-Linux](https://github.com/cloudfoundry-incubator/garden-linux) to construct Linux containers. These containers are built on the same Linux kernel technologies that power all Linux containers: namespaces and cgroups. When a container is created a file system must be mounted as the root file system of the container. Garden-Linux supports mounting Docker images as root file systems for the containers it constructs. Garden-Linux fetches and caches the individual layers associated with the Docker image, then combines and mounts them as the root file system, using the same libraries that power Docker.

This process yields a container with contents that match the contents of the associated Docker image exactly.

Once a container is created Diego then runs and monitors processes inside of it. The [BBS API](https://github.com/cloudfoundry-incubator/bbs) allows the Diego consumer to specify exactly which commands to run within the container. In particular, it is possible to run, monitor, and route to multiple processes within a single container.


## How CC tells Diego to launch Docker images

To reiterate: Diego's LRPs and Tasks can reference a Docker image as the source for the container root filesystem.  In order to then *run* the appropriate process(es) within the container the LRP/Task must be configured appropriately.  Docker images include metadata that describe what should be run in the container.

To run a Docker image on Diego the Cloud Controller first performs a staging step.  This step runs as a Diego Task that fetches the metadata associated with the Docker image and returns some of it to the CC.  Once CC receives the Docker metadata, it uses it to construct an appropriate LRP and submits the LRP to Diego.  When constructing the LRP, the CC takes into account any user-specified overrides (for example, a custom start command or custom environment variables). The CC will also run the start command as the user specified in the Docker image, and will currently tell Diego and the Gorouter to route traffic to the lowest-numbered port exposed in the Docker image. (We expect CC to support routing to multiple ports soon.)

> Technically the responsibilities ascribed to the CC are shared between the CC and CC-Bridge, but this detail is not particularly important.

## Docker in a multi-tenant world

Since Docker allows users to fully specify the contents of their root filesystems the attack surface area for a Docker-based container running on Diego is somewhat higher than that of a buildpack application (which runs on a trusted root filesystem).

The Garden-Linux team has implemented a host of features to allow us to run Docker images more securely in a multi-tenant context.  In particular, we use the user-namespacing feature found on modern Linux kernels to ensure that even if a user manages to escalate privileges within the container (via a setuid executable, for example) they do not actually gain escalated privileges on the host.

CC always runs Docker containers on Diego with user namespaces enabled.  This does imply that certain features (such as mounting FuseFS devices) will not work in Docker containers.

There is more work to be done to mitigate our security concerns around running Docker containers in multi-tenant environments.  Until we have a greater degree of confidence we recommend running only *trusted* Docker containers on the platform.

Because of these concerns, CC currently default not to allow Docker-based apps to run on the platform. To enable them to run, a CC admin can turn on the `diego_docker` feature flag by running `cf enable-feature-flag diego_docker`. Disabling it later will cause CC to tell Diego to stop all Docker-based apps within a few convergence cycles (on the order of a minute).


## Docker Deltas

At this point, Garden-Linux runs Docker images with robust support for users embedded in the filesystem, and can run even the barest of images as containers.

Note that Diego runs and manages Docker applications just as it runs and manages build-pack based applications. In particular, it assumes that the application is a [12-factor app](http://12factor.net), and is therefore subject to the same lifecycle policies as buildpack-based 12-factor apps (such as restart with crash back-off, and evacuation during rolling updates of the Diego Cells). Diego does not yet support ways of mounting other volumes to Garden containers or linking separate containers, although these are both areas of active interest and research that we intend to address soon.


