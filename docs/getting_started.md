# Getting started

Thanos provides a global query view, historical data access and alerting as
its core features in a single binary. All three features can be run
independently of each other. This allows you to have a subset of Thanos
features ready for immediate benefit or testing, while also making it flexible
for gradual roll outs in more complex environments.

Thanos has the following components:

```
sidecar    - sidecar for Prometheus server: entrypoint for querying data and uploading blocks
             to a block storage (s3, gcs).
store      - store node giving access to blocks in a bucket provider.
query      - query node exposing PromQL enabled Query API with data retrieved from multiple
             store or sidecar nodes.
rule       - ruler evaluating Prometheus rules against given query nodes.
compact    - a component to continuously compacts blocks in an object store bucket.
bucket     - a component to manually handle and fix storage issues.
downsample - a component to continuously downsample blocks in an object store bucket,
             allowing for longer retention periods without losing query performance.
```

These components can be set up with different levels of complexities:

1. [Basic Setup](#basic-setup) - allows you to query against multiple Prometheus, without long term retention.
2. [Long term retention](#long-term-retention) - allows you the same as the basic setup, but introduces block storage to increase your retention arbitrately.
3. [Alerting](#alerting) - allows you to alert against data from multiple Prometheus.

## Basic Setup

Requirements: a

On the basic setup we'll configure Thanos query to query against two Prometheus.


## Get Thanos!

You can find the latest Thanos release [here](https://github.com/improbable-eng/thanos/releases).

If you want to build Thanos from source -
with a working installation of the Go [toolchain](https://github.com/golang/tools) (`GOPATH`, `PATH=${GOPATH}/bin:${PATH}`), Thanos can be downloaded and built by running:

```
go get -d github.com/improbable-eng/thanos/...
cd ${GOPATH}/src/github.com/improbable-eng/thanos
make
```

The `thanos` binary should now be in your `$PATH` and is the only thing required to deploy any of its components.

## [Sidecar](components/sidecar.md)

Thanos integrates with existing Prometheus (v2.2.1+) servers through a [Sidecar process](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar#solution), which runs on the same machine or in the same pod as the Prometheus server.

The purpose of the Sidecar is to backup Prometheus data into an Object Storage bucket, and giving other Thanos components access to the Prometheus instance the Sidecar is attached to. [More details about the Sidecar's functions are available at the sidecar documentation page](components/sidecar.md).

### Backups

The following configures the Sidecar to only backup Prometheus data into a Google Cloud Storage bucket.

```
thanos sidecar \
    --tsdb.path         /var/prometheus \           # Data directory of Prometheus
    --gcs.bucket        example-bucket \            # Bucket to upload data to
```

Rolling this out has little to zero impact on the running Prometheus instance. It is a good start to ensure you are backing up your data while figuring out the other pieces of Thanos.

If you are not interested in backing up any data, the `--gcs.bucket` flag can simply be omitted.

* _[Example Kubernetes manifest](../kube/manifests/prometheus.yaml)_
* _[Example Kubernetes manifest with GCS upload](../kube/manifests/prometheus-gcs.yaml)_
* _[Details & Config for other object stores](./storage.md)_

### [Store API](components/store.md)

The Sidecar component comes with a _[Store API](components/store.md)_. The Store API allows you to query metric data in Prometheus and data backed up into the Object Store bucket.

Let's extend the Sidecar in the previous section to connect to a Prometheus server, and expose the Store API.

```
thanos sidecar \
    --tsdb.path                 /var/prometheus \
    --gcs.bucket                example-bucket \
    --prometheus.url            http://localhost:9090 \    # Location of the Prometheus HTTP server
    --http-address              0.0.0.0:19191 \            # HTTP endpoint for collecting metrics on the Sidecar
    --cluster.peers             127.0.0.1:19391 \          # Static list of peers where the node will get info about the cluster
```

* _[Example Kubernetes manifest](../kube/manifests/prometheus.yaml)_
* _[Example Kubernetes manifest with GCS upload](../kube/manifests/prometheus-gcs.yaml)_

### External Labels
Prometheus allows the configuration of "external labels" of a given Prometheus instance. These are meant to globally identify the role of that instance. As Thanos aims to aggregate data across all instances, providing a consistent set of external labels becomes crucial!

Every Prometheus instance must have a globally unique set of identifying labels. For example, in Prometheus's configuration file:

```
global:
  external_labels:
    region: eu-west
    monitor: infrastructure
    replica: A
# ...
```

## [Query Layer](components/query.md)

Now that we have setup the Sidecar for one or more Prometheus instances, we want to use Thanos' global [Query Layer](components/query.md) to evaluate PromQL queries against all instances at once.

The Query component is stateless and horizontally scalable and can be deployed with any number of replicas. Once connected to the Sidecars, it automatically detects which Prometheus servers need to be contacted for a given PromQL query.

Query also implements Prometheus's offical HTTP API and can thus be used with external tools such as Grafana. It also serves a derivative of Prometheus's UI for ad-hoc querying.

Below, we will set up a Query to connect to our Sidecars, and expose its HTTP UI.

```
thanos query \
    --http-address              0.0.0.0:19192 \         # HTTP Endpoint for Query UI
    --cluster.peers             127.0.0.1:19391 \       # Static cluster peer where the node will get info about the Sidecar cluster
    --cluster.peers             127.0.0.1:19392 \       # Another cluster peer (many can be added to discover nodes)
    --store                     0.0.0.0:18091   \       # Static gRPC Store API Address for the query node to query
    --store                     0.0.0.0:18092   \       # Also repeatable
```

Go to the configured HTTP address that should now show a UI similar to that of Prometheus. If the cluster formed correctly you can now query across all Prometheus instances within the cluster.

### Deduplicating Data from Prometheus HA pairs

The Query component is also capable of deduplicating data collected from Prometheus HA pairs. This requires configuring Prometheus's `global.external_labels` configuration block (as mentioned in the [External Labels section](#external-labels)) to identify the role of a given Prometheus instance.

A typical choice is simply the label name "replica" while letting the value be whatever you wish. For example, you might set up the following in Prometheus's configuration file:

```
global:
  external_labels:
    region: eu-west
    monitor: infrastructure
    replica: A
# ...
```

Reload your Prometheus instances, and then, in Query, we will enable `replica` as the label we want to enable deduplication to occur on:

```
thanos query \
    --http-address              0.0.0.0:19192 \
    --cluster.peers             127.0.0.1:19391 \
    --cluster.peers             127.0.0.1:19392 \
    --store                     0.0.0.0:18091   \
    --store                     0.0.0.0:18092   \
    --query.replica-label       replica \               # Replica label for de-duplication
```

Go to the configured HTTP address, and you should now be able to query across all Prometheus instances and receive de-duplicated data.

* _[Example Kubernetes manifest](../kube/manifests/thanos-query.yaml)_

## Communication Between Components

Components in a Thanos cluster can be connected through a gossip protocol to advertise membership and propagate metadata about other known nodes or by setting static store flags of known components. We added gossip to efficiently and dynamically discover other nodes in the cluster and the metrics information they can access.

This is especially useful for the Query node to know all endpoints to query, time windows and external labels for each node, thus reducing the overhead of querying all nodes in the cluster.

Given a sidecar we can have it join a gossip cluster by advertising itself to other peers within the network.

```
thanos sidecar \
    --prometheus.url            http://localhost:9090 \
    --tsdb.path                 /var/prometheus \
    --gcs.bucket                example-bucket \
    --grpc-address              0.0.0.0:19091 \            # gRPC endpoint for Store API (will be used to perform PromQL queries)
    --http-address              0.0.0.0:19191 \            # HTTP endpoint for collecting metrics on Thanos sidecar
    --cluster.address           0.0.0.0:19391 \            # Endpoint used to meta data about the current node
    --cluster.advertise-address 127.0.0.1:19391 \          # Location at which the node advertise itself at to other members of the cluster
    --cluster.peers             127.0.0.1:19391 \          # Static cluster peer where the node will get info about the cluster (repeatable)
```

With the above configuration a single node will advertise itself in the cluster and query for other members of the cluster (from itself) when you add more sidecars / components you will probably want to sent `cluster.peers` to a well known peer that will allow you to discover other peers within the cluster.

When a peer advertises itself / joins a gossip cluster it sends information about all the peers it currently knows about (including itself). This information for each peer allows you to see what type of component a peer is (Source, Store, Query), the peers Store API address (used for querying) and meta data about the external labels and time window the peer holds information about.

Once the Peer joins the cluster it will periodically update the information it sends out with new / updated information about other peers and the time windows for the metrics that it can access.

```
thanos query \
    --http-address              0.0.0.0:19192 \         # Endpoint for Query UI
    --grpc-address              0.0.0.0:19092 \         # gRPC endpoint for Store API
    --cluster.address           0.0.0.0:19591 \
    --cluster.advertise-address 127.0.0.1:19591 \
    --cluster.peers             127.0.0.1:19391 \       # Static cluster peer where the node will get info about the cluster
```

The Query component however does not have to utilize gossip to discover other nodes and instead can be setup to use a static list of well known addresses to query. These are repeatable so can add as many endpoint as needed. However, if you only use `store` you will automatically discover nodes added to the cluster.

```
thanos query \
    --http-address              0.0.0.0:19192 \         # Endpoint for Query UI
    --grpc-address              0.0.0.0:19092 \         # gRPC endpoint for Store API
    --store                     0.0.0.0:19091   \       # Static gRPC Store API Address for the query node to query
    --store                     0.0.0.0:19092   \       # Also repeatable
```

You can mix both static `store` and `cluster` based approaches:

```
thanos query \
    --http-address              0.0.0.0:19192 \         # Endpoint for Query UI
    --grpc-address              0.0.0.0:19092 \         # gRPC endpoint for Store API
    --cluster.address           0.0.0.0:19591 \
    --cluster.advertise-address 127.0.0.1:19591 \
    --cluster.peers             127.0.0.1:19391 \       # Static cluster peer where the node will get info about the cluster
    --cluster.peers             127.0.0.1:19392 \       # Another cluster peer (many can be added to discover nodes)
    --store                     0.0.0.0:18091   \       # Static gRPC Store API Address for the query node to query
    --store                     0.0.0.0:18092   \       # Also repeatable
```

When to use gossip vs store flags?
- Use gossip if you want to maintain single gossip cluster that is able to dynamically join and remove components.
- Use static store when you want to have full control of which components are connected. It is also easier to user static store options when setting up communication with remote (cross-cluster) components e.g (sidecar in different network through some proxy)

Configuration of initial peers is flexible and the argument can be repeated for Thanos to try different approaches.
Additional flags for cluster configuration exist but are typically not needed. Check the `--help` output for further information.

* _[Example Kubernetes manifest](../kube/manifests/prometheus.yaml)_
* _[Example Kubernetes manifest with GCS upload](../kube/manifests/prometheus-gcs.yaml)_

## Store Gateway

As the sidecar backs up data into the object storage of your choice, you can decrease Prometheus retention and store less locally. However we need a way to query all that historical data again.
The store gateway does just that by implementing the same gRPC data API as the sidecars but backing it with data it can find in your object storage bucket.
Just like sidecars and query nodes, the store gateway joins the gossip cluster and is automatically picked up by running query nodes as yet another data provider.

```
thanos store \
    --data-dir                  /var/thanos/store \     # Disk space for local caches
    --gcs.bucket                example-bucket \        # Bucket to fetch data from
    --cluster.address           0.0.0.0:19891 \
    --cluster.advertise-address 127.0.0.1:19891 \
    --cluster.peers             127.0.0.1:19391 \
```

The store gateway occupies small amounts of disk space for caching basic information about data in the object storage. This will rarely exceed more than a few gigabytes and is used to improve restart times. It is not useful but not required to preserve it across restarts.

* _[Example Kubernetes manifest](../kube/manifests/thanos-store.yaml)_

## Compactor

A local Prometheus installation periodically compacts older data to improve query efficieny. Since the sidecar backs up data as soon as possible, we need a way to apply the same process to data in the object storage.

The compactor component simple scans the object storage and processes compaction where required. At the same time it is responsible for creating downsampled copies of data to speed up queries.

```
thanos compact \
    --data-dir /var/thanos/compact \  # Temporary workspace for data processing
    --gcs-bucket example-bucket
```

The compactor is not in the critical path of querying or data backup. It can either be run as a periodic batch job or be left running to always compact data as soon as possible. It is recommended to provide 100-300GB of local disk space for data processing.

_NOTE: The compactor must be run as a **singleton** and must not run when manually modifying data in the bucket._

# All-in-one example

You can find one-box example with minikube [here](../kube/README.md).

# Dashboards

You can find example Grafana dashboards [here](../examples/grafana/monitoring.md)

# Alerts

You can find example Alert configuration [here](../examples/alerts/alerts.md)
