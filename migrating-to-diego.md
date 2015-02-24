# Migrating to Diego

Diego is meant to be an in-place replacement of the DEAs.  Droplets staged on DEAs should run on Diego without any changes (and vice versa).

With that said, there are a handful of differences between Diego and the DEAs.  Some of these are slated to be addressed.  Some are not (though they can be if we get feedback that they need to be).

This migration guide is made up of two sections: 
- [Targeting Diego](#targeting-diego) describes the API calls necessary to run on Diego
    + [Installing the `diego-beta` CLI Plugin](#installing-the-diego-beta-cli-plugin)
    + [Starting a new application on Diego](#starting-a-new-application-on-diego)
    + [Transitioning an application between backends](#transitioning-an-application-between-backends)
    + [Running applications without routes](#running-applications-without-routes)
    + [Recognizing capacity issues](#recognizing-capacity-issues)
- [Diego Deltas](#diego-deltas) describes the differences between Diego and the DEAs
    + [Staging Performance](#staging-performance)
    + [Files API](#files-api)
    + [Health Checks](#health-checks)
    + [Behavior of Crashing Applications](#behavior-of-crashing-applications)
    + [Environment Variables](#environment-variables)
    + [Mixed Instances](#mixed-instances)
    + [Uptime During Deploys](#uptime-during-deploys)
    + [Application Metrics](#application-metrics)

## Targeting Diego

App developers can ask CF to run their applications on Diego by setting the `diego` boolean on their application to `true`.  Applications with `diego=true` will stage and run on Diego.

It is possible to modify the `diego` boolean on a running application.  This will cause it to transition from one backend to the other dynamically (though we make no guarantees around uptime).  The preferred approach is to perform a blue-green style deployment onto Diego.

The following instructions assume you have the `diego-beta` CLI plugin.  Instructions for installing it follow.

### Installing the `diego-beta` CLI Plugin

We have published a `cf` CLI plugin that makes opting into Diego easure.

For CF CLI v6.10.0+ you can install from the CF-Community repo:

```
cf add-plugin-repo CF-Community http://plugins.cloudfoundry.org/
cf install-plugin Diego-Beta -r CF-Community
```

For older versions of the CF CLI:

*To install the OS X binary:*
```
cf install-plugin https://github.com/cloudfoundry-incubator/diego-cli-plugin/raw/master/bin/osx/diego-beta.osx
```

*To install the linux binary:*
```
cf install-plugin https://github.com/cloudfoundry-incubator/diego-cli-plugin/raw/master/bin/linux64/diego-beta.linux64
```

*To install the windows binary:*
```
cf install-plugin https://github.com/cloudfoundry-incubator/diego-cli-plugin/raw/master/bin/win64/diego-beta.win64
```

The `diego-beta` plugin includes subcommands to `enable-diego` and `disable-diego`.  You can also check on whether an application has opted into Diego via `has-diego-enabled`.  There is also support around modfying the application's health check with `set-health-check` and `get-health-check`.

### Starting a new application on Diego

To start a new application on Diego you must push the application *without starting it*.  Once created, you can set the `diego` boolean on the application and *then* start it.

1. Push the application without starting it:

```
cf push APPLICATION_NAME --no-start
```

2. Set the `diego` boolean:

```
cf enable-diego APPLICATION_NAME
```

> This is equivalent to cf ``curl /v2/apps/`cf app APPLICATION_NAME --guid` -X PUT -d '{"diego":true}'``

3. Start the application:

```
cf start APPLICATION_NAME
```

### Transitioning an application between backends

Simply setting the `diego` boolean:

```
cf enable-diego APPLICATION_NAME
```

will cause an existing application to transition to Diego.  The application will immediately start running on Diego and will *eventually* stop running on the DEAs.  While this gives some safety there are no strong guarantees around uptime.

If you want to ensure uptime we recommend performing a blue-green deploy (i.e. push a copy of your application to Diego then swap routes and scale down the DEA application).

Note that you *must* ensure that the application has the correct stack when switching to Diego.  Diego, currently, runs a `lucid64` stack but there are plans to (very soon) transition to `trusty64` only.  Since a stack change necessitates a restage you must choose to either restage directly on Diego *or* restage on `trusty64` DEAs and then switch over the `diego:true` boolean.

To transition back to the DEAs:

```
cf disable-diego APPLICATION_NAME
```

To tell which backend the application is targetting:

```
cf has-diego-enabled APPLICATION_NAME
```

### Running applications without routes

For the DEA backend `cf push APP_NAME --no-routes` does two things:

- it skips creating and binding a route for the application
- it (indirectly) causes the DEAs to skip the port healthcheck

> By default, when starting an application the DEAs wait until the application is listening on its assigned port *before* marking it as ready to receive traffic.  To determine whether or not to perform this check, the DEA peeks into the routes bound to the application and determines: if they're present the port check is performed.  If they're empty, no port check is performed.

Diego does health checks [differently compared to the DEAs](#health-checks).  With Diego `cf push APP_NAME --no-routes` only skips creating and binding a route for the application.  It does not tell Diego which type of health check to perform.

By default, Diego does the same port-based health check that the DEA performs.  If your applcation does *not* listen on a port (e.g. you are pushing a resque worker) then Diego will never see the application come up and will eventually mark it as crashed.  In thsese cases you must tell Diego to perform no health check:

```
cf set-health-check APPLICATION_NAME none
```

Note: there are two valid values for the health check: `port` and `none`.  `port` is the default.  You can get the current health check via

```
cf get-health-check APPLICATION_NAME
```

### Recognizing capacity issues

The Cloud Controller is responsible for scheduling applications on the DEAs.  With Diego this responsibility shifts entirely to Diego.  As a result, the Cloud Controller does not know, ahead of time, whether or not there is capacity to stage/run the application.  Instead, this information (referred to as a placement error) is available *asynchronously* and comes from Diego via the `cf app` api.

The CLI has already been updated to:

- display placement error information when `cf app` is invoked
- inform users when staging fails because of a placement error
- inform users when `cf push` fails because the application cannot be placed

> Currently, `cf apps` is misleading.  It will show all instances as healthy even if some of them have a placement error.  We intend to address this soon.

## Diego Deltas

Here's a list of some of the (known!) differences between Diego and the DEAs.

### Staging Performance

Diego's staging performance is somewhat slower than the DEAs.

##### Why?

The DEAs are tightly coupled to the notion that staging entails running through a set of buildpacks.  Because of this there are optimizations in place that treat buildpacks as *special things*: in short, the DEAs basically mount the buildpacks directly into containers.

Diego, being a generic container runtime, does not treat buildpacks in a special way.  They are simply assets that are downloaded and copied into containers.  The only optimization in place is a local download cache that allows Diego to avoid the download step.  At this time, however, the buildpacks need to be *copied* into each container - this copy step is expensive and does not parallelize well (it's disk-performance-bound).  This is excaserbated by the size of CF's offline buildpacks.

##### Workarounds

The simplest way to speed up staging performance is to specify a particular buildpack via the `-b` flag on `cf push`.  This will cause Diego to only fetch and copy in the single specified buildpack.  For example:

```
cf push my-app -b ruby_buildpack --no-start
cf enable-diego my-app
cf start
```

Will stage the requested application using only the Ruby buildpack.

##### Future plans

There are plans to eventually improve Diego's semantics around mounting shared volumes across containers.  When this lands we will be able to directly mount the buildpacks into the containers (much like the DEAs do) without breaking Diego's generic abstractions.  A placeholder story is [here](https://www.pivotaltracker.com/story/show/88734100).

### Files API

Diego does not support the `cf files` API.

##### Why?

CF's existing feature set is massive and we had to cut scope to ship a Diego beta in a timely fashion.  Moreover, while supporting the files API in particular is relatively straightforward - we are hoping to solve the problem of "accessing a running container" more generically and comprehensively.

##### Workarounds

None*

> * Technically, there are hacky ways around this.  There's nothing stopping running applications from having secured endpoints that fetch files from their local filesystems...

##### Future Plans

We are planning on providing full-blown SSH access to running containers.  The intent is to support at least the following three usecases:

- Shell access via SSH
- `scp` support for fetching files
- Support for port-forwarding

We believe this solves a host of problems by building on top of an established, and secure, protocol.

### Health Checks

The DEAs perform a single health check when launching an application.  This health is used to verify that the application is "up" before routing to it.  The default health check simply checks that the application has started listening on `$PORT`.  Once the application is up the DEA no longer performs any health checks.  The application is considered crashed *only* when it exits.  As mentioned [above](#running-applications-without-routes) applications with no associated routes aren't health-checked at all.

Diego does health checks differently.  Like the DEAs, Diego performs the health check to identify when the application is "up".  Diego *continues* to perform the health check (every 30 seconds) after the application comes up.  This allows Diego to identify stuck applications - applications that may still be running but are (actually) in an unhealthy state - and restart them.

Currently Diego supports a port-based health check (like the DEAs).  However, Diego's health check is completely generic (Diego simply runs a process in the container periodically - if the process exits succesfully the application is considered healthy).  There are plans to support URL-based health checks.  We can also support launching arbitrary binaries to allow users to perform custom health checks.

Applications that do not listen on a port will need to disable the health check.  THis is described [above](#running-applications-without-routes).

### Behavior of Crashing Applications

As with the DEAs/Health Manager, Diego restarts crashed applications.  There are a handful of differences:

- Diego does not keep crashed containers around.
- Diego *stops* restarting crashed applications eventually.

##### Why?

*Diego does not keep crashed containers around:*

With the advent of loggregator the need to keep crashed containers around (e.g. to fetch logs) is substantially reduced.

However, because Diego supports health checks it will be possible to identify containers that have "stuck" applications.  In that context having access to the container could help debug the broken application.  Once Diego enables SSH support we will consider allowing containers with unhealthy (but still running) applications to stick around (for an hour or so) to allow users to access and debug the stuck processes.

*Diego stops restarting crashed applications eventually:*

Our experience with large installations of CF is that there are a sizable number of applications that repeatedly crash and never fail to stay up.  (e.g. poorly written hello world applications).  These place a strain on the installation and consume unnecessary resources.  Diego attempts to restart these applications for ~2 days but then gives up on them.

##### Workarounds

None

##### Future Plans

Diego's restart policy is currently static.  There are plans to make it configurable on a space-by-space level.  This is discussed at length on [vcap-dev](https://groups.google.com/a/cloudfoundry.org/forum/#!topic/vcap-dev/tJTIkoD8__o/discussion) with stories beginning [here](https://www.pivotaltracker.com/story/show/87479698).

### Environment Variables

Diego does not interpolate environment variables (i.e. referring to one environment variable in another via `$` will not work).  We'd like to see if this is, in fact, an issue for people.  If you have trouble because of this please reach out on vcap-dev and we can look into workarounds/fixes.

### Mixed Instances

With the DEAs it is currently possible to create end up with instances that are configured differently from other instances of an application.  For example, it is possible to modify the start command then scale an application up.  The new instances will have the new start command whereas the old ones will not.

This is bad.

Should an old instance need to restart it will restart with the *new* start command!  This is almost certainly not what you intended!

Diego does not allow this behavior.

##### Why?

Diego is actually very opinionated about what can change about a running application.  Currently, only routes and the number of instances can be modified without restarting the application.  Any other changes (e.g. changes to environment variables, start commands, etc.) necessarily require a restart.

Diego does not orchestrate this restart for you - it leaves it to the user to bring up new instances and bring down old instances.  This is the safest way to correctly and safely transition applications between configurations without causing downtime.

##### Workarounds

Always use a green-blue deploy strategy when modifying anything about a running application.  If you'd like some instances of an application to have different configuration than other instances you should, instead, stage and deploy two different applications.

> Alternatively you can use the `INSTANCE_INDEX` environment variable to dynamically change your applications behavior based on its instance number.  This is not recommended.

##### Future plans

None

### Uptime During Deploys

Diego ensures that a applications will remain available during a rolling deploy of the Diego cluster.

However, during the beta period, we will likely need to make backward incompatible changes to the cluster which may result in downtime/the need for user intervention (typically an application restart - not a restage - should suffice).

##### Why?

Diego is moving fast - our next major effort is going to be around optimization which may force us to modify how we store application data.  Rather than get bogged down in ensuring backward compatibility and migrating the data between deploys we reserve the right to make breaking changes that will require application restarts.

##### Workarounds

If you absolutely must have 100% uptime during the beta period we recommend cloning your application and running a subset of instances on the DEA backends.  Your DEA application and Diego application could share the same route and the router will transparently load-balance between them both.

##### Future plans

Diego already ensures that applications remain available during a rolling deploy.  As we get closer to a final candidate we will introduce infrastructure to ensure that backward incompatible changes are migrated appropriately.

### Application Metrics

Application Metrics (CPU/Memory/Disk usage) flow through loggregator with Diego.  The work to support this is almost complete with some pending CLI work still to be done.  Until then it is not possible to fetch application metrics with `cf app`.
