---
name: Authenticate and read a Fenergo legal entity
description: Obtain a Fen-X access token with least-privilege scopes and read a legal entity and its journeys from the Query APIs.
api: openapi/fenergo-entitydataquery-v2-0-openapi.json
operations: [GetEntityById, EntityAdvancedSearch, SearchByName, GetEntitiesPagedListV2, GetInstancesByEntityId, GetLifecycleStatusByEntityId]
generated: '2026-09-09'
method: generated
source: openapi/ + https://docs.fenergox.com/api-docs/tenant-access
---

# Authenticate and read a Fenergo legal entity

Fenergo's Fen-X platform is CQRS: you **read** from a `*query` service and **write** to a `*command`
service. This skill covers the read path and the token every other Fenergo skill depends on.

## 1. Get an access token

`POST https://identity.fenergox.com/connect/token` (or `https://identity.{region}.fenergox.com/connect/token`
for your tenant's region), `Content-Type: application/x-www-form-urlencoded`:

```
grant_type=client_credentials
client_id=<your client id>
client_secret=<your client secret>
scope=fenx.entitydata.read fenx.journey.read
```

- Request **only** the scopes you need. `.read` scopes map to Query APIs, `.write` scopes to Command APIs.
- The `scope` string is space-separated and capped at **300 characters**.
- The response is `{"access_token": "...", "expires_in": 900, "token_type": "Bearer", "scope": "..."}`.
- **Cache the token for its full 900-second TTL.** The identity service allows only 100 token requests per
  source IP per 5 minutes — a token request per API call will rate-limit you out.
- Do **not** send `x-tenant-id` on the token request.

## 2. Call the API

Every CLM call needs two headers:

```
Authorization: Bearer <access_token>
x-tenant-id: <tenant GUID>
```

A missing tenant header returns HTTP 400 with `errorCode: MissingTenantHeader` at the gateway.
Optionally send `x-correlation-id: <GUID>` — it is echoed onto the event notifications the call produces.

## 3. Find the entity

Base URL: `https://api.fenergox.com/entitydataquery`

- `SearchByName` — `POST /api/v2/entity/searchbyname` when you have a name.
- `EntityAdvancedSearch` — `POST /api/v2/entity/entityadvancedsearch` for structured criteria.
- `GetEntitiesPagedListV2` — `POST /api/v2/entity/getentitiespagedlist` for a paged list with a total count.

## 4. Read it

- `GetEntityById` — `GET /api/v2/entity/{id}`.
- PII fields come back masked. Reveal one deliberately with `GetEntityPIIPropertyValue`
  (`GET /api/v2/entity/{entityId}/property/{propertyName}`) — that is an audited action, so do it only when
  the task genuinely needs the value.

## 5. Read its journeys

Base URL: `https://api.fenergox.com/journeyquery` (scope `fenx.journey.read`)

- `GetInstancesByEntityId` — `GET /api/journey-instance/search`.
- `GetLifecycleStatusByEntityId` — `GET /api/journey-instance/lifecycle-status/entity/{entityId}`.
- `GetJourneyInstanceById` — `GET /api/journey-instance/{journeyInstanceId}`.

## Rules

- **Eventual consistency.** A write accepted by a Command API is not immediately readable on the Query API.
  Do not poll tightly; subscribe to the matching event (`entitydata:dataPublished`) or back off.
- **Rate limits.** 200 req/s per production tenant (burst 50); Entity Data has its own 200/minute,
  2,500/hour, 15,000/day domain limit. On 429 read `X-Rate-Limit-Remaining` and `X-Rate-Limit-Reset` and
  retry with exponential backoff plus jitter.
- **Errors** arrive in the ServiceResponse envelope: `{"data": …, "messages": [{"message","type","errorCode"}]}`.
  See `errors/fenergo-problem-types.yml`.
- **Client credentials bypass access layers.** A credential holding `fenx.entitydata.read` reads *all* entity
  data in the tenant, not a user's slice. Scope the credential, not the query.
