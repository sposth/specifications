# Verifiable Credential Catalog

> **Status: DRAFT** – proposal for the `specifications` repo, pending developer approval. Reflects code on `creator-credentials-backend@main` as of 2026-07-10.

This is the definitive list of every Verifiable Credential (VC) the Creator Credentials backend issues: its claims schema, issuance trigger, signing path, uniqueness, and revocation behaviour. It is the highest-priority reference for anyone integrating with, verifying, or extending the credential surface.

Related specs:

- [`04-connections-and-issuance.md`](./04-connections-and-issuance.md) – the request → accept → verify-signature issuance flow and the issuer↔creator connection lifecycle.
- [`06-signing-and-trust-model.md`](./06-signing-and-trust-model.md) – the signing keys, certificates, and DID resolution in depth.
- [`09-data-model.md`](./09-data-model.md) – the `credential` row, `credential_status`, and persistence.

Object builders live in `src/credentials/credentials.helpers.ts` (a few inline in `credentials.service.ts`); read-side formatters in `credentials.formatters.ts`; the enum in `src/shared/typings/Credentials.ts`.

---

## Common shape (W3C VC 2.0)

Every issued VC is a W3C VC 2.0 object with the same envelope:

```jsonc
{
  "@context": ["https://www.w3.org/ns/credentials/v2"],
  "id": "urn:uuid:<uuid v4>",
  "type": ["VerifiableCredential", "VerifiableAttestation", "<TypeSpecific>"],
  "issuer": "<DID string>",
  "validFrom": "<now, ISO 8601>",
  "validUntil": "<now + 3 years, ISO 8601>",   // hardcoded 3y everywhere
  "credentialSubject": { /* per-type, see below */ },
  "credentialSchema": [
    { "id": "<GitHub schema URL>", "type": "JsonSchema" }
  ],
  "termsOfUse": {
    "type": "PresentationPolicy",
    "confidentialityLevel": "<public | restricted>",
    "pii": "<none | sensitive>"
  }
}
```

Notes on the envelope:

- **Type chain.** The first two entries are always `VerifiableCredential` and `VerifiableAttestation`; the third is the type-specific marker (e.g. `VerifiableEmail`). Two credential types share the same third marker – see the Data Supplier subsections.
- **`validUntil` is a fixed 3-year stamp** (`end.setFullYear(end.getFullYear() + 3)`, e.g. `credentials.helpers.ts:76`). Nothing re-checks expiry server-side; see [Revocation](#revocation).
- **`issuer` DID.** Platform-self-issued VCs use `issuer: 'did:web:liccium.com'` (hardcoded `credentialsHost = 'liccium.com'`, `credentials.helpers.ts:13`). Issuer-issued VCs derive the issuer DID from the issuer's certificate CN/SAN (`resolveIssuerDidFromCert`, `credentials.helpers.ts:46-66`), falling back to `did:web:<domain>` / the stored `didWeb` / `did:web:liccium.com`.
- **`credentialSubject.id`** is the subject's **did:key**: `resolveDidKey(user)` returns the external did:key when `activeDidKeySource === 'external'`, otherwise the platform did:key derived from the user's self-signed cert (`credentials.helpers.ts:15-20`); it may instead be the did:key just proven through a keypair challenge when one was consumed.

### Persistence

Each VC is stored as one `credential` row:

- `credential_object` – the VC object above.
- `token` – the detached JWS/JWT (the signature; see [Signing paths](#signing-paths)).
- `credential_type` – the `CredentialType` enum value (this, not the `type` array, is what distinguishes the two Data Supplier variants).
- `credential_status` – `PENDING` or `SUCCESS`.

On read, the formatters wrap the object and attach the proof from the token: `proof: { type: 'JwtProof2020', jwt: <token> }` (`credentials.formatters.ts`). The `proof` is therefore materialised at read time, not stored inside `credential_object`. See [`09-data-model.md`](./09-data-model.md).

---

## Summary of all types

| Enum value (`CredentialType`) | User-facing label | Self- vs issuer-issued | Subject id | Signing path | Trigger |
|---|---|---|---|---|---|
| `EMAIL` | E-Mail | Self-issued | `resolveDidKey(user)` | Path 1 (RS256 helper) | `POST /v1/credentials/create/email` + auto on user create |
| `DOMAIN` | Domain | Self-issued | `resolveDidKey(user)` | Path 1 (RS256 helper) | Domain TXT-record verification (cron-confirmed) |
| `DID_WEB` | did:web | Self-issued | `resolveDidKey(user)` | Path 2 (jose ES256) | `did:web` well-known verification (cron-confirmed) |
| `CONNECT` | Connect | Self-issued | user's did:key | Path 1 (RS256 helper) | `POST /v1/users/did-liccium/connect` / `POST /v1/credentials/export` |
| `EXTERNAL_KEYPAIR_VERIFICATION` | Keypair Verification | Self-issued | (n/a – `sameAs`) | Path 1 (RS256 helper) | Side-effect of a Data Supplier request |
| `MEMBER` | Member | Issuer-issued | `user.didKey` | Path 3 (issuer cert) | `POST /v1/credentials/request` (`MEMBER`) → accept flow |
| `DATASUPPLIER` | Open Future Data Supplier | Issuer-issued | proven did:key | Path 3 (issuer cert) | `POST /v1/credentials/request` (`DATASUPPLIER`) → accept flow |
| `LICCIUM_DATASUPPLIER` | Liccium Data Supplier | Issuer-issued | proven did:key | Path 3 (issuer cert) | `POST /v1/credentials/request` (`LICCIUM_DATASUPPLIER`) → accept flow |
| `STUDENT` | (none surfaced) | Planned / not implemented | – | – | – (no builder, no path) |

User-facing labels are from `public/locales/en/cards.json` → `credential.types.*` (UI repo); `did:web` label from `verification-cards.json`. The **enum string values** are `'EMAIL'`, `'DOMAIN'`, `'DID_WEB'`, `'CONNECT'`, `'EXTERNAL_KEYPAIR_VERIFICATION'`, `'MEMBER'`, `'DATASUPPLIER'`, `'LICCIUM_DATASUPPLIER'`, `'STUDENT'` (the enum also carries `'WALLET'`, which is Wallet/MetaMask – out of scope for this doc set; see [`09-data-model.md`](./09-data-model.md)).

---

## Self-issued VCs

These are platform-signed. There is no issuer and no accept flow – the backend mints them directly once a proof-of-control step completes.

### Email – `CredentialType.EMail = 'EMAIL'`

- **Purpose:** attests the creator's or issuer's verified email address.
- **Builder:** `generateEmailCredentialObjectAndJWS` (`credentials.helpers.ts:285-320`).
- **Type marker:** `VerifiableEmail`.
- **Claims (`credentialSubject`):** `{ id: resolveDidKey(user), email }`.
- **`termsOfUse`:** `pii: sensitive`, `confidentialityLevel: restricted`. Schema `.../email/schema.json`.
- **Trigger:** `POST /v1/credentials/create/email` (`credentials.controller.ts:51`) and, automatically, on user creation / the `users/check` back-fill (`users.service.ts:167, 188`).
- **Signing:** path 1 (RS256, platform `x5c`).
- **Status / uniqueness:** created `SUCCESS` immediately; one-per-user (ConflictException + partial unique index).
- **Revocation:** row deletion via `removeEmailCredential` on re-provision.

### Domain – `CredentialType.Domain = 'DOMAIN'`

- **Purpose:** attests control of a DNS domain (creator or issuer side).
- **Builder:** `generateDomainCredentialObjectAndJWS` (`credentials.helpers.ts:322-357`).
- **Type marker:** `VerifiableDomain`.
- **Claims (`credentialSubject`):** `{ id: resolveDidKey(user), domain }`.
- **Trigger (two-phase):** `POST /v1/users/verification/domain/txt-record` returns a TXT value (`cc-verification=...`, prefix in `credentials.constants.ts`); `.../confirm` writes a **PENDING** domain credential and arms polling. The `@Cron(EVERY_10_SECONDS) checkUserDomains` poller resolves the domain's TXT records and, on match, calls `createDomainCredential`, replacing the PENDING row with a signed **SUCCESS** VC (`users.service.ts:554-591`).
- **Signing:** path 1 (RS256, platform `x5c`).
- **Status / uniqueness:** `PENDING → SUCCESS` via cron, or deleted on disconnect; partial unique index enforces one-`DOMAIN`-per-user.
- **Revocation:** row deletion on domain disconnect.

### DID:Web – `CredentialType.DidWeb = 'DID_WEB'`

- **Purpose:** attests control of a `did:web` (via a hosted `.well-known/did.json`). Typically an issuer-side identity proof.
- **Builder:** inline in `createDidWebCredential` (`credentials.service.ts:415-497`).
- **Type marker:** `VerifiableDidWeb`.
- **Claims (`credentialSubject`):** `{ id: resolveDidKey(user), didWeb }`. Reuses the domain JSON-schema URL.
- **Trigger (two-phase):** `POST /v1/users/verification/did-web/well-known` returns the `did.json` to host; `.../confirm` writes a PENDING row and arms polling. `@Cron(EVERY_10_SECONDS) checkUsersDidWeb` fetches `https://<didWeb>/.well-known/did.json` (with `rejectUnauthorized: false` – it accepts self-signed TLS, `users.service.ts:662-666`) and compares `verificationMethod[0].publicKeyJwk.x`; on match it calls `createDidWebCredential` → `SUCCESS`.
- **Signing:** path 2 (jose **ES256**, `SIGNATURE_KEY_*`).
- **Status / uniqueness:** `PENDING → SUCCESS` via cron, or deleted on disconnect.
- **Revocation:** row deletion on did:web disconnect.

### Connect – `CredentialType.Connect = 'CONNECT'`

- **Purpose:** binds the creator's Creator-Credentials did:key to their **Liccium app** did:key (`sameAs`), enabling cross-app credential import.
- **Builder:** `generateConnectCredentialObjectAndJWS` (`credentials.helpers.ts:359-396`).
- **Type marker:** `VerifiableDidConnect`.
- **Claims (`credentialSubject`):** `{ id: didKey, sameAs: licciumDidKey }`. Reuses the **email** schema URL (likely a copy-paste bug; unverified whether intentional).
- **Trigger:** `POST /v1/users/did-liccium/connect` (`connectLicciumDidKeyToUser`, `users.service.ts:487`) or the public cross-app import `POST /v1/credentials/export` (`credentials.controller.ts:383-387`).
- **Signing:** path 1 (RS256, platform `x5c`).
- **Status / uniqueness:** `SUCCESS`. Not conflict-blocked: an existing Connect credential is deleted and re-created (`credentials.service.ts:264-269`).
- **Revocation:** row deletion via `removeConnectCredential` on disconnect / re-issue.

### External Keypair Verification (EKVC) – `CredentialType.ExternalKeypairVerification = 'EXTERNAL_KEYPAIR_VERIFICATION'`

- **Purpose:** platform-issued proof that the creator owns an external EC P-256 keypair (the one proven via the keypair challenge). It is a supporting credential for the Data Supplier VCs.
- **Builder:** `generateExternalKeypairVerificationCredentialObjectAndJWS` (`credentials.helpers.ts:241-283`).
- **Type marker:** `ExternalKeypairVerification` – note this is **not** prefixed `Verifiable`.
- **Claims (`credentialSubject`):** `{ email, sameAs: derivedDidKey, organizationName? }`. `issuer: did:web:liccium.com`. Note the subject has no `id` field; the proven did:key is carried in `sameAs`.
- **Trigger:** a **side-effect**, not a direct endpoint. Auto-created (`SUCCESS`) whenever a creator requests a **Data Supplier** (with `organizationName`) or **Liccium Data Supplier** VC – `createPendingDataSupplierCredential` / `createPendingLicciumDataSupplierCredential` mint it from the consumed keypair snapshot (`credentials.service.ts:691-707, 869-884`).
- **Signing:** path 1 (RS256, platform `x5c`).
- **Status:** `SUCCESS`. No explicit revocation path (deleted with the user).

---

## Issuer-issued VCs

All three follow the same request → accept → verify-signature choreography (full detail in [`04-connections-and-issuance.md`](./04-connections-and-issuance.md)):

1. Creator `POST /v1/credentials/request` → a `PENDING` row is created.
2. Issuer `POST /v1/credentials/:id/accept` → the backend returns a `signingInput` challenge, an openssl command, and a "supporting credential".
3. Issuer signs the `signingInput` offline with their eIDAS certificate's private key.
4. Issuer `POST /v1/credentials/:id/accept/verify-signature` → the backend verifies the signature against `issuer.externalCertPem`, flips the row to `SUCCESS`, and stores the issuer-produced JWS as `token`.

The pending type is dispatched by `getPendingCredentialType` (`credentials.controller.ts:468-518`).

**Precondition for all three:** the issuer must have completed the certificate challenge (`externalCertPem` present) or the request/accept is rejected (`credentials.service.ts:539, 716, 893`; `credentials.controller.ts:221-225`).

### Member(ship) – `CredentialType.Member = 'MEMBER'`

- **Purpose:** the issuer attests that the creator is a member of the issuer's organisation.
- **Builder:** `generateMembershipCredentialObjectAndJWS` (`credentials.helpers.ts:68-119`).
- **Type marker:** `VerifiableMembership`.
- **Claims (`credentialSubject`):** `{ id: subjectDidKey, memberOf: resolveMemberOf(issuer) }`, where `memberOf` is `did:web:<issuer.domain>` / the issuer's stored `didWeb` / `urn:issuer:<id>`. `issuer` comes from the cert when `activeSigningCertSource === 'external'`, else `resolveIssuerDid`. Schema `.../member-cert-signed/schema.json`; `pii: sensitive`.
- **Trigger:** `POST /v1/credentials/request` with `credentialType: MEMBER`. The subject did:key defaults to `user.didKey` (no keypair challenge required; `credentials.controller.ts:271-275`). The supporting credential returned at accept is the creator's **Email** VC (`findMembershipSupportingCredential`, `credentials.service.ts:618-637`).
- **Signing:** path 3 (issuer's external cert signature; falls back to helper path 1 while building the unsigned *draft*).
- **Status / uniqueness:** `PENDING → SUCCESS` (verify-signature) or deleted (reject); one-pending-per-(issuer, user).
- **Revocation:** see [Revocation](#revocation) – no status list; issuer reject / delete only.

### Data Supplier – `CredentialType.DataSupplier = 'DATASUPPLIER'`

- **Purpose:** the issuer (e.g. Open Future) attests that the creator is an authorised data supplier for a registry.
- **Builder:** `generateDataSupplierCredentialObjectAndJWS` (`credentials.helpers.ts:126-181`).
- **Type marker:** `VerifiableDataSupplier`.
- **Claims (`credentialSubject`):** `{ id: subjectDidKey, dataSupplierFor, sameAs: organizationName? }`. The `issuer` is **always** derived from the cert (not the profile). Hardcoded workaround: if the issuer DID contains `openfuture.eu`, `dataSupplierFor` is forced to `registry.commonsdb.org` (`credentials.helpers.ts:123-124, 145-147`). Schema `.../data-supplier-cert-signed/schema.json`; `pii: none`, `confidentialityLevel: public`.
- **Trigger:** `POST /v1/credentials/request` with `credentialType: DATASUPPLIER`. Requires (`credentials.controller.ts:232-253`): (a) `user.organizationName` is set; (b) a **consumed verified keypair challenge** (`consumeLatestVerified`) – the proven did:key becomes the subject id; (c) the issuer has `externalCertPem`. Auto-creates the EKVC (above) as the supporting credential (`findDataSupplierSupportingCredential`, `credentials.service.ts:796-815`). Hidden from creators until the issuer has a cert (`users.service.ts:399-402`, `templates.controller.ts:46-52`).
- **Signing:** path 3.
- **Status / uniqueness:** `PENDING → SUCCESS` or deleted.
- **Revocation:** see [Revocation](#revocation).

### Liccium Data Supplier – `CredentialType.LicciumDataSupplier = 'LICCIUM_DATASUPPLIER'`

- **Purpose:** the Liccium-flavoured data-supplier VC. Same machinery as Data Supplier, but with no org-name requirement.
- **Builder:** `generateLicciumDataSupplierCredentialObjectAndJWS` (`credentials.helpers.ts:183-239`).
- **Type marker:** `VerifiableDataSupplier` – the **same** type array as `DATASUPPLIER`; the two are distinguished only by the DB `credential_type`.
- **Claims (`credentialSubject`):** as Data Supplier. `dataSupplierFor` is `registry.commonsdb.org` when the issuer is Open Future, else the DTO `value`. `issuer` from the cert when the source is external, else `resolveIssuerDid`. Same schema and terms as Data Supplier.
- **Trigger:** `POST /v1/credentials/request` with `credentialType: LICCIUM_DATASUPPLIER`. Requires a consumed verified keypair challenge; **no** org-name requirement (`credentials.controller.ts:254-269`). Auto-creates the EKVC without an org name.
- **Signing:** path 3.
- **Status / uniqueness:** `PENDING → SUCCESS` or deleted.
- **Revocation:** see [Revocation](#revocation).

---

## `STUDENT` – Planned / not implemented

`CredentialType.Student = 'STUDENT'` exists in the enum, in `CredentialTemplateType.Student`, and in the template-type map (`templates.controller.ts:17`), but has **no builder and no create/request path**. It is dead / future scaffolding – nothing issues it today. Treat it as **Planned / not implemented**.

---

## Signing paths

There are exactly three signing paths. Which one a VC uses is fixed per type (see the summary table). Full key and certificate detail is in [`06-signing-and-trust-model.md`](./06-signing-and-trust-model.md).

1. **`signJWTWithX5c(payload, issuerCertPem?)`** (`credentials.helpers.ts:422-453`) – `jsonwebtoken.sign`, **RS256**, private key from `HALCOM_CERT_PRIVATE_KEY`. The `x5c` header carries either the supplied issuer cert (external eIDAS cert, base64 DER stripped from PEM) or, when none is supplied, the **platform cert loaded from `./certificates/LICCIUM.der`**. This is the default path for Email, Domain, DID:Web (draft), Connect, EKVC, and cert-less Member drafts.
   - Caveat: the private key is **always** `HALCOM_CERT_PRIVATE_KEY`, even when the `x5c` header carries an issuer's own cert. For the "issuer-signed via helper" draft path, the header cert and the signing key can therefore be mismatched. The genuinely issuer-signed path is path 3.
2. **`jose.CompactSign` ES256** (`credentials.service.ts:168-182, 467-481`) – EC P-256 key from `SIGNATURE_KEY_D/X/Y`. Used by the inline **DID:Web** builder (also the out-of-scope Wallet builder). (The Email builder has this path commented out and now uses path 1.)
3. **Issuer's external X.509 signature** (`credentials.service.ts:988-1029`) – for the Member / Data Supplier / Liccium Data Supplier *accept* flow. The issuer signs the `header.payload` (`buildSigningInput`, `x5c` = issuer cert, alg `ES256 | RS256` per the cert's key type) offline with their private key; the backend verifies with `crypto.createVerify('SHA256')` against `issuer.externalCertPem` and stores `token = signingInput.signatureB64url`. This is a real detached JWS the issuer produced with their own key.

**No XAdES signing on VC issuance.** `xadesjs` (`cert-challenge/trust-store/eidas/xades-verifier.ts`) is used only to *verify* eIDAS LOTL/TL signatures, never to sign VCs.

---

## Revocation

**Current reality: there is no cryptographic revocation.** No status list, no revocation VC, no `credentialStatus` entry in the VC object. "Revocation" today means one of:

- **Row deletion** – Email / Domain / DID:Web / Connect on disconnect or re-provision; Member / Data Supplier via `POST /v1/credentials/:id/reject` or `DELETE /v1/credentials/:id` (issuer-only, `credentials.controller.ts:521-544`).
- **Connection flip** – `POST /v1/users/creators/:id/revoke` sets `ConnectionStatus.REVOKED` (`connections.service.ts:82-96`). Revoked connections are filtered out of issuer↔creator reads but **do not delete or invalidate already-issued VCs**.
- **No `validUntil` enforcement** anywhere beyond the 3-year stamp written into each VC; nothing re-checks expiry server-side.

Consequently, a VC that has been "revoked" by connection flip is still a valid, verifiable, unexpired object in any holder's possession. A verifier cannot today learn revocation status from the credential.

**Planned / not implemented:** a **Bitstring Status List** (W3C `credentialStatus` / Bitstring Status List v1.0) is the intended path to real, checkable revocation. It is not yet implemented; when it lands it should be specified in its own sibling doc and cross-linked here.
