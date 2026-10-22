# Design Proposal: identity clients, registered and removed with the instance that uses them

> Tracking issue: none yet
> Status: **DRAFT** (revision 3)

## 1. Problem

Most OKDP services sign users in with OIDC, so each instance needs its own OIDC client. Today the
packages register them with a Helm chart that runs a Job before the install. That works for a
single install, but:

- two instances installed at the same time in one namespace overwrite each other's Job: the
  install is reported as successful, the client Secret is missing, the pod never starts;
- uninstalling an instance, or deleting a project, never removes its clients from the identity
  provider;
- each package carries the registration URL and the authentication of the registration.

Goal: packages declare the clients they need; the platform registers them, keeps them, and removes
them with the instance, whatever the identity provider and the deployment engine.

## 2. Current state

| Area | What exists | Reference |
|---|---|---|
| Registration | `quay.io/adaltas/oidc-dcr` 0.3.2, one module per client in 8 packages (10 modules). Fixed object names (Job and ConfigMap `dcr`, ServiceAccount and Role `oidc-dcr`); never updates a client; never unregisters; writes the Secret outside Helm | `platform-packages`, `community-packages` |
| Workarounds | polaris and superset chain their two client modules with `dependsOn`; polaris sets a placeholder redirect URI on a service client | `polaris.yaml`, `superset.yaml` |
| Provider settings | `oidc.dcr.registrationUrl` and `oidc.dcr.authMethod` in the Context, read by every package | sandbox Contexts |
| Uninstall | the packaging contract lists `identity-client` as an external resource to clean up (P6.6), nothing implements it | packaging contract, descriptor 3.4 |
| Project deletion | the control plane deletes the namespace: no Helm hook can run in a terminating namespace | `okdp-control-plane-server`, `project_repository.go` |

## 3. Decisions

1. Registration is a **platform capability**, `okdp.io/identity-client-registration`, separate
   from the identity provider and from any package.
2. It uses **dynamic client registration only** (RFC 7591, RFC 7592): no admin API, so it works
   with any provider that supports it.
3. The implementation is a **generic Kubernetes operator** that knows nothing about OKDP. OKDP
   depends on its API, never on its internals.
4. The operator reads **no Secret outside its namespace** unless a namespace delegates it.
5. A deletion counts only when the **provider confirms it**. Anything else blocks, and only an
   explicit administrator action leaves a client behind.

## 4. The contract

### 4.1 Terms

- **Identity client**: an OIDC client registered with the identity provider for one service of one
  instance. External resource of kind `identity-client` (descriptor 3.4).
- **Provisioning mode**: `dcr` (the platform registers the clients when the instance is installed)
  or `existing` (an administrator registered them beforehand and the platform gives their Secrets).
- **Clients component**: the component of a package that declares its `OIDCClient` objects.

### 4.2 Capabilities and interfaces

| Capability | Level, scope, cardinality | Interface (contract, version) |
|---|---|---|
| `okdp.io/identity-provider` (exists) | platform, `system-wide`, `one-per-cluster` | `oidc-provider` 1 |
| `okdp.io/identity-client-registration` (new) | platform, `system-wide`, `one-per-cluster` | `identity-client-registration` 1 |

`oidc-provider` 1, published by the identity provider:

| Value | Meaning |
|---|---|
| `issuerUri` | issuer the services trust |
| `scope` | scopes requested at sign-in, `openid` and `groups` included |
| `groupsClaim` | always `groups` in version 1 (I6) |

`identity-client-registration` 1, published by the registration capability:

| Value | Meaning |
|---|---|
| `api` | `oidc.alex-mabrouk.github.io/v1alpha1`, kinds `OIDCClient` and `OIDCProvider` |
| `provider` | name of the `OIDCProvider` the clients register with |
| `grantee` | ServiceAccount (namespace, name) the credentials Secrets are delegated to (I8) |

The package providing the registration capability creates the `OIDCProvider` from the `issuerUri`
of `oidc-provider`, whatever product provides the identity: it depends on the capability
`okdp.io/identity-provider`, not on a package. It installs the operator, with its CRDs, in one
component, and the `OIDCProvider` in a second component that depends on the first: Helm maps every
object of a release before it creates the CRDs of that release, so a component cannot hold a CRD
and an object of its kind.

**Transition.** Until the engine resolves capability interfaces, the values come from the platform
configuration, as the ingress values do (descriptor 3.1):

| Platform key | Interface value | Default |
|---|---|---|
| `oidc.issuerUri` | `oidc-provider.issuerUri` | required |
| `oidc.scope` | `oidc-provider.scope` | `openid profile email groups` |
| `oidc.clientProvisioning` | (mode, I2) | `dcr` |
| `oidc.dcr.provider` | `identity-client-registration.provider` | `platform` |
| `oidc.dcr.grantee` | `identity-client-registration.grantee` | `oidc-system/oidc-client-operator` |

`oidc.dcr.registrationUrl` and `oidc.dcr.authMethod` are removed: how the registration
authenticates belongs to the `OIDCProvider`.

### 4.3 Rules

**I1 — Declaring the clients** (MUST, package). In mode `dcr`, the package declares one `OIDCClient`
per identity client, all in its clients component, in the namespace of the instance, with the grant
of I8. The descriptor lists `identity-client` in `sideEffects.external`, and declares an optional
dependency on `okdp.io/identity-client-registration`, required when the mode is `dcr`.

| Field | Value |
|---|---|
| name | `<instance>` for the main client, `<instance>-<role>` for any other (`console`, `trino`…) |
| `providerRef.name` | `identity-client-registration.provider` |
| `client.clientName` | omitted (the implementation names it after the namespace and the name) |
| `client.grantTypes`, `client.redirectUris`, `client.postLogoutRedirectUris` | what the service needs, nothing more; no placeholder redirect URI |
| `client.tokenEndpointAuthMethod` | `none` for a client running in a browser, the default otherwise |
| `client.scopes` | `oidc-provider.scope` without `openid`, plus what the service needs (`offline_access`) |
| `credentials.secretName` | `<instance>-oidc` for the main client, `<instance>-oidc-<role>` otherwise |
| `credentials.keys` | omitted: keys `client_id` and `client_secret` (I5) |

Two instances never share a client; two clients of one instance never share a Secret.
*Test: render a package in mode `dcr` → one OIDCClient per client, names and Secret names as above;
two instances in one namespace → disjoint names; descriptor without `identity-client` → CI fails.*

**I2 — The provider comes from the platform** (MUST, package). The package takes the values of
4.2 from the interfaces (or, during the transition, from the platform keys), never from its own
parameters. *Test: package that reads a registration URL or sets its own provider → CI fails;
platform without the registration capability and a package in mode `dcr` → the instance is not
deployed (P5.2), the missing dependency is named.*

**I3 — Waiting for the clients** (MUST, package, engine and okdp-contract). The component that runs
the service starts only once every `OIDCClient` of the instance is ready. Readiness is the kstatus
status of the object: `Current` is ready, `InProgress` is waiting, `Failed` stops the apply. Its
failure, or the end of `lifecycle.applyTimeout` (P5.7), makes the instance `intervention-required`
with the reason of the `Stalled` condition. The service reads its Secret through a required
reference, so it never starts with missing credentials, whatever the engine.
okdp-contract checks the order on the rendered package: every component whose workloads reference
a Secret named by an `OIDCClient` depends, directly or not, on the component declaring that
`OIDCClient`; the clients component depends on no component that consumes its Secrets.
*Test: provider down at install → the service is not started, the instance shows
`ProviderNotReady`; provider back → the install completes without intervention (P6.8); four
instances installed at the same time in one namespace → every Secret exists and every pod is ready;
okdp-contract on a package whose service does not depend on its clients component, or on a package
with a cycle → fails, naming both components.*

**I4 — Removal** (MUST, engine, package and control plane). Identity clients are owned by the
instance. "Gone" below means: the `OIDCClient` no longer exists, the provider confirmed the deletion
of the client, and the credentials Secret no longer exists.

| Event | Required behaviour |
|---|---|
| Uninstall of the instance | every client of the instance is gone before the uninstall completes |
| Deletion of the project | the deletion of the namespace waits until every client is gone; the control plane shows the project as being deleted, and what blocks it |
| Upgrade that removes a client from the clients component | that client is gone once the upgrade completes; the others are untouched (same `client_id`) |
| Upgrade that removes the clients component | every client of that component is gone once the upgrade completes; nothing of the removed component remains |
| Deletion not confirmed | the instance (or the project) is blocked, the reason of the `Stalled` condition is shown; a platform administrator may force it (P6.6), after a confirmation that lists the `client_id` left in the provider; the action is recorded |

Forcing is the `abandon-client` action of the implementation; the control plane never sets it
without the confirmation above.

**Known gap (KuboCD).** A module removed from a package leaves its HelmRelease in the cluster:
nothing of it is deleted, so its `OIDCClient` objects, and their clients, stay. Until the engine
removes the components a new version drops (P6.6 applied to upgrades):
- **I4a (package)** — a package never removes its clients component in an upgrade. To drop every
  client, the component stays and declares none.
- **I4b (okdp-contract)** — okdp-contract compares the new version with the previous one: a
  component removed while it declared `OIDCClient` objects in the previous version fails, naming
  the component, and points to I4a.
- **I4c (okdp-contract, failure of the comparison)** — the check fails closed. The previous
  version is the highest published version lower than the new one. If it cannot be read (registry
  unreachable, artifact missing or unreadable, descriptor invalid), the check fails and says why;
  an unreadable previous version is never taken as "no previous version". Only the first published
  version of a package has nothing to compare, which the check reports. There is no switch to skip
  the comparison.

*Tests:*
- *uninstall → no client left in the provider, no Secret left;*
- *project deletion → same;*
- *provider down during the deletion → blocked, then completed once it is back;*
- *forced deletion → the orphans are listed and recorded;*
- *upgrade from a version with two clients to a version with one → one client gone, the other keeps
  its `client_id`, the service of the kept client keeps working;*
- *upgrade to a version whose clients component declares no client → every client gone;*
- *engine conformance: upgrade to a version without one of the components → no resource of that
  component remains (HelmRelease included, for KuboCD); until KuboCD passes it, okdp-contract
  rejects such a version of a package with clients (I4b);*
- *okdp-contract with the previous version unreachable, missing or invalid → fails, naming the
  version and the cause; first version → passes and says it compared nothing.*

**I5 — Credentials** (MUST, package). The Secret holds `client_id`, and `client_secret` for a
confidential client. The service reads those two keys, by reference, never by copy into its
configuration. In mode `existing`, the platform gives Secrets with the same keys, so the only
difference between the modes is the name of the Secret. The issuer comes from
`oidc-provider.issuerUri`, not from the Secret. A change of client (explicit re-registration)
updates the Secret in place: a package whose service reads the credentials only at start declares
how it restarts, a checksum of the Secret for example. *Test: same rendering in both modes apart
from the Secret names; Secret updated → the service uses the new client after the declared
restart.*

**I6 — Groups and access** (MUST, package and identity provider). Authorization uses the claim
`groups`: membership of the user in groups, short names, emitted when the scope `groups` is
requested. The package providing `okdp.io/identity-provider` provides the scope `groups` and allows
it for registered clients. A package never relies on realm roles or on provider-specific claims,
which registered clients may not receive. A service that maps users to roles exposes the parameter
`access`: application role → list of groups. *Test: client registered in mode `dcr` → its tokens
carry `groups`; a user in a group of `access` gets the role, a user in none is refused.*

**I7 — No secret in the declaration** (MUST, package). The `OIDCClient` carries no credential.
Credentials only exist in the Secret written by the implementation (P4.5). *Test: CI rejects an
`OIDCClient` with a field holding a secret.*

**I8 — Delegated credentials** (MUST, package and registration capability). The registration
capability reads no Secret outside its own namespace unless a namespace delegates it. The clients
component grants `identity-client-registration.grantee`, in the namespace of the instance, `get`
and `update` on the names of its credentials Secrets, and `create` on Secrets; nothing else. The
registration capability gets no right on the Secrets of the cluster.

When a delegation disappears while its clients exist:
- nothing is registered, re-registered or unregistered because of it;
- the credentials Secret is no longer refreshed: the service keeps working with the credentials in
  place, but a Secret deleted or altered by hand is not restored;
- the `OIDCClient` is `Stalled` with `SecretAccessDenied`, so the instance is `not-converged`
  (P5.5), and the control plane names the missing grant;
- the registration is kept, and the uninstall still removes the client: unregistering needs no
  access to the namespace;
- giving the delegation back restores the Secret without intervention.

*Tests: okdp-contract on the rendered package → the grant names exactly the
`credentials.secretName` of its `OIDCClient` objects; the registration capability tries to read a
Secret that is not delegated → refused; a client without grant → `Stalled`, `SecretAccessDenied`,
nothing registered; delegation removed while a client exists, Secret altered → not restored,
`SecretAccessDenied`, the client keeps its `client_id`; instance then uninstalled → the client is
gone; delegation given back instead → the Secret is restored.*

**I9 — Lifecycle of the registration capability** (MUST, registration capability, engine and
control plane). The capability cannot be uninstalled while `OIDCClient` objects exist: the control
plane refuses it (P5.8, listing the instances), and the uninstall itself fails, also when the
release is uninstalled directly with the engine or Helm. The engine keeps retrying, and completes
the uninstall once the last `OIDCClient` is gone, without intervention. While the uninstall is
refused, components that depend on the capability report it as not ready. Bypassing that
protection is an explicit action of a platform administrator. An upgrade keeps the registrations
and resumes the pending deletions. When the capability is stopped, deletions wait and nothing is
lost; the recovery procedure is published with the implementation.
**Known gap (KuboCD).** Deleting a KuboCD Release removes the Release object at once, without
waiting for its HelmReleases. With an `OIDCClient` left, the HelmRelease of the operator stays,
refused and retried, and the operator keeps running, but the Release, which is what the control
plane shows, is gone: the capability looks uninstalled while it still runs. Until the engine keeps
the Release until its components are gone, the control plane refuses the deletion itself (P5.8), and
the package of the capability is installed with `protected: true`, which KuboCD refuses to delete.

*Test: uninstall of the capability, through the control plane and directly through Helm, with one
`OIDCClient` left → refused, the capability keeps running, the `OIDCClient` is listed; last
`OIDCClient` deleted → the uninstall completes without intervention; capability stopped during a
project deletion, then restarted → the deletion completes without intervention.*

## 5. Reference implementation: the OIDC client operator

`github.com/alex-mabrouk/oidc-client-operator` (Apache 2.0, private while it is being validated).

| Contract | Implementation |
|---|---|
| `OIDCClient`, `OIDCProvider` | CRDs `oidc.alex-mabrouk.github.io/v1alpha1`, installed and upgraded with the chart, kept on uninstall |
| Readiness (I3) | conditions `Ready`, `Reconciling`, `Stalled`; a new object reads `observedGeneration: -1` |
| Removal (I4) | finalizer on each `OIDCClient`; released only on a 2xx answer to the RFC 7592 deletion, or on `abandon-client: <client_id>` |
| Registrations | registration access token and a copy of the credentials in the operator namespace, held by a finalizer |
| Delegation (I8) | chart value `rbac.credentials: delegated`; the `oidc-client` chart renders the grant |
| Lifecycle (I9) | `pre-delete` hook `--uninstall-check` |

RFC 7592 answers 401 both for a deleted client and for an invalid registration token: a 401 never
counts as a deletion, and never triggers a new registration.

## 6. Migration

For the 10 modules `oidc-dcr` of today:

- each package gets one clients component with its grant, under the module name `oidc-dcr`, so that
  under I4a the HelmRelease is upgraded, not left behind;
- polaris and superset drop their second module (`oidc-dcr-console`, `oidc-dcr-trino`). Its
  HelmRelease is left behind by KuboCD. It holds no resource, only hooks, so I4b does not apply,
  but the migration notes remove it;
- the clients registered by `oidc-dcr` stay in the provider, and their Secrets stay too, whatever
  happens to the HelmRelease: the migration notes unregister them with their registration token
  (RFC 7592), delete their Secrets, then delete the hook objects with fixed names (`job/dcr`,
  `configmap/dcr`, `service/dcr-headless`, `serviceaccount/oidc-dcr`, `role/oidc-dcr`,
  `rolebinding/dcr`) once no `oidc-dcr` module is left in the namespace;
- Secret names change (`<instance>-<ns>-dcr[-role]` → `<instance>-oidc[-role]`); trino reads
  `client_id` / `client_secret` instead of `OIDC_CLIENT_ID` / `OIDC_CLIENT_SECRET`;
- `oidc.dcr.registrationUrl` and `oidc.dcr.authMethod` leave the Contexts; `oidc.dcr.provider` and
  `oidc.dcr.grantee` come with their defaults;
- new package: the operator, providing `okdp.io/identity-client-registration`, in two components
  (4.2);
- open-webui and rustfs keep `existing` by default (to be decided separately);
- okdp-contract: checks of I3, I4b, I4c and I8.

## 7. Out of scope

- How the implementation authenticates to the provider (anonymous registration, initial access
  token): configuration of the `OIDCProvider`.
- Provider-side configuration of the clients (consent, role mappings): not managed.
- The control plane and its console clients: registered once by the platform, mode `existing`.
- Rotation of client secrets.

## 8. Verification

Done on 2026-10-22, against Keycloak 26.1.3 and Flux 2 (helm-controller 1.6.5):

| Check | Result |
|---|---|
| Keycloak answers (GET keeps the token, PUT rotates it, 401 for a deleted client as for a bad token, no 404) | measured, the rules of 4.3 rely on them |
| Operator scenarios (creation, 20 at once, update, deletion confirmed or not, provider down, abandon, re-registration, lost or uncertain registration, conflict, delegation missing, revoked and given back) | envtest, operator running with the RBAC of its chart |
| Install through Flux, two clients in a namespace | both registered, Secrets written, grant naming the two Secrets |
| Upgrade removing one client (I4) | removed client gone from Keycloak with its Secret; the other keeps its `client_id` |
| Delegation revoked (I8) | `SecretAccessDenied`, Secret not restored, registration kept; uninstall then removed the client |
| Uninstall of the operator with a client left (I9) | uninstall refused, HelmRelease and operator kept, Flux retried; completed on its own after the client was deleted |
| Same, installed as a KuboCD Release | protected Release: deletion denied by KuboCD; unprotected: the Release disappears at once, the operator HelmRelease stays refused until the last client is gone, then completes; a new Release takes back the kept CRDs and `OIDCProvider` |
| Project deletion (namespace deleted directly) | the client is removed from Keycloak, nothing is left |
| Left-behind `oidc-dcr` HelmRelease (migration) | deleting it leaves the client, its Secret and the hook objects; the migration procedure removed them all |

Still to verify with the packages: I3 with the engine (service waiting for its clients), I6 with
the services, the project deletion through the control plane, and the KuboCD conformance test of I4.

## 9. Open questions

1. Who records a forced deletion (I4): the control plane, or the engine?
2. Where does the `grantee` of the transition come from once the capability is a package: a
   published connection, or a platform key set by the package of the capability?
