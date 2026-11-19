# Metrics store contract

This document defines version `1` of the `metrics` contract. It follows the rules of [OKDP connection](../connection.md).

## 1. Purpose

The platform sends the metrics of its cluster to metrics stores outside OKDP. This contract defines what a `connection` gives, how services declare their metrics, what every data point carries, and what a store and a collector guarantee.

A metrics store here receives metrics with OTLP over HTTP, the protocol of OpenTelemetry. The store is everything behind the `endpoint`: a guarantee of this contract can be kept by the engine itself or by a component in front of it. OKDP does not install, choose or read a store. Logs have their own [contract](./log-store.md). Traces are out of scope.

A collector is a platform component that reads metrics in the cluster and sends them to the stores. A cluster can have several collectors.

A `connection` is written by hand for an existing store. The contract holds no requirement for a `claim` and has no `provisioner`. Each `connection` is a destination. A cluster can have several, and a store can serve several clusters.

**The connections live in a platform namespace. No project gets one.**

## 2. Connection

| Property | Meaning |
|---|---|
| `endpoint` | Base URL of the OTLP/HTTP receiver, in `https`. Metrics are sent to `<endpoint>/v1/metrics` |
| `secretRef` | Secret holding `headers`: the HTTP headers sent with every request, in the format of `OTEL_EXPORTER_OTLP_HEADERS`, for example `Authorization=Bearer <token>` |
| `caRef` | Optional. Secret holding `ca.crt`, the CA that signed the certificate of the store. Unset: the CA bundle of the collector |

## 3. Rules

### 3.1 Store

- The store accepts OTLP/HTTP metrics in binary Protobuf encoding, compressed with gzip (`Content-Encoding: gzip`) or not.
- It keeps the resource attributes of section 3.3, so that a query can filter on them.
- The identity of a `connection` can only write. It cannot read, change or remove data.

### 3.2 Sources

A collector reads:

- the metrics of the nodes, pods and containers, and the state of the Kubernetes objects;
- its own metrics, including those of section 3.4;
- the targets that `ServiceMonitor` and `PodMonitor` objects declare (API `monitoring.coreos.com/v1`), in the Prometheus exposition format.

A service that exposes metrics declares them with a `ServiceMonitor` or a `PodMonitor` in its namespace. A monitor tells a platform component what to read, so the platform refuses, when it is written, a monitor that:

- selects pods or services of another namespace;
- gives a credential for the target: `basicAuth`, `authorization`, `oauth2`, `bearerTokenSecret`, or a TLS certificate or key taken from a Secret.

### 3.3 Attributes

Every data point carries these resource attributes:

| Attribute | Value |
|---|---|
| `k8s.cluster.uid` | UID of the `kube-system` namespace of the cluster |
| `k8s.namespace.name` | Namespace of the data point |

- The collector sets them from where it read the metric, never from the labels of the metric. A target never chooses them: a value it writes is replaced.
- The namespace of a data point is the namespace of the pod it was read from, or, for a metric about a Kubernetes object, the namespace of that object. A data point that belongs to no namespace, for example about a node, carries `_platform`. No namespace can have this name.
- A store that serves several clusters tells their data apart with `k8s.cluster.uid`: two clusters can have a namespace with the same name.

### 3.4 Delivery

- A collector sends each data point at least once to each destination. A store can receive the same data point twice.
- A collector keeps on disk, for each destination, the data the store has not accepted yet. A destination that fails never stops the others.
- A data point is lost when the queue of its destination is full or when the store refuses it for good. The collector counts the lost data points of each destination in a metric. A loss is never silent.

### 3.5 Reading

- No component of OKDP reads a store.
- Keeping projects and clusters apart when people read is the job of the store and of the tools of its owner. They do it with the attributes of section 3.3.
