# Diego Design Notes

These are design notes intended to convey how the various components of Diego communicate and interrelate.  It is not comprehensive and is generally up-to-date, although not guaranteed to be.  If you find something that you suspect is not up-to-date, please [open an issue](https://github.com/cloudfoundry/diego-design-notes/issues) on this repository.


## Migrating to Diego

We've put together some [guidelines](migrating-to-diego.md) around transitioning applications off of the DEAs and on to Diego. One reason to move your apps to Diego is to [try out SSH access to your CF app instances and Diego LRPs](ssh-access-and-policy.md).


## What does Diego do?

Diego schedules and runs *Tasks* and *Long-Running Processes*:

- A **Task** is guaranteed to be run *at most once*.

- A **Long-Running Process** (LRP) may have multiple instances. Diego is told of the *desired LRPs*. Each desired LRP may desire multiple instances, which Diego represents as *actual LRPs*. Diego attempts to keep the correct number of instances running in the face of network failures and crashes.

Clients submit, update, and retrieve Tasks and LRPs to the [BBS](https://github.com/cloudfoundry/bbs) (Bulletin Board System) via an RPC-style API over HTTP. Diego's [Auctioneer](https://github.com/cloudfoundry/auctioneer) optimally distributes Tasks and LRPs to the cluster of Diego Cells via an [Auction](https://github.com/cloudfoundry/auction) that queries and then sends work to the Cell [Rep](https://github.com/cloudfoundry/rep)s. Once the auction assigns a Task or LRP to a Cell, the [Executor](https://github.com/cloudfoundry/executor) creates a [Garden](https://github.com/cloudfoundry/garden) container and executes the work encoded in the Task/LRP. This work is encoded as a generic, platform-independent recipe of composable [actions](https://github.com/cloudfoundry/bbs/blob/master/doc/actions.md).

The BBS also provides a real-time representation of the state of the Diego cluster (including all desired LRPs, running LRP instances, and in-flight Tasks). The [Converger](https://github.com/cloudfoundry/converger) periodically analyzes snapshots of this representation and corrects discrepancies, ensuring that Diego is eventually consistent.

Diego sends real-time streaming logs for Tasks/LRPs to the [Loggregator](https://github.com/cloudfoundry/loggregator) system.  Diego also registers its running LRP instances with the [Gorouter](https://github.com/cloudfoundry/gorouter) to route external web traffic to them.

Diego is the next-generation runtime powering Cloud Foundry (CF), but Diego is abstracted away from CF: CF simply acts as another Diego client via the BBS API.  For now, there is a translation layer called the CC-Bridge that converts the [Cloud Controller](https://github.com/cloudfoundry/cloud_controller_ng)'s domain-specific requests to stage and run applications into requests for Tasks and LRPs.  Eventually Cloud Controller will be modified to communicate directly with the BBS.  The process of staging and running a CF application is complex and filled with platform and implementation-specific details.  A collection of binaries known collectively as the [App Lifecycle](#app-lifecycles) encapsulate these concerns. The Tasks and LRPs produced by the CC-Bridge download the App Lifecycle binaries and execute them to stage, to run, and to health-check CF applications.


## CF Summit Talks on Diego

- 2016: [YouTube video](https://www.youtube.com/watch?v=iv5EpheLLh0) &middot; [Apple Keynote slides](https://drive.google.com/file/d/0Bw2c1Jc_v7t1T0dZZ2l3ZWJ5SHM/view?usp=sharing) &middot; [PDF slides](https://drive.google.com/file/d/0Bw2c1Jc_v7t1WlM4U09WVUE4bWc/view)
- 2015: [YouTube video](https://www.youtube.com/watch?v=SSxI9eonBVs)
- 2014: [YouTube video](https://www.youtube.com/watch?v=1OkmVTFhfLY) &middot; [Apple Keynote slides](https://drive.google.com/file/d/0B55cOnKV7PrQaTBKRjg4MjE1Ujg/view?usp=sharing) &middot; [SlideShare slides](http://www.slideshare.net/Pivotal/cloud-foundry-summit-2014-diego-reenvisioning-the-elastic-runtime)


## What are all these repos and what do they do?

Below is a diagrammatic overview of the major repositories and components in Diego and CF (also [PDF](https://github.com/cloudfoundry/diego-design-notes/raw/master/diego-overview.pdf) &middot; [clickable map](http://htmlpreview.github.io/?https://raw.githubusercontent.com/cloudfoundry/diego-design-notes/master/clickable-diego-overview/clickable-diego-overview.html)).

[![Diego Overview](./diego-overview.png)](http://htmlpreview.github.io/?https://raw.githubusercontent.com/cloudfoundry/diego-design-notes/master/clickable-diego-overview/clickable-diego-overview.html)

Components in the blue region are part of the Diego core and handle the running and monitoring of Tasks and LRPs.  These components all come from the [Diego BOSH release](https://github.com/cloudfoundry/diego-release).

Components in the yellow region provide infrastructure support to Diego and CF components. At the moment, this primarily includes [Consul](https://github.com/hashicorp/consul) for DNS-based dynamic service discovery and a consistent key-value store for distributed locks and component discovery.

Components in the orange region support routing HTTP traffic to Diego containers. This includes the [Route-Emitter](https://github.com/cloudfoundry/route-emitter) from Diego and the [Gorouter](https://github.com/cloudfoundry/gorouter) from CF.

Components in the red region support log and metric aggregation from Diego containers and CF and Diego components.

The green region brings in [Cloud Controller](https://github.com/cloudfoundry/cloud_controller_ng) and the CC-Bridge.  As the diagram shows, the CC-Bridge merely interfaces with the BBS, translating app-specific messages from the CC to the more generic language of Tasks and LRPs.

The following summarizes the roles and responsibilities of the various components in this diagram.

#### "User-facing" Components

These "user-facing" components all live in [cf-release](https://github.com/cloudfoundry/cf-release):

- [**Cloud Controller**](https://github.com/cloudfoundry/cloud_controller_ng) (CC):
    - provides an API for staging and running apps and provisioning and binding services to them,
    - organizes apps and services into a hierarchy with role-based access control suitable for a multi-tenant platform.
- [**Loggregator**](https://github.com/cloudfoundry/loggregator):
    - Doppler aggregates app logs and component and container metrics relayed through the local Metron agents.
    - Traffic Controller retrieves logs and metrics from Doppler for end users, enforcing access based on CC roles
- [**Gorouter**](https://github.com/cloudfoundry/gorouter):
    - routes incoming HTTP traffic to processes within the CF/Diego deployment
    - this includes routing traffic to both developer apps running within Garden containers and CF components such as CC.

Developers typically interact with CC and the logging system through a client such as the [CF CLI](https://github.com/cloudfoundry/cli).

### CC-Bridge Components

The CC-Bridge components interact with the Cloud Controller.  They serve primarily to translate app-specific notions into the more general notions of LRPs and Tasks:

- [**Stager**](https://github.com/cloudfoundry/stager):
    - receives staging requests from CC, translates them into Diego Tasks, and submits those Tasks to the BBS
    - sends a response to CC when a staging Task is completed, successfully or otherwise.
- [**CC-Uploader**](https://github.com/cloudfoundry/cc-uploader):
    - mediates staging uploads from the Executor to CC, translating the Executor's simple HTTP POST into the complex multipart-form upload CC requires.
- [**Nsync**](https://github.com/cloudfoundry/nsync) splits its responsibilities between two independent processes:
    - The **nsync-listener** listens for desired app requests and updates/creates the desired LRPs via the BBS.
    - The **nsync-bulker** periodically polls CC for all desired apps to ensure the desired state known to Diego is up-to-date.
- [**TPS**](https://github.com/cloudfoundry/tps) also splits its responsibilities between two independent processes:
    - The **tps-listener** provides the CC with information about running LRP instances for `cf apps` and `cf app X` requests.
    - The **tps-watcher** monitors `ActualLRP` activity for crashes and reports them to CC.

Many of the CC-Bridge components are inherently stateless and will eventually be consolidated into Cloud Controller itself.


### Components on the Database VMs

The Database VMs provide Diego's core components and clients a consistent API to the shared state and operations that manage Tasks and LRPs, as well as the data store for that shared state.

- [**BBS**](https://github.com/cloudfoundry/bbs):
    - provides an RPC-style API over HTTP to both core Diego components (rep, auctioneer, converger) and external clients (CC-Bridge, route emitter, SSH proxy),
    - encapsulates access to the backing database and manages data migrations, encoding, and encryption,
    - maintains a lock in consul to ensure ***only one*** BBS handles requests and migrations at a time.


The BBS requires a backing persistent data store. Currently this is [**etcd**](https://github.com/coreos/etcd), a consistent key-value store based on the Raft concensus algorithm.


### Components on the Cell

These Diego components run and monitor Tasks and LRPs in Garden containers:

- [**Rep**](https://github.com/cloudfoundry/rep):
    - maintains a presence record for the Cell in the BBS,
    - participates in [auctions](https://github.com/cloudfoundry/auction) to accept new Tasks and LRP instances,
    - runs Tasks and LRPs by telling its in-process Executor to create a container and then to run actions in it,
    - reacts to container events coming from the Executor,
    - periodically ensures its set of Tasks and ActualLRPs in the BBS is in sync with the containers actually present on the Cell.
- [**Executor**](https://github.com/cloudfoundry/executor) (now a logical process running inside the Rep):
    - manages container allocations against resource constraints on the Cell, such as memory and disk space,
    - implements the actions detailed in the [API documentation](https://github.com/cloudfoundry/bbs/blob/master/doc/actions.md),
    - streams stdout and stderr from container processes to the metron-agent running on the Cell, which in turn forwards to the Loggregator system,
    - periodically collects container metrics and emits them to Loggregator.
- [**Garden**](https://github.com/cloudfoundry/garden)
    - provides a platform-independent server and client to manage garden containers,
    - defines an interface to be implemented by container-runners, such as [guardian](https://github.com/cloudfoundry/guardian) and [garden-windows](https://github.com/cloudfoundry/garden-windows).
- [**Metron**](https://github.com/cloudfoundry/loggregator/tree/develop/src/metron)
    - forwards application logs and application and component metrics to [doppler](https://github.com/cloudfoundry/loggregator)

Note that there is a specificity gradient across the Rep, the Executor, and Garden. The Rep is concerned with Tasks and LRPs and knows details about their lifecycles. The Executor knows only how to manage a collection of containers and to run actions in these containers. Garden knows nothing about actions and simply provides a concrete implementation of a platform-specific containerization technology that can run arbitrary commands in containers.


### Components on the Brain

- [**Auctioneer**](https://github.com/cloudfoundry/auctioneer)
    - holds auctions for Tasks and LRP instances.
    - runs auctions using the [auction](https://github.com/cloudfoundry/auction) package.  Auction communication goes over HTTP and is between the Auctioneer and the Cell Reps.
    - maintains a lock in consul to ensure ***only one*** auctioneer handles auctions at a time.
- [**Converger**](https://github.com/cloudfoundry/converger)
    - maintains a lock in consul to ensure that ***only one*** converger performs convergence. This exclusivity is primarily for performance considerations, as convergence is idempotent.
    - compares DesiredLRPs and their ActualLRPs and takes action to enforce the desired state:
        - if an instance is missing or unclaimed for too long, it a new auction is requested.
        - if an extra instance is identified, a stop message is sent to the Rep on the Cell hosting the instance.
    - resends auction requests for Tasks that have been pending for too long and completion callbacks for Tasks that have remained completed for too long,
    - periodically sends aggregate metrics about DesiredLRPs, ActualLRPs, and Tasks to Loggregator.

In practice, much of the convergence processing actually takes place inside the BBS.


### Components on the Access VMs

- [**File-Server**](https://github.com/cloudfoundry/file-server)
    - serves static assets used by our various components, such as the App Lifecycle binaries (see [below](#app-lifecycles)).
- [**SSH Proxy**](https://github.com/cloudfoundry/diego-ssh)
    - brokers connections between SSH clients and SSH servers running inside instance containers,
    - authorizes access to CF app instances based on Cloud Controller roles.


### Routing Translation Components

- [**Route-Emitter**](https://github.com/cloudfoundry/route-emitter)
    - monitors DesiredLRP state and ActualLRP state via the BBS.  When a change is detected, the Route-Emitter emits route registration and unregistration messages to the [gorouter](https://github.com/cloudfoundry/gorouter) via the [NATS](https://github.com/nats-io/gnatsd) message bus,
    - periodically emits the entire routing table to the router,
    - maintains a lock in consul to ensure ***only one*** route-emitter handles route registration at a time.


### Service Registration and Component Coordination

- [**Consul**](https://github.com/hashicorp/consul):
    - provides dynamic service registration and load-balancing via DNS resolution,
    - provides a consistent key-value store for maintenance of distributed locks and component presence.
- [**Locket**](https://github.com/cloudfoundry/locket):
    - provides abstractions for locks and service registration that encapsulate interactions with consul.


### Platform-Specific Components

Diego is largely platform-agnostic.  All platform-specific concerns are delegated to two types of components: the *garden backends* and the *app lifecycles*.

#### Garden Backends

[**Garden**](https://github.com/cloudfoundry/garden) contains a set of interfaces each platform-specific backend must implement. These interfaces contain methods to perform the following actions:

- create/delete containers
- apply resource limits to containers
- open and attach network ports to containers
- copy files into/out of containers
- run processes within containers, streaming back stdout and stderr data
- annotate containers with arbitrary metadata
- snapshot containers for down-timeless redeploys

Current implementations:

- [**Garden-runC (a.k.a. Guardian)**](https://github.com/cloudfoundry/guardian) provides a linux-specific implementation of a Garden interface.
- [**Garden-Windows**](https://github.com/cloudfoundry/garden-windows) provides a Windows-specific implementation of a Garden interface.

#### App Lifecycles

Each App Lifecycle provides a set of binaries that manage a *Cloud Foundry*-specific application lifecycle.  There are three binaries:

- The **Builder** *stages* a CF application.  The CC-Bridge runs the Builder as a Task on every staging request.  The Builder perfoms static analysis on the application code and does any necessary pre-processing before the application is first run.
- The **Launcher** *runs* a CF application.  The CC-Bridge sets the Launcher as the Action on the CF application's DesiredLRP.  The Launcher executes the user's start command with the correct system context (working directory, environment variables, etc.).
- The **Healthcheck** performs a status check of a running CF application from inside the container. The CC-Bridge sets the Healthcheck as the Monitor action on the CF application's DesiredLRP.

Current implementations:

- [**Buildpack-App-Lifecycle**](https://github.com/cloudfoundry/buildpackapplifecycle) implements a buildpack-based lifecycle.
- [**Docker-App-Lifecycle**](https://github.com/cloudfoundry/docker-app-lifecycle) implements a lifecycle to stage and run Docker images as CF apps.
- [**Windows-App-Lifecycle**](https://github.com/cloudfoundry/windows_app_lifecycle) implements a lifecycle for .NET applications on Windows.


### Bringing it all together

CF and Diego consist of many disparate components.  Ensuring that these components work together correctly is a challenge addressed by these entities:

- [**Inigo**](https://github.com/cloudfoundry/inigo):
    - is an integration test suite that launches the various Diego components and exercises them through various test cases.  As such, Inigo *validates* that a given set of component versions are mutually compatible.
    - in addition to exercising various ordinary test cases, Inigo can exercise *exceptional* cases, such as when a component fails or is unavailable for a period, that would be more difficult to orchestrate against a BOSH-deployed Diego cluster.
- [**Vizzini**](https://github.com/cloudfoundry/vizzini):
    - is a suite of acceptance-level tests that run against a deployment of Diego with consul and routing components from CF,
    - interacts directly with the BBS API to run the tests,
    - ensures that Diego executes work and recovers from failure quickly by placing stringent timing requirements on many of the tests.
- [**CF Acceptance Tests**](https://github.com/cloudfoundry/cf-acceptance-tests):
    - is a suite of acceptance-level tests that run against CF and Diego deployed together,
    - uses the [CF CLI](https://github.com/cloudfoundry/cli) to run the tests.
- [**Auction**](https://github.com/cloudfoundry/auction):
    - encodes the behavioral details around the auction.
    - includes a simulation test suite that validates the correctness and performance of the auction algorithm.  The simulation can be run for different algorithms, at different scales.  The simulation can either be run in-process (for quick feedback loops) or across multiple processes (to understand the role of communication in the auction) or even across multiple machines in a cloud-like infrastructure (to understand the impact of latency on the auction).
    - the auctioneer and rep use the auction package to participate in the auction.


### The BOSH Release

[**Diego-Release**](https://github.com/cloudfoundry/diego-release) packages Diego as a BOSH release. Its [README](https://github.com/cloudfoundry/diego-release) includes detailed instructions for deploying CF and Diego to a local [BOSH-Lite](https://github.com/cloudfoundry/bosh-lite).

Diego-Release is also the **canonical** `GOPATH` for the Diego. All Diego development takes place inside the Diego-Release directory.

