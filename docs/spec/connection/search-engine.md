# Search engine contract

This document defines version `1` of the `search-engine` contract. It follows the rules of [OKDP connection](../connection.md).

## 1. Purpose

Services index documents and search them, for example a data catalog. This contract defines what a `connection` to a search engine gives and what a service that uses it guarantees.

A search engine here is a server reached over HTTP with its REST API and a username and a password. The contract applies to any engine that matches this definition, for example OpenSearch or Elasticsearch.

A `connection` is published by the component that runs the engine, or written by hand for an existing server. Version `1` holds no requirement for a `claim` and has no `provisioner`: a service names the `connection` it uses.

## 2. Connection

| Property | Meaning |
|---|---|
| `engine` | Name of the engine, in lowercase, for example `opensearch` |
| `host`, `port` | Address of the server |
| `tls` | Optional. `true`: the client uses `https` and verifies the certificate and the name of the server |
| `secretRef` | Secret holding `username` and `password` |
| `caRef` | Optional. Secret holding `ca.crt`, the CA that signed the certificate of the server. Unset: the CA bundle of the client |
| `serverVersion` | Optional. Version of the server, as numbered by the engine, for example `3.10.0` |

## 3. Rules

- The Secrets of a `connection` are in the namespace of the service that uses it.
- The identity of a `connection` can create, read, change and remove indexes. Two services that use the same `connection` can reach each other's data.
- A service writes only indexes whose name starts with a prefix of its own, for example the name of its instance, so that several services can share a server.
- A service keeps its own data elsewhere when it cannot rebuild it: the content of a search engine can be lost.

## 4. Removal

- Removing a `connection` changes nothing on the server. The indexes and their data are kept.
- Removing a service does not remove its indexes.
