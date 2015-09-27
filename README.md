# Diego Design Notes

These are design notes intended to convey how the various components of Diego communicate and interrelate.  It is not comprehensive and is not guaranteed to be up-to-date.  If you find something that you suspect is not up-to-date please [open an issue](https://github.com/cloudfoundry-incubator/diego-design-notes/issues).

## Migrating to Diego

Diego is getting close to production readiness!  We've put together some [guidelines](migrating-to-diego.md) around transitioning applications off of the DEAs and on to Diego. One reason to move your apps to Diego is to [try out SSH access to your CF app instances and Diego LRPs](ssh-access-and-policy.md).


## What does Diego do?

Diego schedules and runs *Tasks* and *Long-Running Processes*:

- A **Task** is guaranteed to be run *at most once*.

- A **Long-Running Process** (LRP) may have multiple instances. Diego is told of the *desired LRPs*. Each desired LRP may desire multiple instances, which Diego represents as *actual LRPs*. Diego attempts to keep the correct number of instances running in the face of network failures and crashes.

Clients submit, update, and retrieve Tasks and LRPs to the [BBS](https://github.com/cloudfoundry-incubator/runtime-schema) (Bulletin Board System) via an RPC-style API over HTTP. Diego's [Auctioneer](https://github.com/cloudfoundry-incubator/auctioneer) optimally distributes Tasks and LRPs to the cluster of Diego Cells via an [Auction](https://github.com/cloudfoundry-incubator/auction) that queries and then sends work to the Cell [Rep](https://github.com/cloudfoundry-incubator/rep)s. Once the auction assigns a Task or LRP to a Cell, the [Executor](https://github.com/cloudfoundry-incubator/executor) creates a [Garden](https://github.com/cloudfoundry-incubator/garden) container and executes the work encoded in the Task/LRP. This work is encoded as a generic, platform-independent recipe of composable [actions](https://github.com/cloudfoundry-incubator/receptor/blob/master/doc/actions.md).

The BBS also provides a real-time representation of the state of the Diego cluster (including all desired LRPs, running LRP instances, and in-flight Tasks). The [Converger](https://github.com/cloudfoundry-incubator/converger) periodically analyzes snapshots of this representation and corrects discrepancies, ensuring that Diego is eventually consistent.

Diego sends real-time streaming logs for Tasks/LRPs to the [Loggregator](https://github.com/cloudfoundry/loggregator) system.  Diego also registers its running LRP instances with the [Gorouter](https://github.com/cloudfoundry/gorouter) to route external web traffic to them.

Diego is the next-generation runtime powering Cloud Foundry (CF), but Diego is abstracted away from CF: CF simply acts as another Diego client via the BBS API.  For now, there is a translation layer called the CC-Bridge that converts the [Cloud Controller](https://github.com/cloudfoundry/cloud_controller_ng)'s domain-specific requests to stage and run applications into requests for Tasks and LRPs.  Eventually Cloud Controller will be modified to communicate directly with the BBS.  The process of staging and running a CF application is complex and filled with platform and implementation-specific details.  We encapsulate these concerns in a collection of binaries known collectively as the [App Lifecycle](#app-lifecycles).  The Tasks and LRPs produced by the CC-Bridge download the App Lifecycle binaries and run them to stage, start, and health-check CF applications.


## CF Summit Talks

- 2014 CF Summit talk on Diego: [YouTube video](https://www.youtube.com/watch?v=1OkmVTFhfLY) &middot; [Apple Keynote slides](https://drive.google.com/file/d/0B55cOnKV7PrQaTBKRjg4MjE1Ujg/view?usp=sharing) &middot; [SlideShare slides](http://www.slideshare.net/Pivotal/cloud-foundry-summit-2014-diego-reenvisioning-the-elastic-runtime)


## What are all these repos and what do they do?

Here's a diagrammatic overview (*warning: diagram is out of date*).  Ingest it slowly as you read through this section.

[![Diego Overview](./diego-overview.png)](http://htmlpreview.github.io/?https://raw.githubusercontent.com/cloudfoundry-incubator/diego-design-notes/master/clickable-diego-overview/clickable-diego-overview.html)

Here's a [PDF](https://github.com/cloudfoundry-incubator/diego-design-notes/raw/master/diego-overview.pdf) version.

Here's a [clickable image map](http://htmlpreview.github.io/?https://raw.githubusercontent.com/cloudfoundry-incubator/diego-design-notes/master/clickable-diego-overview/clickable-diego-overview.html)

This diagram includes all the major repositories/components associated with Diego.

Components in the blue region are part of the Diego core and handle the running and monitoring of Tasks and LRPs.  These components all live in the [Diego-Release](https://github.com/cloudfoundry-incubator/diego-release) BOSH release.

Components in the yellow region bring support for streaming logs and routing to Diego containers.  Some of these components live in [Diego-Release](https://github.com/cloudfoundry-incubator/diego-release) while others live in [CF-Release](https://github.com/cloudfoundry/cf-release).  The [Lattice](https://github.com/cloudfoundry-incubator/lattice) distribution includes these components and offers developers an easy-to-install environment for interacting with Diego.

The red region brings in [Cloud Controller](https://github.com/cloudfoundry/cloud_controller_ng) and the CC-Bridge.  As the diagram shows, the CC-Bridge merely interfaces with the BBS, translating app-specific messages from the CC to the more generic language of Tasks and LRPs.

The following summarizes the roles and responsibilities of the various components in this diagram.

#### "User-facing" Components

These "user-facing" components all live in [cf-release](https://github.com/cloudfoundry/cf-release):

- [**Cloud Controller**](https://github.com/cloudfoundry/cloud_controller_ng) (CC)
    - provides an API for staging and running Apps.
    - implements all the object modelling around Apps (permissions, buildpack selection, service binding, etc...).
    - Developers interact with the cloud controller via the [CLI](https://github.com/cloudfoundry/cli)
- [**Doppler/Loggregator Traffic Controller**](https://github.com/cloudfoundry/loggregator)
    - aggregates logs and streams them to developers
- [**Router**](https://github.com/cloudfoundry/gorouter)
    - routes incoming network traffic to processes within the CF installation
    - this includes routing traffic to both developer apps running within Garden containers and CF components such as CC.

### CC-Bridge Components

The CC-Bridge components interact with the Cloud Controller.  They serve, primarily, to translate app-specific notions into the generic language of LRPs and Tasks:

- [**Stager**](https://github.com/cloudfoundry-incubator/stager)
    - receives staging requests from CC
    - translates these requests into generic Tasks and submits the Tasks to the BBS
        - this Task instructs the Cell (via the Task actions) to inject a platform-specific binary to perform the actual staging process (see [below](#platform-specific-components))
    - sends a response to CC when a Task is completed (succesfully or otherwise).
- [**CC-Uploader**](https://github.com/cloudfoundry-incubator/cc-uploader)
    - mediates staging uploads from the Executor to CC, translating the Executor's simple HTTP POST into the complex multipart-form upload CC requires.
- [**Nsync**](https://github.com/cloudfoundry-incubator/nsync) splits its responsibilities between two independent processes:
    - The **nsync-listener** listens for desired app requests and updates/creates the desired LRPs via the BBS.
    - The **nsync-bulker** periodically polls CC for all desired apps to ensure the desired state known to Diego is up-to-date.
- [**TPS**](https://github.com/cloudfoundry-incubator/tps) also splits its responsibilities between two independent processes:
    - The **tps-listener** provides the CC with information about running LRP instances for `cf apps` and `cf app X` requests.
    - The **tps-watcher** monitors `ActualLRP` activity for crashes and reports them to CC.

### Components on the Database VMs

The Database VMs provide Diego's core components and clients a consistent API to the shared state and operations that manage Tasks and LRPs, as well as the data store for that shared state.

- [**BBS**](https://github.com/cloudfoundry-incubator/stager)
    - provides an RPC-style API over HTTP to both core Diego components (rep, auctioneer, converger) and external clients (receptor, SSH proxy, CC-bridge, route emitter).
    - encapsulates access to the backing database and manages data migrations, encoding, and encryption.
- [**ETCD**](https://github.com/coreos/etcd)
    - is Diego's consistent key-value data store.


### Components on the Cell

These Diego components deal with running and maintaining generic Tasks and LRPs:

- [**Rep**](https://github.com/cloudfoundry-incubator/rep)
    - represents a Cell and mediates all communication with the BBS by:
        + ensuring the set of Tasks and ActualLRPs in the BBS is in sync with the containers actually present on the Cell
        + maintaining the presence of the Cell in the BBS.  Should the Cell fail catastrophically, the Converger will automatically move the missing instances to other Cells.
    - participates in [auctions](https://github.com/cloudfoundry-incubator/auction) to accept Tasks/LRPs
    - runs Tasks/LRPs by asking its in-process Executor to create a container and run generic action recipes in said container.
- [**Executor**](https://github.com/cloudfoundry-incubator/executor) (now a logical process running inside the Rep)
    - the Executor doesn't know about the Task vs LRP distinction.  It is primarily responsible for implementing the generic executor actions detailed in the [API documentation](https://github.com/cloudfoundry-incubator/receptor/blob/master/doc/actions.md)
    - the Executor streams Stdout and Stderr to the metron-agent running on the Cell.  These then get forwarded to Loggregator.
- [**Garden**](https://github.com/cloudfoundry-incubator/garden)
    - provides a platform-independent server/client to manage garden containers
    - defines an interface to be implemented by container-runners (e.g. [garden-linux](https://github.com/cloudfoundry-incubator/garden-linux))
- [**Metron**](https://github.com/cloudfoundry/loggregator/tree/develop/src/metron)
    - Forwards application logs and application/Diego metrics to [doppler](https://github.com/cloudfoundry/loggregator)

Note that there is a specificity gradient across the Rep/Executor/Garden.  The Rep is concerned with Tasks and LRPs and knows details about their lifecycles.  The Executor knows nothing about Tasks/LRPs but merely knows how to manage a collection of containers and run actions in these containers.  Garden, in turn, knows nothing about actions and simply provides a concrete implementation of a platform-specific containerization technology that can run arbitrary commands in containers.

Only the Rep communicates with the BBS.

### Components on the Brain

- [**Auctioneer**](https://github.com/cloudfoundry-incubator/auctioneer)
    - holds auctions for Tasks and ActualLRP instances.
    - auctions are run using the [auction](https://github.com/cloudfoundry-incubator/auction) package.  Auction communication goes over HTTP and is between the Auctioneer and the Cell Reps.
    - maintains a lock in the BBS such that ***only one*** auctioneer may handles auctions at a time.
- [**Converger**](https://github.com/cloudfoundry-incubator/converger)
    - maintains a lock in the BBS to ensure that ***only one*** converger performs convergence.  This is primarily for performance considerations.  Convergence should be idempotent.
    - uses the converge methods in the runtime-schema/bbs to ensure eventual consistency and fault tolerance for Tasks and LRPs
    - when converging LRPs, the converger identifies which actions need to take place to bring DesiredLRP state and ActualLRP state into accord.  Two actions are possible:
        - if an instance is missing, a start auction is sent.
        - if an extra instance is identified, a stop message is sent to the Rep on the Cell hosting the instance.
    - in addition, the converger watches out for any potentially missed messages.  For example, if a Task has been in the PENDING state for too long it's possible that the request to hold an auction for the Task never made it to the Auctioneer.  In this case the Converger is responsible for resending the auction message.
    - periodically sends aggregate metrics about DesiredLRPs, ActualLRPs, and Tasks to Doppler.

### Components on the Access VMs

- [**File-Server**](https://github.com/cloudfoundry-incubator/file-server)
    - serves static assets used by our various components.  In particular, it serves the App Lifecycle binaries (see [below](#app-lifecycles)).
- [**SSH Proxy**](https://github.com/cloudfoundry-incubator/diego-ssh)
    - brokers connections between SSH clients and SSH servers running inside instance containers



### Additional (shim-like) components

- [**Route-Emitter**](https://github.com/cloudfoundry-incubator/route-emitter)
    - monitors DesiredLRP state and ActualLRP state via the BBS.  When a change is detected, the Route-Emitter emits route registration/unregistration messages to the [router](https://github.com/cloudfoundry/gorouter)
    - periodically emits the entire routing table to the router.


### Platform-Specific Components

Diego is largely platform-agnostic.  All platform-specific concerns are delegated to two types of components: the *garden backends* and the *app lifecycles*.

#### Garden Backends

[**Garden**](https://github.com/cloudfoundry-incubator/garden) contains a set of interfaces each platform-specific backend must implement. These interfaces contain methods to perform the following actions:

- create/delete containers
- apply resource limits to containers
- open and attach network ports to containers
- copy files into/out of containers
- run processes within containers, streaming back stdout and stderr data
- annotate containers with arbitrary metadata
- snapshot containers for down-timeless redeploys

Current implementations:

- [**Garden-Linux**](https://github.com/cloudfoundry-incubator/garden-linux) provides a linux-specific implementation of a Garden interface.

#### App Lifecycles

Each App Lifecycle provides a set of binaries that manage a *Cloud Foundry*-specific application lifecycle.  There are three binaries:

- The **Builder** *stages* a CF application.  The CC-Bridge runs the Builder as a Task on every staging request.  The Builder perfoms static analysis on the application code and does any necessary pre-processing before the application is first run.
- The **Launcher** *runs* a CF application.  The CC-Bridge sets the Launcher as the Action on the CF application's DesiredLRP.  The Launcher executes the user's start command with the correct system context (working directory, environment variables, etc).  
- The **Healthcheck** performs a status check of running CF application from inside the container.  The CC-Bridge sets the Healthcheck as the Monitor action on the CF application's DesiredLRP. 

Current implementations:

- [**Buildpack-App-Lifecycle**](https://github.com/cloudfoundry-incubator/buildpack-app-lifecycle) implements a traditional buildpack-based lifecycle.
- [**Docker-App-Lifecycle**](https://github.com/cloudfoundry-incubator/docker-app-lifecycle) implements a docker-based lifecycle.

### Bringing it all together

Diego is made of very many disparate components.  Ensuring that these components work together correctly is a challenge addressed by these entities:

- [**Inigo**](https://github.com/cloudfoundry-incubator/inigo)
    - is an integration test suite that launches the various Diego components and excercises them through various test cases.  As such, Inigo *validates* that a given set of component versions are mutually compatible.
    - in addition to excercising various *non-exceptional* test cases, Inigo can excercise exceptional cases (e.g. when a component fails).
- [**Auction**](https://github.com/cloudfoundry-incubator/auction)
    - encodes the behavioral details around the auction.
    - includes a simulation test suite that validates the correctness and performance of the auction algorithm.  The simulation can be run for different algorithms, at different scales.  The simulation can either be run in-process (for quick feedback loops) or across multiple processes (to understand the role of communication in the auction) or even across multiple machines in a cloud-like infrastructure (to understand the impact of latency on the auction).
    - the auctioneer and rep use the auction package to participate in the auction.
- [**Diego-Acceptance-Tests**](https://github.com/cloudfoundry-incubator/diego-acceptance-tests)
    - are a suite of acceptance-level tests that run against a deployment of CF release and Diego release.
    - these exercise a number of happy-path test cases across the entire stack.
    - use the CF cli to run the tests.


### The Release

[**Diego-Release**](https://github.com/cloudfoundry-incubator/diego-release) packages Diego as a BOSH release. its [README](https://github.com/cloudfoundry-incubator/diego-release) includes detailed instructions for getting a BOSH-lite deployment up and running.

Diego-Release is also the **canonical** `GOPATH` for the Diego. All Diego development takes place inside the Diego-Release directory.


### Other Components

- [**Consul**](https://github.com/hashicorp/consul)
    - provides a dynamic service registration and load balancing via DNS resolution.
    - provides a consistent key-value store for maintenance of distributed locks and component presence.
- [**Consuladapter**](https://github.com/cloudfoundry-incubator/consuladapter)
    - provides a driver for interfacing with etcd.
