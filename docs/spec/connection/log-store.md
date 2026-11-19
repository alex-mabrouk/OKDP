# Log store contract

This document defines version `1` of the `logs` contract. It follows the rules of [OKDP connection](../connection.md).

## 1. Purpose

The platform sends the logs of its cluster to log stores outside OKDP: a central log store, a SIEM, an archive. This contract defines what a `connection` gives, what every entry carries, and what a store and a collector guarantee.

A log store here receives logs with OTLP over HTTP, the protocol of OpenTelemetry. The store is everything behind the `endpoint`: a guarantee of this contract can be kept by the engine itself or by a component in front of it. OKDP does not install, choose or read a store. Metrics have their own [contract](./metrics-store.md). Traces are out of scope.

A collector is a platform component that reads logs in the cluster and sends them to the stores. A cluster can have several collectors.

A `connection` is written by hand for an existing store. The contract holds no requirement for a `claim` and has no `provisioner`. Each `connection` is a destination. A cluster can have several, and a store can serve several clusters.

**The connections live in a platform namespace. No project gets one.**

## 2. Connection

| Property | Meaning |
|---|---|
| `endpoint` | Base URL of the OTLP/HTTP receiver, in `https`. Logs are sent to `<endpoint>/v1/logs` |
| `secretRef` | Secret holding `headers`: the HTTP headers sent with every request, in the format of `OTEL_EXPORTER_OTLP_HEADERS`, for example `Authorization=Bearer <token>` |
| `caRef` | Optional. Secret holding `ca.crt`, the CA that signed the certificate of the store. Unset: the CA bundle of the collector |
| `types` | Optional. Values of `okdp.log.type` (section 3.2) the destination receives, for example `[audit]` for a SIEM. Unset: all of them |

## 3. Rules

### 3.1 Store

- The store accepts OTLP/HTTP logs in binary Protobuf encoding, compressed with gzip (`Content-Encoding: gzip`) or not.
- It keeps the body and the attributes of each entry.
- The identity of a `connection` can only write. It cannot read, change or remove entries.

### 3.2 Attributes

Every entry carries these resource attributes:

| Attribute | Value |
|---|---|
| `k8s.cluster.uid` | UID of the `kube-system` namespace of the cluster |
| `k8s.namespace.name` | Namespace of the entry |
| `okdp.log.type` | `audit`, `application` or `infrastructure` |

- The collector sets them. A workload never chooses them: a value it writes is replaced.
- The namespace of an entry is the namespace of the pod it comes from. An entry about an object, a Kubernetes object or data, carries the namespace that owns the object, for example the namespace of a catalog for an access decision on one of its tables. An entry that belongs to no namespace carries `_platform`. No namespace can have this name.
- `okdp.log.type` is:
  - `audit` for an audit event of the platform: a request to the Kubernetes API, or an access decision of a data engine;
  - otherwise `application` for a namespace of a project, that is a namespace with the label `okdp.io/project`;
  - otherwise `infrastructure`, `_platform` included.
- A store that serves several clusters tells their entries apart with `k8s.cluster.uid`: two clusters can have a namespace with the same name.

### 3.3 Delivery

- A collector sends each entry at least once to each destination whose `types` hold the `okdp.log.type` of the entry. A store can receive the same entry twice.
- A collector keeps on disk, for each destination, the entries the store has not accepted yet. A destination that fails never stops the others.
- An entry is lost when the queue of its destination is full or when the store refuses it for good. The collector counts the lost entries of each destination in a metric. A loss is never silent.

### 3.4 Reading

- No component of OKDP reads a store.
- Keeping projects and clusters apart when people read is the job of the store and of the tools of its owner. They do it with the attributes of section 3.2.
