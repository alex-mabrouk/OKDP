# Log store contract

This document defines version `1` of the `logs` contract. It follows the rules of [OKDP connection](../connection.md).

## 1. Purpose

Services and the platform write logs to a log store, and some platform components read them back. This contract defines what the `connection` gives and what the store guarantees.

A log store here receives logs with OTLP over HTTP, the protocol of OpenTelemetry, and answers queries with its own query API. The store is everything behind the `endpoint`: a guarantee of this contract can be kept by the engine itself or by a component in front of it. Metrics and traces are out of scope.

There is one store per cluster, used by every namespace. A store serves no other cluster. Projects are kept apart by the namespace of each entry, not by identities. The contract therefore holds no requirement for a `claim` and has no `provisioner`: a service uses the `connection` directly. The `connection` is published by the package of the store, or written by hand for an existing store. 

**The connections live in a platform namespace. No project gets one.**

## 2. Connection

A `connection` gives one identity, for writing or for reading.

| Property | Meaning |
|---|---|
| `access` | `write`: the identity sends logs. `read`: the identity runs queries |
| `endpoint` | For `write`, base URL of the OTLP/HTTP receiver: logs are sent to `<endpoint>/v1/logs`. For `read`, base URL of the query API |
| `api` | Only for `read`. Name of the query API, in lowercase, for example `logql` or `logsql` |
| `secretRef` | Secret holding `headers`: the HTTP headers sent with every request, in the format of `OTEL_EXPORTER_OTLP_HEADERS`, for example `Authorization=Bearer <token>` |

- A `write` identity cannot read, and a `read` identity cannot write.
- A `read` identity reads every namespace.

## 3. Rules

### 3.1 Writing

- The store accepts OTLP/HTTP logs in binary Protobuf encoding.
- It keeps the body and the attributes of each entry, and a query can filter on any of them.

### 3.2 Namespace

- Every entry carries the resource attribute `k8s.namespace.name`.
- The collector sets it from where the entry comes from: the namespace of the pod, or, for an entry about a Kubernetes object, the namespace of that object. A workload never chooses the namespace of its entries: a value it writes is replaced.
- An entry that belongs to no namespace carries `_platform`. No namespace can have this name.

### 3.3 Reading

- A reader can limit a query to one namespace or to `_platform`. The store applies the limit itself, to the whole query, subqueries included. A query cannot remove or widen it.
- How a reader passes the limit is part of the `api`.
- No user gets a `read` identity. A user reads through a platform component, which checks the rights of that user before it sets the limit. The store holds no rights of its own.
