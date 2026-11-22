# Database metadata contract

This document defines version `1` of the `database-metadata` contract. It follows the rules of [OKDP connection](../connection.md).

## 1. Purpose

A data catalog describes the databases of a server: their schemas, tables, columns and views, and the queries that run on them, from which it derives usage and lineage. This contract defines what a `claim` for that access holds, what the `connection` gives, and what a `provisioner` guarantees.

It applies to the engines of the [database contract](database.md) that have catalogs and query statistics, for example PostgreSQL.

## 2. Connection

| Property | Meaning |
|---|---|
| `engine` | Name of the database engine, in lowercase, for example `postgresql` |
| `host`, `port` | Address of the server |
| `dbName` | Database to connect to first. It lists the other databases and gives the query statistics of the server |
| `secretRef` | Secret holding `username`, `password` and `uri` |
| `serverVersion` | Optional. Version of the server, as numbered by the engine |
| `tls` | Optional. `required`: the client must use TLS. `verify`: the client must also verify the certificate and the name of the server |

`uri`, `tls` and `serverVersion` follow the [database contract](database.md).

## 3. Claim

| Requirement | Meaning | Met when |
|---|---|---|
| `engines` | Engines the `claimer` supports | `engine` is in the list |
| `serverVersion` | Constraint on the version of the server, for example `>=15` | `serverVersion` meets the constraint |

- Every requirement is optional. A constraint follows the syntax of the [database contract](database.md).
- A `claim` passes no other value to the `provisioner`.

## 4. Provisioner

- The `provisioner` binds only the `claimers` that the owner of the server names. It refuses the others, and the reason says so.
- Each `claim` gets its own user. The user connects to every database the `provisioner` created on the server, reads their structure and the query statistics of the server, with the text of the queries.
- The user reads no row of the databases, and writes nothing: it cannot create objects, databases or users, or bypass access control.
- The users of the [database contract](database.md) get no access to the query statistics.
- The Secret is written in the namespace of the `claimer`.
- The `connection` is defined once the user exists and reads the query statistics.
- A `claimer` that the owner of the server no longer names loses its access: its user can no longer log in, and its Secret and its `connection` are removed.

## 5. Removal

When a `claim` is removed, its user can no longer log in and its Secret is removed. The databases are not changed.
