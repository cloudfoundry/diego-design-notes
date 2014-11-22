# LRPs: Long Running Processes

Diego can distribute and monitor multiple instances of a Long Running Process (LRP).  These instances are distributed across Diego Cells and restarted automatically if they crash or disappear.  The instances are identical (though each instance is given a unique index (in the range `0, 1, ...N-1`) and a a unique instance guid).

LRPs are described by providing Diego with a `DesiredLRP`.  The `DesiredLRP` can be thought of as a manifest that describes how an LRP should be executed and monitored.

The instances that end up running on Diego cells are referred to as `ActualLRP`s.  The `ActualLRP`s contain information about the state of the instance and about the host Cell the instance is running on.  

When describing a property common to both `DesiredLRP`s and `ActualLRP`s (e.g. the `process_guid`) we may refer to both notions collectively simply as LRPs.

Diego is constantly monitoring and reconciling desired state and actual state.  As such it is important to ensure that the desired state is up-to-date and accurate.  This is covered in detail in the section below on [Freshness](#freshness).

First, let's discuss DesiredLRPs.

## Describing DesiredLRPs

When desiring an LRP you `POST` a valid `DesiredLRPCreateRequest`.  The [API reference](api_lrps.md) includes the details of the request.  Here we simply describe what goes into a `DesiredLRPCreateRequest`:

```
{
    "process_guid": "some-guid",
    "domain": "some-domain",

    "stack": "lucid64",

    "instances": 17,

    "root_fs": "docker:///docker-org/docker-image",
    "env": [
        {"name": "ENV_NAME_A", "value": "ENV_VALUE_A"},
        {"name": "ENV_NAME_B", "value": "ENV_VALUE_B"}
    ],

    "cpu_weight": 57,
    "disk_mb": 1024,
    "memory_mb": 128,

    "setup": ACTION,
    "action":  ACTION,
    "monitor": ACTION,
    "start_timeout": N seconds,

    "ports": [8080, 5050],
    "routes": ["a.example.com", "b.example.com"],

    "log_guid": "some-log-guid",
    "log_source": "some-log-source",

    "annotation": "arbitrary metadata"
}
```

Let's describe each of these fields in turn.

#### LRP Identifiers

#### `process_guid` [required]

It is up to the consumer of Diego to provide a *globally unique* `process_guid`.  To subsequently fetch the DesiredLRP and its ActualLRP you refer to it by its `process_guid`.

- The `process_guid` must only include the characters `a-z`, `A-Z`, `0-9`, `_` and `-`.
- The `process_guid` must not be empty
- If you attempt to create a DesiredLRP with a `process_guid` that matches that of an existing DesiredLRP, Diego will attempt to update the existing DesiredLRP.  This is subject to the rules described in [updating DesiredLRPs](#updating-desiredlrps) below.

#### `domain` [required]

The consumer of Diego may organize their LRPs into groupings called Domains.  These are purely organizational (e.g. for enabling multiple consumers to use Diego without colliding) and have no implications on the ActualLRP's placement or lifecycle.  It is possible to fetch all LRPs in a given Domain.

- It is an error to provide an empty `domain`.

#### LRP Placement

In the future Diego will support the notion of Placement Pools via arbitrary tags associated with Cells.  For now, this functionality is limited to the notion of `stack`.

#### `stack` [required]

Diego can support different target platforms (linux, windows, etc.). `stack` allows you to select which target platform the Task must run against.  For a typical Diego deployment you should set `stack` to `lucid64`

- It is an error to provide an empty `stack`.

#### Instances

#### `instances` [required]

Diego can run and manage multiple instances (`ActualLRP`s) for each `DesiredLRP`.  `instances` specifies the number of desired instances and must be a positive integer.

- It is an error for `instances` to be `0`.  You must explicitly `DELETE` the `DesiredLRP` to shut down all its instances.

#### Container Contents and Environment

#### `root_fs` [optional]

By default, when provisioning a container Diego will mount a pre-configured root filesystem.  Currently, the default filesystem provided by [diego-release](https://github.com/cloudfoundry-incubator/diego-release) is based on lucid64 and is geared towards supporting the Cloud Foundry buildpacks.

It is possible, however, to provide a custom root filesystem by specifying a Dockerimage for `root_fs`:

```
"root_fs": "docker:///docker-org/docker-image#docker-tag"
```

Currently, only the public docker hub is supported.

> You *must* specify the dockerimage `root_fs` uri as specified, including the leading `docker:///`!

> [Diego-Edge](http://github.com/cloudfoundry-incubator/diego-lite) does not ship with a default rootfs.  You must specify a docker-image when using Diego-Edge.  You can mount the filesystem provided by diego-release by specifying `"root_fs": "docker:///cloudfoundry/lucid64"` or `"root_fs": "docker:///cloudfoundry/trusty64"`.

#### `env` [optional]

Diego supports the notion of container-level environment variables.  All processes that run in the container will inherit these environment variables.

For more details on the environment variables provided to processes in the container, read [Container Runtime Environment](environment.md)

#### Container Limits

#### `cpu_weight` [optional]

To control the CPU shares provided to a container, set `cpu_weight`.  This must be a positive number in the range `1-100`.  The `cpu_weight` enforces a relative fair share of the CPU among containers.  It's best explained with examples.  Consider the following scenarios (we shall assume that each container is running a busy process that is attempting to consumer as many CPU resources as possible):

- Two containers, with equal values of `cpu_weight`: both containers will receive equal shares of CPU time.
- Two containers, one with `cpu_weight=50` the other with `cpu_weight=100`: the latter will get (roughly) 2/3 of the CPU time, the former 1/3.

#### `disk_mb` [optional]

A disk quota applied to the entire container.  Any data written on top of the RootFS counts against the Disk Quota.  Processes that attempt to exceed this limit will not be allowed to write to disk.

- `disk_mb` must be an integer > 0
- The units are megabytes

#### `memory_mb` [optional]

A memory limit applied to the entire container.  If the aggregate memory consumption by all processs running in the container exceeds this value, the container will be destroyed.

- `memory_mb` must be an integer > 0
- The units are megabytes

#### Actions

When an LRP instance is instantiated, a container is created with the specified `root_fs` mounted.  Diego is responsible for performing any container setup necessary to successfully launch processes and monitor said processes.

#### `setup` [optional]

After creating a container, Diego will first run the action specified in the `setup` field.  This field is optional and is typically used to download files and run (short-lived) processes that configure the container.  For more details on the available actions see [actions](actions.md).

- If the `setup` action fails the `ActualLRP` is considered to have crashed and will be restarted

#### `action` [required]

After completing any `setup` action, Diego will launch the `action` action.  This `action` is intended to launch any long running processes.  For more details on the available actions see [actions](actions.md).

#### `monitor` [optional]

If provided, Diego will monitor the long running processes encoded in `action` by periodically invoking the `monitor` action.  If the `monitor` action returns succesfully (exit status code 0), the container is deemed "healthy", otherwise the container is deemed "unhealthy".  Monitoring is quite flexible in Diego and is outlined in more detail [below](#monitoring-health).

#### `start_timeout` [optional]

If provided, Diego will give the `action` action up to `start_timeout` seconds to become healthy before marking the LRP as failed.

#### Networking

Diego can open and expose arbitrary `ports` inside the container.  Currently, if at least one `port` is opened and `routes` are defined, Diego will automatically register the **first** port on the container with the [router](https://github.com/cloudfoundry/gorouter).  Consumers that attempt to access one of the routes in the `routes` array will be connected via the router to one of the ActualLRP instances currently running.

There are plans to generalize this interface and make it possible to build custom service discovery solutions on top of Diego.  The API is likely to change in backward-incompatible ways as we work these requirements out.

#### `ports` [optional]

`ports` is a list of ports to open in the container.  Processes running in the container can bind to these ports to receive incoming traffic.  These ports are only valid within the container namespace and an arbitrary host-side port is created when the container is created.  This host-side port is made available on the `ActualLRP`.

#### `routes` [optional]

`routes` are a list of fully qualified domain names (e.g. `"foo.example.com"`).  These routes are automatically registered with the router and point to the *first* port in the `ports` list.

#### Logging

Diego uses [loggregator](https://github.com/cloudfoundry-incubator/loggregator) to emit logs generated by container processes to the user.

#### `log_guid` [optional]

`log_guid` controls the loggregator guid associated with logs coming from LRP processes.  One typically sets the `log_guid` to the `process_guid` though this is not strictly necessary.

#### `log_source` [optional]

`log_source` is an identifier emitted with each log line.  Individual `RunAction`s can override the `log_source`.  This allows a consumer of the log stream to distinguish between the logs of different processes.

#### Attaching Arbitrary Metadata

#### `annotation` [optional]

Diego allows arbitrary annotations to be attached to a DesiredLRP.  The annotation must not exceed 10 kilobytes in size.

## Updating DesiredLRPs

Only a subset of the DesiredLRP's fields may updated dynamically.  In particular, changes that require the process to be restarted are not allowed - instead, you should submit a new DesiredLRP and orchestrate the upgrade path from one LRP to the next.  This provides the consumer of Diego the flexibility to pick the most appropriate upgrade strategy (blue-green, etc...)

It is possible, however, to dynamically modify the number of instances, and the routes associated with the LRP.  Diego's API makes this explicit -- when updating a DesiredLRP you provide a `DesiredLRPUpdateRequest`:

```
{
    "instances": 17,
    "routes": ["a.example.com", "b.example.com"],
    "annotation": "arbitrary metadata"
}
```

These may be provided simultaneously in one request, or independendantly over several requests.

> Note, while it is recommended that an update to a `DesiredLRP` be explicitly provided as a `DesiredLRPUpdateRequest`, the API does allow consumers to re-`POST` an existing `DesiredLRP`.  In such cases, the API will validate that only the mutable fields (i.e. fields also present in `DesiredLRPUpdateRequest`) have changed.

### Monitoring Health

It is up to the consumer to tell Diego how to monitor an LRP instance.  If provided, Diego uses the `monitor` action to ascertain when an LRP is up.

Typically, an ActualLRP instance begins in an unhealthy state (`STARTING`).  At this point the `monitor` action is polled every 0.5 seconds.  Eventually the `monitor` action succeeds and the instance enters a healthy state (`RUNNING`).  At this point the `monitor` action is polled every 30 seconds.  If the `monitor` action subsequently fails, the ActualLRP is considered crashed.  Diego's consumer is free to define an arbitrary `monitor` action - a `monitor` action may check that a port is accepting connections, or that a URL returns a happy status code, or that a file is present in the container.  In fact, a single `monitor` action might be a composition of other actions that can monitor multiple processes running in the container.

Normally, the `action` action on the DesiredLRP does not exit.  It is possible, however, to launch and daemonize a process in Diego.  If the `action` action exits succesfully Diego assumes the process is a daemon and continues monitoring it with the `monitor` action.  If the `action` action fails (e.g. exit with non-zero status code for a `RunAction`) Diego assumes the ActualLRP has failed and schedules it to be restarted.

Finally, it is possible to opt out of monitoring.  If no `monitor` action is specified then the health of the ActualLRP is dependent on the `action` continuing to run indefinitely.  The ActualLRP is considered `RUNNING` as soon as the `action` action begins, and is considered to have failed if the `action` action ever exits.

> Note that Diego does not currently stream back logs for processes that daemonize.

### Fetching DesiredLRPs

Diego allows consumers to fetch DesiredLRPs -- the response object (`DesiredLRPResponse`) is identical to the `DesiredLRPCreateRequest` object described above.

When fetching DesiredLRPs one can fetch *all* DesiredLRPs in Diego, all DesiredLRPs of a given `domain`, and a specific DesiredLRP by `process_guid`.

The fact that a DesiredLRP is present in Diego does not mean that the corresponding ActualLRP instances are up and running.  Diego converges on the desired state and starting/stopping ActualLRPs may take time.  The presence of a DesiredLRP in Diego signifies the consumer's intent for Diego to run instances - not that those instances are currently running.  For that you must fetch the ActualLRPs.

## Fetching ActualLRPs

As outlined above, DesiredLRPs represent the consumer's intent for Diego to run instances.  To fetch running instances consumers must [fetch ActualLRPs](api_lrps.md#fetching-actuallrps).

When fetching ActualLRPs one can fetch *all* ActualLRPs in Diego, all ActualLRPs of a given `domain`, all ActualLRPs for a given DesiredLRP by `process_guid`, and all ActualLRPs at a given *index* for a given `process_guid`.

In all cases, the consumer is given an array of `ActualLRPResponse`:

```
[
    {
        "process_guid": "some-process-guid",
        "instance_guid": "some-instnace-guid",
        "cell_id": "some-cell-id",
        "domain": "some-domain",
        "index": 15,
        "state": "STARTING" or "RUNNING"

        "host": "10.10.11.11",
        "ports": [
            {"container_port": 8080, "host_port": 60001},
            {"container_port": 5000, "host_port": 60002},        
        ],
    },
    ...
]
```

Let's describe each of these fields in turn.

#### ActualLRP Identifiers

#### `process_guid`

The `process_guid` for this ActualLRP -- this is used to correlate ActualLRPs with DesiredLRPs.

#### `instance_guid`

An arbitrary identifier unique to this ActualLRP instance.

#### `cell_id`

The identifier of the Diego Cell running the ActualLRP instance.

#### `domain`

The `domain` associated with this ActualLRP's DesiredLRP.

#### `index`

The `index` of the ActualLRP - an integer between `0` and `N-1` where `N` is the desired number of instances.

#### `state`

The state of the ActualLRP.  When an ActualLRP is first scheduled onto a Cell it enters the `STARTING` state.  During this time a container is being created and the various processes inside the container are being spun up.

Once the `action` action begins running, Diego begins periodically running the `monitor` action.  As soon as the `monitor` action reports that the processes are healthy the ActualLRP will transition into the `RUNNING` state.

#### Networking
#### `host`

`host` contains the externally accessible IP of the host running the container.

> `host` is only populated when the ActualLRP enters the `RUNNING` state.

#### `ports`

`ports` is an array containing mappings between the `container_port`s requested in the DesiredLRP and the `host_port`s associated with said `container_port`s.  In the example above to connect to the process bound to port `5000` inside the container, a request must be made to `10.10.11.11:60002`.

> `ports` is only populated when the ActualLRP enters the `RUNNING` state.

### Killing ActualLRPs

Diego supports killing the `ActualLRP`s for a given `process_guid` at a given `index`.  This is documented [here](api_lrps.md#killing-actuallrps).  Note that this does not change the *desired* state -- Diego will simply shut down the `ActualLRP` at the given `index` and will eventually converge on desired state by restarting the (now-missing) instance.  To permanently scale down a DesiredLRP you must update the `instances` field on the DesiredLRP.

## Freshness

Diego periodically compares desired state (the set of DesiredLRPs) to actual state (the set of ActualLRPs) and takes actions to keep the actual state in sync with the desired state.  This eventual consistency model is at the core of Diego's robustness.

In order to perform this responsibility safely, however, Diego must have some way of knowing that it's knowledge of the desired state is complete and up-to-date.  In particular, consider a scenario where Diego's database has crashed and must be repopulated.  In this context it is possible to enter a state where the actual state (the ActualLRPs) are known to Diego but the desired state (the DesiredLRPs) is not.  It would be catastrophic for Diego to attempt to converge by shutting down all actual state!

To circumvent this, it is up to the consumer of Diego to inform Diego that its knowledge of the desired state is up-to-date.  We refer to this as the "freshness" of the desired state.  Consumers explicitly mark desired state as *fresh* on a domain-by-domain basis.  Failing to do so will prevent Diego from taking actions to ensure eventual consistency (in particular, Diego will refuse to stop extra instances with no corresponding desired state).

To maintain freshness you perform a simple [POST](api_lrps.md#freshness).  The consumer typically supplies a TTL and attempts to bump the freshness of the domain before the TTL expires (verifying along the way, of course, that the contents of Diego's DesiredLRP are up-to-date).

It is possible to opt out of this by posting freshness with *no* TTL.  In this case the freshness will never expire and Diego will always perform all its eventual consistency operations.

> Note: only destructive operations performed during an eventual consistency convergence cycle are gated on freshness.  Diego will continue to start/stop instances when explicitly instructed to.

[back](README.md)