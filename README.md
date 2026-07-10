# CreatorCredentials – specifications

> **Proposed rewrite (draft), pending developer approval.** This is a
> ground-truth technical reference reconstructed from the current code
> (`creator-credentials-backend@main`, `creator-credentials-ui@develop`,
> read 2026-07-10). It supersedes the earlier design specs, which described a
> system that was largely not built (OIDC IdP, a three-role "Host" model,
> did:web-everywhere, Bitstring-Status-List revocation, discovery, and
> log-in-with-CC). Those unbuilt-but-intended designs are preserved under
> [`future/`](future/).

Creator Credentials is a software application that media organisations,
membership organisations, and other trust services use to issue Verifiable
Credentials to creators and other rightsholders. It is a **Clerk**-authenticated
two-role app (**Issuer** and **Creator**); the platform (Liccium) self-signs a
set of Verification Credentials and issuers sign membership / data-supplier
credentials with their own eIDAS certificate.

## Technical reference

Read in order; each doc is cited to `path:line` in the code.

1. [Architecture overview](specs/01-architecture-overview.md) – services, ports, data flow.
2. [Roles & actors](specs/02-roles-and-actors.md) – platform vs Issuer vs Creator (there is no "Host" role).
3. [Verifiable Credential catalog](specs/03-verifiable-credentials-catalog.md) – every VC type the backend issues.
4. [Connections & issuance](specs/04-connections-and-issuance.md) – the creator↔issuer connection lifecycle and the request → accept → cert-signed verify state machine.
5. [Authentication & provisioning](specs/05-auth-and-provisioning.md) – Clerk, the svix webhook, and the terms gate.
6. [Signing & trust model](specs/06-signing-and-trust-model.md) – the three signing paths and the eIDAS LOTL trust store.
7. [Verification flows](specs/07-verification-flows.md) – email / domain / did:web / external-keypair / issuer eIDAS cert.
8. [API reference](specs/08-api-reference.md) – the real `/v1` HTTP surface.
9. [Data model](specs/09-data-model.md) – entities, enums, and the migration-derived schema.

- [Profile](specs/profile.md) – the normative DID-method and data-model profile.

## Data models and schemas

JSON Schemas + examples for the issued Verifiable Credentials:
[`json-schema/verification-credentials/`](json-schema/verification-credentials/).
See the [schema index](json-schema/README.md) for the canonical schema URL of
each VC type. **Note for developers:** the backend currently emits several
`credentialSchema.id` URLs that do not resolve – tracked in the schema index.

## Planned / not yet implemented

Parked designs, clearly labelled, kept so the intent is not lost:
[`future/`](future/) – Bitstring-Status-List revocation, host/issuer discovery,
and log-in-with-Creator-Credentials.

## Reference

- did:web Method Specification: https://w3c-ccg.github.io/did-method-web/
- W3C Verifiable Credentials Data Model v2: https://www.w3.org/TR/vc-data-model-2.0/
- did:key Method: https://w3c-ccg.github.io/did-method-key/
- JSON Web Tokens: https://jwt.io/
