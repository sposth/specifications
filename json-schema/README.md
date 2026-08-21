# Creator Credentials – Verification-Credential JSON Schemas (draft)

JSON Schemas + examples for every Verifiable Credential
the backend issues. Each schema reuses the existing EBSI-style VC 2.0
envelope from `email/schema.json`; only the `credentialSubject.properties`,
title/description, and `$id` change per type.

## Canonical schema URLs

Every `credentialSchema.id` (and each schema's `$id`) should resolve to a
**raw** GitHub file under this canonical base:

```
https://raw.githubusercontent.com/CreatorCredentials/specifications/main/json-schema/verification-credentials/<name>/schema.json
```

| VC (backend `CredentialType`) | `type` marker | Folder `<name>` | Canonical schema URL |
|---|---|---|---|
| Email (`EMAIL`) | `VerifiableEmail` | `email` | `…/verification-credentials/email/schema.json` |
| Domain (`DOMAIN`) | `VerifiableDomain` | `domain` | `…/verification-credentials/domain/schema.json` |
| DID:Web (`DID_WEB`) | `VerifiableDidWeb` | `did-web` | `…/verification-credentials/did-web/schema.json` |
| Connect (`CONNECT`) | `VerifiableDidConnect` | `connect` | `…/verification-credentials/connect/schema.json` |
| Member (`MEMBER`) | `VerifiableMembership` | `member-cert-signed` | `…/verification-credentials/member-cert-signed/schema.json` |
| Data Supplier (`DATASUPPLIER`) + Liccium Data Supplier (`LICCIUM_DATASUPPLIER`) | `VerifiableDataSupplier` | `data-supplier-cert-signed` | `…/verification-credentials/data-supplier-cert-signed/schema.json` |
| External Keypair Verification (`EXTERNAL_KEYPAIR_VERIFICATION`) | `ExternalKeypairVerification` | `external-keypair` | `…/verification-credentials/external-keypair/schema.json` |

(Full base prefix on every row:
`https://raw.githubusercontent.com/CreatorCredentials/specifications/main/json-schema/verification-credentials/…`)

Notes on the markers:
- `DataSupplier` and `LicciumDataSupplier` emit the **same** `type`
  (`VerifiableDataSupplier`) and share this one schema; they are distinguished
  only by the DB `credential_type`.
- `ExternalKeypairVerification` is deliberately **not** prefixed `Verifiable`,
  and its `credentialSubject` has **no `id`** (subject fields are
  `email`, `sameAs`, optional `organizationName`).
- `Wallet` is intentionally out of scope – no schema is authored here.

## credentialSubject shapes

| `<name>` | `credentialSubject` fields |
|---|---|
| `email` | `id` (did), `email` |
| `domain` | `id` (did), `domain` |
| `did-web` | `id` (did), `didWeb` |
| `connect` | `id` (did), `sameAs` (did) |
| `member-cert-signed` | `id` (did), `memberOf` (string/did) |
| `data-supplier-cert-signed` | `id` (did), `dataSupplierFor` (string), `sameAs` (string, optional) |
| `external-keypair` | `email` (string), `sameAs` (did), `organizationName` (string, optional) |

## Developer note – the backend currently emits `credentialSchema.id`s that 404

Per `notes/old-spec-audit.md` §JSON schemas, the code
(`credentials.helpers.ts`, `credentials.service.ts`) emits
`credentialSchema.id` URLs that do **not** resolve today. They should be
updated to the raw canonical URLs above. Specifically:

1. **`/blob/main/` HTML path** – every emitted URL uses
   `github.com/CreatorCredentials/specifications/blob/main/…`, which is a GitHub
   **HTML page**, not raw JSON. A resolver fetching it gets HTML. Switch to
   `raw.githubusercontent.com/CreatorCredentials/specifications/main/…`.
2. **`domain` vs `domain-name` path mismatch** – the code emits
   `…/domain/schema.json` (used by both the Domain **and** DID:Web VCs), but the
   repo only had `domain-name/`. This draft settles on `domain/` (matching the
   emitted URL) and adds a **distinct** `did-web/schema.json` so the DID:Web VC
   stops borrowing the domain schema (its subject field is `didWeb`, not
   `domain`).
3. **Five previously-missing schemas** – Member (`member-cert-signed`),
   DataSupplier (`data-supplier-cert-signed` – the old `supplier/` schema was
   mis-authored with a `domainName` subject), Connect (was borrowing the email
   schema), DID:Web (was borrowing the domain schema), and External-Keypair
   (`external-keypair`) had **no** correct schema. All are added here.

Once these files live at the canonical location, update the backend to emit the
raw URLs above so every issued VC's `credentialSchema.id` resolves to the schema
that actually describes it.
