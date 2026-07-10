# API reference

> **Status: DRAFT** – proposal for the `specifications` repo, pending developer approval. Reflects code on `creator-credentials-backend@main` / `creator-credentials-ui@develop` as of 2026-07-10.

The Creator Credentials backend is a NestJS service. This document maps its real HTTP surface: every controller route, its role gate, and the terse request → response shape. Companion docs: [`04-connections-and-issuance.md`](04-connections-and-issuance.md) (connection + VC-issuance flows), [`07-verification-flows.md`](07-verification-flows.md) (email/domain/did:web/keypair/cert challenges), [`09-data-model.md`](09-data-model.md) (entities + enums).

## Base URL and versioning

- The service uses NestJS URI versioning with `defaultVersion: '1'` (`src/main.ts`), so **every route is served under `/v1/...`** except controllers declared `VERSION_NEUTRAL` (see [Health](#health) and [App / `.well-known`](#app--well-known)).
- Default listen port is `APP_PORT || 3100`, plain HTTP (`src/main.ts:32`). NB: `README` / `.env.example` claim HTTPS on `3200` – stale vs `main.ts` (see `backend-system-map.md` §Bootstrap).
- CORS is a hardcoded allow-list (`src/main.ts:13-24`): `cc-backend.liccium.network`, `liccium.app`, `localhost:3105`, and `(www.)creatorcredentials.dev` / `.app`.

## Authentication

Two layers of auth stack on top of each other; there is **no role guard**.

**Layer 1 – global Clerk middleware gate.** `ClerkExpressWithAuth()` runs on every request and populates `req.auth`. An inline gate (`src/app.module.ts:62-69`) throws `UnauthorizedException` whenever `req.auth.userId` is absent. This applies to all routes *except* the excluded/public paths below.

**Layer 2 – per-route `AuthGuard`.** Handlers annotate `@UseGuards(AuthGuard)` (`src/users/guards/clerk-user.guard.ts`). The guard loads the DB `User` by `req.auth.userId` (the Clerk id) and attaches it as `req.user`; it returns `false` (→ 403) if no DB row exists yet. `@GetUser()` injects that `User`; `@GetClerkUserAuth()` exposes raw `req.auth`.

**Role enforcement is manual, not a guard.** The only roles are `ClerkRole { Issuer='issuer', Creator='creator' }` (`src/users/user.entity.ts:19-22`). Each handler re-checks `user.clerkRole` itself and throws `NotFoundException('This api is only for ...')` on a mismatch (e.g. `src/users/users.controller.ts:91`). So calling an issuer-only route as a creator returns **404**, not 403. The "role" column in the tables below is what the handler enforces this way.

**Client token wiring (UI).** Every UI request function sends the Clerk session token as `Authorization: Bearer <token>` (`src/shared/utils/tokenHeader.ts`). The legacy next-auth Bearer interceptor is commented out, so unauthenticated request functions send no token at all – see [Discrepancies](#discrepancies-to-verify).

### Excluded / public paths

Opened up by `.exclude(...)` on the global gate (`src/app.module.ts:71-77`):

| Path | Notes |
|---|---|
| `.well-known/*` | Let's Encrypt acme-challenge (see [App](#app--well-known)). |
| `health` | DB liveness ping. |
| `v1/mocks` (and `v1/mocks/*`) | Static fixtures; most routes commented out. |
| `v1/credentials/export` | Cross-app import, JWT-authenticated on its own (see [Cross-app import](#cross-app-import)). |
| `v1/webhooks/*` | Clerk lifecycle events, svix-signature-verified. |

---

## Users

Identity + all self-service verification. Controller `UsersController` (`src/users/users.controller.ts`). Every route is `@UseGuards(AuthGuard)` except `GET users/check`, which reads `req.auth` directly.

| Method + path (`/v1`) | Role | Purpose | Request → response |
|---|---|---|---|
| `GET users/check` | any (no guard) | "Get me" for the Clerk session; lazily back-fills cert/did:key/email VC if missing (`:37`). | – → `User` (404 if webhook has not provisioned the row) |
| `GET users` | any | Return `req.user` (`:59`). | – → `User` |
| `GET users/profile` | any | Issuer profile shape (`:65`). | – → `{ companyName, description, domain, email:'' }` |
| `PATCH users/profile` | any | Update profile (`:76`). | `{ description?, domain? }` → updated `User` |
| `POST users/organization-name` | any | Set write-once org name (`:281`). | `{ organizationName }` → `User` |
| `POST users/did-liccium/connect` | any | Bind Liccium app did:key → Connect VC (`:216`). | `{ licciumDidKey }` → `User` |
| `POST users/did-liccium/disconnect` | any | Unbind Liccium did:key (`:229`). | – → `User` |
| `POST users/verification/domain/txt-record` | any | Generate DNS TXT record to publish (`:235`). | `{ domain }` → `{ txtRecord }` |
| `POST users/verification/domain/confirm` | any | Write PENDING Domain VC + arm poller (`:245`). | `{ domain }` (unverified) → `User` |
| `POST users/domain/disconnect` | any | Remove domain verification (`:252`). | – → `User` |
| `POST users/verification/did-web/well-known` | any | Generate the `did.json` to host (`:258`). | `{ didWeb }` → did.json string |
| `POST users/verification/did-web/confirm` | any | Write PENDING DidWeb VC + arm poller (`:268`). | `{ didWeb }` (unverified) → `User` |
| `POST users/did-web/disconnect` | any | Remove did:web (`:275`). | – → `User` |
| `GET users/creators?status=` | Issuer | Connected creators filtered by `CreatorVerificationStatus` (`:87`). | – → `{ creators }` |
| `GET users/creators/:creatorId` | Issuer | One creator + their credentials (`:106`). | – → `{ creator, credentials }` (unverified) |
| `POST users/creators/:creatorId/accept` | Issuer | Accept a creator connection → `ACCEPTED` (`:123`). | – → `Connection` |
| `POST users/creators/:creatorId/reject` | Issuer | Reject connection → `REJECTED` (`:135`). | – → `Connection` |
| `POST users/creators/:creatorId/revoke` | Issuer | Revoke connection → `REVOKED` (`:147`). | – → `Connection` |
| `GET users/issuers` | Creator | Active issuers (≥1 `credentialsToIssue`) (`:159`). | – → `Issuer[]` |
| `GET users/issuers/:issuerId` | Creator | One issuer + VC offer list (`:168`). | – → issuer + verified credentials |
| `POST users/issuers/:issuerId/confirm-request` | Creator | Request a connection → `REQUESTED` (`:182`). | – → `Connection` |

The DNS-TXT and did:web confirmations are asynchronous: two `@Cron(EVERY_10_SECONDS)` pollers (`checkUserDomains`, `checkUsersDidWeb`) re-check pending users and mint the VC on match. See [`07-verification-flows.md`](07-verification-flows.md).

---

## Credentials

The VC issuance engine. Controller `CredentialsController` (`src/credentials/credentials.controller.ts`).

| Method + path (`/v1`) | Role | Purpose | Request → response |
|---|---|---|---|
| `POST credentials/create/email` | any (guard) | Self-issue an email VC (normally automatic) (`:51`). | – → email VC (unverified) |
| `GET credentials/creator` | Creator | Creator's full VC wallet: email/domain/membership/dataSupplier/connect/keypair (`:116`). | – → grouped credential bundle |
| `GET credentials/issuer` | Issuer | Issuer's own email/domain/didWeb/membership VCs (`:80`). | – → grouped bundle |
| `GET credentials/issuers?status=` | Issuer | Issued member creds + requesting creators, filtered by status (`:63`). | – → issuance requests |
| `POST credentials/request` | Creator | Request a Member / DataSupplier / LicciumDataSupplier VC (`:186`). | `{ issuerId, credentialType }` → PENDING credential |
| `POST credentials/:credentialId/accept` | Issuer | Begin cert-signed acceptance; returns the signing challenge (`:468`). | – → `{ challenge, commands[], supportingCredential? }` |
| `POST credentials/:credentialId/accept/verify-signature` | Issuer | Finish acceptance with the issuer cert signature (`:494`). | `{ signature }` → SUCCESS credential |
| `POST credentials/:credentialId/reject` | Issuer | Delete a pending credential (`:521`). | – → `void` |
| `DELETE credentials/:credentialId` | Issuer | Delete a credential (`:534`). | – → `void` |
| `POST credentials/export` | public (JWT) | Cross-app import (see below) (`:279`). | `{ token }` → `{ userId, userEmail, credentials }` |

The accept → verify-signature pair is a challenge/response state machine: `accept` computes a JWS `signingInput` (`x5c` header) and stashes it as `__acceptanceChallenge` on the pending row; `verify-signature` checks the issuer's external-cert signature (`crypto.createVerify('SHA256')`) and, on success, sets `SUCCESS`, writes `credentialObject` + `token`, and attaches `proof.jwt`. Full trace in [`04-connections-and-issuance.md`](04-connections-and-issuance.md).

### Cross-app import

`POST /v1/credentials/export` is the **public** Liccium-app → Creator-Credentials import path. It is excluded from the global Clerk gate and authenticates on its own:

1. The RS256 JWT in `{ token }` is verified against a public key rebuilt from `LICCIUM_CLERK_KEYS_{KID,N,E}` (`src/credentials/credentials.controller.ts:282-307`). Missing/invalid token or missing `email`/`sub` → `UnauthorizedException`.
2. `email` + `licciumDidKey` are extracted from the token; the user is resolved via the Clerk Admin API (`GET api.clerk.dev/v1/users?email_address=`, Bearer `CLERK_SECRET_KEY`).
3. A **Connect VC** is (re)issued binding the CC did:key ↔ `licciumDidKey`; the response returns the creator's SUCCESS email/domain/membership/connect VCs: `{ userId, userEmail, credentials }`.

**(known issue)** In the response assembly (`src/credentials/credentials.controller.ts:435-461`) the `domain` and `connect` (and out-of-scope `wallet`) branches each gate on `emailCredential[0].credentialStatus` instead of their own credential's status (copy-paste bug). If the email VC is absent this also throws on `emailCredential[0]`; if the email VC exists but a domain/connect VC is `PENDING`/`FAILED`, it is still emitted as though verified. Only the `membership` branch filters on each credential's own status correctly.

---

## Cert-challenge

Issuer imports an external eIDAS QSeal/QSig X.509 cert and proves private-key possession. Controller `CertChallengeController` (`src/cert-challenge/cert-challenge.controller.ts`). All routes `@UseGuards(AuthGuard)`; effectively issuer-facing.

| Method + path (`/v1`) | Purpose | Request → response |
|---|---|---|
| `GET cert-challenge/status` | Latest challenge; self-heals `externalCertPem` from a verified row (`:20`). | – → challenge state, external cert PEM, active source, commands |
| `POST cert-challenge/initiate` | Delete non-terminal rows; create `initiated` (`:26`). | – → challenge row |
| `POST cert-challenge/submit-cert` | Validate cert vs eIDAS trust store; issue random challenge (`:32`). | `{ certPem }` → `{ challenge, commands[] }` |
| `POST cert-challenge/verify-signature` | Verify signature; persist `user.externalCertPem` (`:38`). | `{ signature }` → verified challenge |
| `POST cert-challenge/reset` | Wipe non-terminal rows (`:47`). | – → `void` |
| `DELETE cert-challenge/external-cert` | Remove cert; reset signing source to `platform` (`:53`). | – → `User` |
| `PATCH cert-challenge/active-source` | Toggle `platform` / `external` signing source (`:59`). | `{ source }` → `User` |

Notes (dev/test escape hatches, see `backend-system-map.md` §cert-challenge): if the eIDAS trust store is not ready, chain validation is **skipped and passes** (`validation/cert-validator.service.ts:78-83`); the 60-min challenge TTL check is **commented out** (`cert-challenge.service.ts:110-117`).

---

## Keypair-challenge

Creator proves ownership of an external EC P-256 keypair (for Data Supplier VCs). Controller `KeypairChallengeController` (`src/keypair-challenge/keypair-challenge.controller.ts`). All routes `@UseGuards(AuthGuard)`.

| Method + path (`/v1`) | Purpose | Request → response |
|---|---|---|
| `GET keypair-challenge/status` | Latest challenge; surfaces verified-unconsumed did:key (`:24`). | – → challenge state, external did:key, active source, commands |
| `GET keypair-challenge/did-key-pem?did=` | Reconstruct PEM from a `did:key:z...` (rejects legacy hash DIDs) (`:30`). | – → `{ publicKeyPem }` (unverified) |
| `POST keypair-challenge/initiate` | Wipe prior in-progress + verified rows (single-use); create `initiated` (`:49`). | `{ keyFilePrefix? }` → `{ challenge, commands[] }` |
| `POST keypair-challenge/submit-public-key` | Validate EC P-256 PEM; issue challenge; derive did:key (`:58`). | `{ publicKeyPem }` → challenge + did:key |
| `POST keypair-challenge/verify-signature` | Verify signature; mark `verified` (NOT written to `User`) (`:67`). | `{ signature }` → `{ verified, didKey }` |
| `POST keypair-challenge/reset` | Reset the challenge (`:76`). | – → `void` |
| `DELETE keypair-challenge/external-key` | Legacy wipe (`:82`). | – → `User` |
| `PATCH keypair-challenge/active-source` | **No-op**, kept for compatibility (`:88`). | `{ source }` → `User` |

The verified keypair is ephemeral and single-use: never persisted on `User`, it is `consumeLatestVerified()`-ed at credential-request time and snapshotted onto the pending credential. See [`07-verification-flows.md`](07-verification-flows.md).

---

## Webhooks

Controller `WebhooksController` (`src/webhooks/webhooks.controller.ts`). Public (excluded from the global gate); authenticated by svix signature over the raw body.

| Method + path (`/v1`) | Purpose | Request → response |
|---|---|---|
| `POST webhooks/clerk` | Clerk user lifecycle; svix-verified with `CLERK_WEBHOOK_SIGNING_SECRET` (`:24`). | svix event (`user.created`/`updated`/`deleted`) → `{ received: true }` |

This is the **only writer of new `User` rows** in normal operation. A DB user is created only once Clerk metadata carries `termsAreAccepted` + `termsLink`; otherwise creation is deferred to a later `user.updated`. Role is resolved from `public_metadata.role ?? unsafe_metadata.role` (only literal `'ISSUER'` → `Issuer`).

---

## Health

Controller `HealthController` (`src/health/health.controller.ts`), `VERSION_NEUTRAL` and unguarded.

| Method + path | Purpose | Request → response |
|---|---|---|
| `GET health` | `@nestjs/terminus` DB ping (`:15`). | – → terminus health report |

---

## App / `.well-known`

Root `AppController` (`src/app.controller.ts`), `VERSION_NEUTRAL`.

| Method + path | Purpose | Request → response |
|---|---|---|
| `GET /` | Hello. | – → string |
| `GET /.well-known/acme-challenge/:id` | Let's Encrypt HTTP-01; returns `${id}.${CERT_SECRET_KEY}`. | – → challenge string |

Note: there is **no** route that serves the platform's own `did:web` `did.json`. `.well-known/did.json` is only *fetched* from users' domains during did:web verification (`src/users/users.service.ts:662`), never served here.

---

## Mocks

Controller `MocksController` (`src/mocks/mocks.controller.ts`), excluded from auth. Demo scaffolding – most routes are commented out. Only the following are live:

| Method + path (`/v1`) | Purpose |
|---|---|
| `GET mocks` | Static fixture (`:32`). |
| `GET mocks/issuer/profile` | Static issuer profile (`:54`). |
| `GET mocks/creator/credentials` | Requestable-credentials fixture (`:77`). |

The `getIssuersBySelectedCredentials` handler exists in the controller but its `@Get('creator/credentials/issuers')` decorator is **commented out** (`:64`), so that route is **not** served. This matters for the discrepancies below.

---

## Discrepancies to verify

Cross-checking the UI request table (`ui-system-map.md` §3) against the backend surface surfaces three mismatches that a developer should confirm before treating them as bugs.

1. **`getRequestableCredentials` → `GET /v1/mocks/creator/credentials?issuerId=`** (`getRequestableCredentials.ts:15`). The backend *does* serve this (mocks controller `:77`), but it returns **static fixtures**, ignores `issuerId`, and is auth-excluded – and this UI fn sends **no auth token**. Confirm this is intended dev scaffolding and not a route that should be a real `/v1/credentials/...` endpoint.

2. **`getIssuersBySelectedCredentials` → `GET /v1/mocks/creator/credentials/issuers`** (`getIssuersBySelectedCredentials.ts:18`). The backend **does not serve this** – the decorator is commented out (mocks controller `:64`) – so this call 404s. It is also one of only two UI request fns that send **no auth token**. This is a live UI call with no backend counterpart; verify whether the route should be re-enabled or the caller retired.

3. **`acceptCreatorCredentialsRequest` → `POST /v1/users/creators/{creatorId}/credentials/accept`** (`ui-system-map.md` §3). **No backend counterpart exists.** `UsersController` defines `creators/:creatorId/accept|reject|revoke` (connection lifecycle) but no `.../credentials/accept` (verified via `src/users/users.controller.ts:123-147`). Credential acceptance is issuer-side per-credential at `POST /v1/credentials/:credentialId/accept` instead. Confirm whether this UI fn is dead or a bulk endpoint was planned and never built.
