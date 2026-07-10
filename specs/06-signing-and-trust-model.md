# Signing & trust model

> **Status: DRAFT** – proposal for the `specifications` repo, pending developer approval. Reflects code as of 2026-07-10.

This document specifies how the Creator Credentials backend signs Verifiable Credentials, which DIDs it uses, and how it decides whether an issuer's external eIDAS certificate is trustworthy. It complements the credential-by-credential breakdown in [`03-verifiable-credentials-catalog.md`](03-verifiable-credentials-catalog.md) and the endpoint surface in [`08-api-reference.md`](08-api-reference.md).

It supersedes the retired `signature-profile.md` and refines the surviving `10-profile.md`:

- `signature-profile.md` (retired) – its abstract JAdES/EBSI framing is replaced by this code-grounded account of the signature formats actually emitted (RS256 JWT with `x5c`, ES256 JOSE compact, and detached issuer JWS).
- [`10-profile.md`](10-profile.md) – the DID conventions below (platform `did:web:liccium.com`, subject `did:key`) are the authoritative statement of what the running system uses.

---

## 1. Signing paths

The backend emits credentials via **three distinct signing paths**. Which path a given credential type uses is fixed in code, not configurable.

### Path 1 – RS256 JWT with `x5c` header (default platform path)

Helper: `signJWTWithX5c(payload, issuerCertPem?)` (`src/credentials/credentials.helpers.ts:422-453`).

- Algorithm: **RS256** via `jsonwebtoken.sign`.
- Signing key: **always** `HALCOM_CERT_PRIVATE_KEY` (PEM, with `\n` unescaped), regardless of whose cert appears in the header (`credentials.helpers.ts:425-428`).
- `x5c` header:
  - If `issuerCertPem` is supplied, the cert's DER (PEM header/footer and whitespace stripped) is placed in `x5c` (`credentials.helpers.ts:434-439`).
  - If not, the **platform certificate** is loaded from `./certificates/LICCIUM.der` and used instead (`credentials.helpers.ts:440-442`).

**Used by:** email, domain, did:web, connect, and EKVC. It also builds the *draft* Member / Data Supplier VC object, but that draft's JWS is **discarded** – the only stored signature for those types is the Path 3 issuer detached JWS.

> **Caveat (mismatch risk).** On this path the signing key is always `HALCOM_CERT_PRIVATE_KEY`, even when the `x5c` header carries an issuer's own certificate. The genuinely issuer-signed path is Path 3 below (`src/credentials/credentials.helpers.ts:425`).

### Path 2 – JOSE ES256 compact (legacy inline path)

Implementation: `jose.CompactSign` with an EC P-256 key rebuilt from `SIGNATURE_KEY_D` / `SIGNATURE_KEY_X` / `SIGNATURE_KEY_Y` (`src/credentials/credentials.service.ts:168-182` and `:467-481`).

- Algorithm: **ES256** (EC P-256).
- Signing key: the static platform EC key from the three `SIGNATURE_KEY_*` env vars.

**Used by:** the inline **did:web** VC builder (also the out-of-scope Wallet builder). The email builder once used this path; it is now commented out there and email uses Path 1 (`credentials.service.ts:168`).

### Path 3 – Issuer external X.509 detached JWS (accept flow)

Implementation: `buildSigningInput` + `executeSignatureVerification` (`src/credentials/credentials.service.ts:988-1029`, `:1048-1064`).

This is the only path where the issuer's own private key produces the signature. For the Member / DataSupplier / LicciumDataSupplier **accept** flow:

1. The backend builds `signingInput = base64url(header{x5c, alg}) . base64url(payload)`, where `x5c` is the issuer's cert and `alg` is `ES256` or `RS256` chosen from the cert's key type (`buildSigningInput`, `credentials.service.ts:1048-1064`).
2. The issuer signs `header.payload` **offline** with their eIDAS certificate's private key (an `openssl dgst -sha256 -sign` command is returned to them).
3. The backend verifies the returned signature with `crypto.createVerify('SHA256')` against `issuer.externalCertPem`, and on success stores `token = signingInput.signatureB64url` (`credentials.service.ts:988-1029`).

The result is a genuine detached JWS the issuer produced with their own key. See Part (b) of [`04-connections-and-issuance.md`](04-connections-and-issuance.md).

> **No XAdES on issuance.** `xadesjs` is used only to verify eIDAS LOTL/TL signatures (§3), never to sign VCs (`credentials.helpers.ts` has no XAdES call; `cert-challenge/trust-store/eidas/xades-verifier.ts` is verification-only).

### Path selection at a glance

| Credential type | Signing path |
|---|---|
| Email | 1 (RS256, platform `x5c`) |
| Domain | 1 (RS256, platform `x5c`) |
| Connect | 1 (RS256, platform `x5c`) |
| External Keypair Verification (EKVC) | 1 (RS256, platform `x5c`) |
| did:web | 2 (JOSE ES256, `SIGNATURE_KEY_*`) |
| Member | 3 (issuer detached JWS) |
| Data Supplier | 3 (issuer detached JWS) |
| Liccium Data Supplier | 3 (issuer detached JWS) |

---

## 2. DIDs

### Platform issuer DID

Platform-self-issued VCs use a hardcoded `issuer: 'did:web:liccium.com'` (`credentialsHost = 'liccium.com'`, `src/credentials/credentials.helpers.ts:13`).

> There is **no route serving the platform's own `.well-known/did.json`** – `did:web:liccium.com` is asserted but not resolvable from this backend. `.well-known/did.json` is only *fetched* from users' domains during did:web verification (`src/users/users.service.ts:662`), never served (see the AppController note in the system map).

### Subject DID (`credentialSubject.id`)

Resolved by `resolveDidKey(user)` (`src/credentials/credentials.helpers.ts:15-20`):

- If `user.activeDidKeySource === 'external'` and `user.externalDidKey` is set, use the **external `did:key`**.
- Otherwise use the platform `did:key` (`user.didKey`), which is derived from the public key of the user's per-user self-signed X.509 certificate at provisioning time (`publicKeyPemToDid`, `src/users/users.service.ts:73-123`).
- For DataSupplier / LicciumDataSupplier, the subject id is instead the **just-consumed keypair-challenge `did:key`** – the external EC P-256 key the creator proved possession of (snapshotted at request time; see §04 (external keypair) in [`07-verification-flows.md`](07-verification-flows.md)).

`did:key` math (P-256 point compress/decompress, base58btc, multicodec `0x1200`) lives in `src/shared/did-key.util.ts`.

### Issuer DID for issuer-issued VCs

For issuer-signed credentials the `issuer` field is derived **from the issuer's certificate**, not from a profile field, by `resolveIssuerDidFromCert(certPem)` (`src/credentials/credentials.helpers.ts:46-66`):

1. Prefer the certificate SAN DNS entry (`DNS:<value>`), returned as-is if already `did:web:`, else wrapped `did:web:<value>`.
2. Else the certificate subject CN (`CN=<value>`).
3. Else fall back to `did:web:liccium.com`.

DataSupplier and LicciumDataSupplier **always** take `issuer` from the cert. Member takes it from the cert when `activeSigningCertSource === 'external'`, else from `resolveIssuerDid` (which prefers `issuer.didWeb`, then `did:web:<issuer.domain>`, then `did:web:liccium.com`; `credentials.helpers.ts:34-38`).

---

## 3. eIDAS trust store

Issuer certificates are not trusted blindly. Before an issuer's external certificate is accepted (the cert-challenge flow in `07-verification-flows.md` §5), it is validated against an in-memory trust store built from the EU eIDAS **List of Trusted Lists (LOTL)**.

### What it is

An in-memory anchor set of the QSeal/QSig CA certificates published across the EU member-state Trusted Lists, keyed by fingerprint and subject DN, held by `TrustStoreService` (`src/cert-challenge/trust-store/`).

### How it is built

The `EidasPipelineService.collectAnchors` pipeline (`src/cert-challenge/trust-store/eidas/`):

1. Load the top-level **LOTL** from `EIDAS_LOTL_URL` (`LotlLoaderService`, with `LotlPivotBootstrap` for signer bootstrapping).
2. Follow each per-country **Trusted List (TL)** pointer (`TlLoaderService`).
3. **XAdES-verify** each list's signature (`XadesVerifier`, `xadesjs`) against the expected signer certificates (from `EIDAS_LOTL_SIGNERS_DIR`).
4. Collect the verified QSeal/QSig CA certificates as trust anchors.
5. Refuse to publish an empty anchor set (fail rather than trust nothing).

### Refresh lifecycle

`TrustStoreRefreshService`:

- Bootstrap refresh on `OnApplicationBootstrap` (non-blocking, so boot is not delayed).
- Daily refresh via `@Cron(EVERY_DAY_AT_3AM)`.

The store is **fail-closed until ready**: `TrustStoreService.isReady()` is false until a non-empty anchor set has been published, and chain validation depends on it (subject to the dev escape hatch below).

### Certificate validation (`CertValidatorService`)

On `POST /cert-challenge/submit-cert`, `validateLeafPem` runs, in order (`src/cert-challenge/validation/cert-validator.service.ts:55-190`):

1. Format parse + validity window (not-before / not-after).
2. **Algorithm allowlist:** SHA-256 / SHA-384 / SHA-512; RSA ≥ 2048; EC P-256 / P-384 / P-521.
3. **KeyUsage allowlist:** `digitalSignature` or `nonRepudiation`.
4. **eIDAS chain check:** candidate anchors are found by issuer DN (`findCandidateIssuers`), then the leaf must `cert.verify({ publicKey: anchor, signatureOnly })` **directly** against an anchor – no intermediate certificates are followed (`cert-validator.service.ts:85-127`).

### Security caveats (dev escape hatches)

These are deliberate development conveniences that **must be closed before production**:

1. **Chain check skipped when the store is not ready.** If `!trustStore.isReady()`, chain validation is bypassed and the certificate passes on algorithm + key-usage checks alone (`src/cert-challenge/validation/cert-validator.service.ts:78-83`). During the bootstrap window, or any time the LOTL refresh has not populated anchors, any well-formed cert with an allowed algorithm and key usage is accepted.
2. **Cert-challenge TTL not enforced.** The 60-minute challenge expiry (`expiresAt`) is written to the row but the expiry check on `verify-signature` is commented out (`src/cert-challenge/cert-challenge.service.ts:110-117`). Challenges do not currently time out.

---

## 4. Standards alignment

- **W3C Verifiable Credentials 2.0** – every issued VC is a VC 2.0 object (`@context: ['https://www.w3.org/ns/credentials/v2']`, `validFrom`/`validUntil`, `credentialSubject`, `credentialSchema`, `termsOfUse`). On read it is wrapped with `proof: { type: 'JwtProof2020', jwt: <token> }`. See [`03-verifiable-credentials-catalog.md`](03-verifiable-credentials-catalog.md) §Common shape.
- **did:web / did:key** – platform issuer identity uses `did:web`; subjects use `did:key` derived from EC/cert public keys. This matches the DID conventions in [`10-profile.md`](10-profile.md).
- **eIDAS / JAdES direction** – issuer certificates are validated against the eIDAS LOTL trust store, and the issuer accept flow already produces a detached JWS over `x5c`+`alg` header material (Path 3). That JWS is the natural substrate for a JAdES profile. Current issuance is **not** yet JAdES-conformant – there is no XAdES/JAdES signature-level packaging on the VC itself. A future JAdES profile is the intended target; this document describes the current state.

> **Refinement note.** The earlier profile documents described a single uniform signature format; the running system uses the three paths in §1. This document is the ground truth for the current implementation.
