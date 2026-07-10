# Creator Credentials (CC) Verifiable Credentials (VC) profile

> **Status: DRAFT** – proposal for the `specifications` repo, pending developer approval. Reflects code as of 2026-07-10.

This profile documents the system **as built**. It records the DID methods, the
VC data model, and – at a high level – how credentials are signed. It is
deliberately short and normative; the detailed, code-grounded breakdowns live in
the numbered technical reference (linked below).

## DID Methods

- Legal Entities: [`did:web`](https://w3c-ccg.github.io/did-method-web/)
- Natural Persons: [`did:key`](https://hub.ebsi.eu/vc-framework/did/did-methods/natural-person)

This split matches the code: an issuer identified by a verified domain resolves
to `did:web`, while a natural-person subject resolves to a `did:key` derived from
their key material (`resolveDidKey` / `resolveIssuerDidFromCert`;
`06-signing-and-trust-model.md`).

## Verifiable Credentials data model

- VC data model: [Verifiable Credentials data model v2](https://w3c.github.io/vc-data-model/)

## Signature profile

Credentials are issued as JWS. The backend uses **three** concrete signing paths,
not one: the default platform path (**RS256** JWT with an `x5c` header, signed
with the platform key), a legacy JOSE **ES256** path used only for the Wallet and
DID:Web credentials, and a detached issuer-signed JWS produced during the
cert-challenge acceptance flow. Which path a given credential type takes is fixed
in code. For the exact algorithms, keys, headers, per-type mapping, and the eIDAS
LOTL trust model behind issuer certificates, see
[`06-signing-and-trust-model.md`](06-signing-and-trust-model.md).

## Verifiable Credentials Exchange profile

**Planned / not implemented.** No Verifiable-Presentation, holder-wallet, or
credential-exchange surface exists in the product today – there is no VP creation,
no authorization-request endpoint, and no verifier surface in either repository.
The earlier reference to the EBSI holder-wallet functional flows described an
unbuilt design and has been dropped from the as-built profile; it may be re-specced
if and when an exchange surface ships.
