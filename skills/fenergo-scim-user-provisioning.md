---
name: Provision Fen-X users with SCIM 2.0
description: Create, search, patch and deprovision users and groups on the Fenergo SCIM 2.0 identity service, and assign teams afterwards.
api: openapi/fenergo-identity-scim-v1-openapi.json
operations: [CreateUser, GetAllUsers, SearchUsers, GetUserById, UpdateUser, PatchUser, DeleteUser, CreateGroup, GetAllGroups, SearchGroups, PatchGroup, Bulk, GetServiceProviderConfig, GetAllSchemas]
generated: '2026-09-09'
method: generated
source: openapi/ + https://docs.fenergox.com/developer-hub/api-and-system-security/scim-overview
---

# Provision Fen-X users with SCIM 2.0

Fenergo runs a genuine **SCIM 2.0** service — the contract declares
`urn:ietf:params:scim:schemas:core:2.0:User`, `:Group`, `:ResourceType`, `:ServiceProviderConfig` and the
`urn:ietf:params:scim:api:messages:2.0:*` message URNs. If your IdP speaks SCIM (Okta and Microsoft Entra ID
are both tested by Fenergo), use a connector rather than writing this by hand.

Base URL: `https://identity.fenergox.com/scim`. Headers: `Authorization: Bearer …`.

## Scopes

SCIM uses its own scope family, **not** the `fenx.*` ones:

| Scope | Needed for |
|---|---|
| `scimapi.resource.query` | GET |
| `scimapi.resource.add` | POST |
| `scimapi.resource.update` | PUT, PATCH |
| `scimapi.resource.delete` | DELETE |
| `scimapi.resource.bulk` | POST /Bulk |

`scimapi.resource.bulk` does **not** imply add/update/delete — a bulk payload mixing creations and deletions
needs those scopes in the same token. Configure one tenant per SCIM client credential; the service resolves
the tenant from the token's tenant claim and cannot disambiguate two.

Request the SCIM client credential through a SaaS Request; it is not the same credential as your CLM one.

## Discover the server's own rules first

- `GetServiceProviderConfig` — `GET /ServiceProviderConfig` reports what this deployment supports.
- `GetAllSchemas` — `GET /Schemas`, `GetAllResourceTypes` — `GET /ResourceTypes`.

## Users

- `CreateUser` — `POST /Users`. **A primary email address is mandatory on every Fen-X SCIM operation.**
- `GetAllUsers` — `GET /Users` with `filter`, `sortBy`, `sortOrder`.
- `SearchUsers` — `POST /Users/.search` when the filter is too long or complex for a query string.
- `GetUserById` — `GET /Users/{id}`.
- `UpdateUser` — `PUT /Users/{id}` replaces the whole resource. `PatchUser` — `PATCH /Users/{id}` applies a
  `urn:ietf:params:scim:api:messages:2.0:PatchOp`. **Prefer PATCH**: a PUT built from a partial object will
  silently clear attributes you did not send.
- `DeleteUser` — `DELETE /Users/{id}`. There is no restore operation and no stated retention window.
  Deprovision by disabling (`active: false` via PATCH) unless erasure is actually what you were asked for.

## Groups

- `CreateGroup` — `POST /Groups`, `GetAllGroups` — `GET /Groups`, `SearchGroups` — `POST /Groups/.search`,
  `GetGroupById`, `UpdateGroup`, `PatchGroup`, `DeleteGroup`.

Published limits: **500 member PATCH updates per request**, **9,000 users per group**, **5 teams and 10
access layers** per group. Chunk membership changes to 500 and expect delays beyond these ceilings.

## Bulk

`Bulk` — `POST /Bulk` takes a `urn:ietf:params:scim:api:messages:2.0:BulkRequest` and returns a
`BulkResponse`. Read the per-operation status in the response: a 200 on the bulk envelope does not mean every
operation inside it succeeded.

## Authentication is not authorization

Creating the user does **not** grant it anything. Assign teams afterwards on the Authorization Command API:

`PUT https://api.fenergox.com/authorizationcommand/api/team/{id}/user/{userId}` (scope
`fenx.authorization.write`, with `x-tenant-id`).

Two steps, two services, two scope families. A user created and never assigned to a team can sign in and see
nothing.
