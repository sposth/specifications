# Roles & actors

> **Status: DRAFT** – proposal for the `specifications` repo, pending developer approval. Reflects code as of 2026-07-10.

This document reconciles the actor model used across Creator Credentials. The
older specs describe **three** roles – Host, Issuer, Creator – but the code has
only **two** user roles. This is not a contradiction once you separate the
*platform* from *user accounts*: the "Host" is the deployed platform, not a
login. See `01-architecture-overview.md` for the service topology and
`06-signing-and-trust-model.md` for how the platform signs.

---

## 1. The three actors, reconciled

| Actor | Is it a user account? | Identity | What it is |
|---|---|---|---|
| **Host** | **No** | `did:web:liccium.com` | The **platform** (Liccium) itself. Self-signs platform-level VCs. Not a `ClerkRole`, not a row you can sign in as. |
| **Issuer** | Yes | Clerk user, `ClerkRole.Issuer` | An organization that issues membership / data-supplier VCs to creators. |
| **Creator** | Yes | Clerk user, `ClerkRole.Creator` | An individual/holder who self-verifies and requests VCs from issuers. |

The code's role enum is only two-valued:

```
ClerkRole { Issuer = 'issuer', Creator = 'creator' }   // user.entity.ts:19-22
```

**"Host" ≠ a user role.** The platform acts as the Host by self-signing its own
VCs under `did:web:liccium.com` (`credentialsHost = 'liccium.com'`,
`credentials.helpers.ts:13`). There is no Host account, no Host login, and no
`ClerkRole.Host`. Wherever the older Host-role design (retired) implies a Host
*user*, read it as **the platform**. (Note the asymmetry from
`01-architecture-overview.md`: the backend signs *as* `did:web:liccium.com` but
never serves its own `.well-known/did.json` – it only fetches users' did.json
documents.)

So there are exactly **two real user roles: Issuer and Creator.**

---

## 2. How a role is assigned

There is **no post-signup "pick your role" screen.** Role is chosen *implicitly*
by which signup/login URL the user takes from the `/welcome` role-picker, then
persisted and resolved in three steps:

1. **URL choice (UI).** `/welcome` presents Creator vs Issuer cards linking to
   `/auth/signup/creator` or `/auth/signup/issuer`. The chosen path is the
   single source of role truth.

2. **Written to Clerk metadata (UI).** On first hit of a protected page, the SSR
   gate `withAuth` (`src/components/modules/app/withAuth.ts`) infers the role
   from the URL (`.includes('issuer' / 'creator')`) if Clerk
   `publicMetadata.role` isn't already set, and writes it back to
   `publicMetadata.role` (`withAuth.ts:37-55`). It also promotes the
   `pendingTosLink` cookie into `publicMetadata.termsAreAccepted` + `termsLink`
   so the backend can enforce the terms gate.

3. **Resolved by the webhook (backend).** The svix-verified Clerk webhook reads
   `public_metadata.role ?? unsafe_metadata.role` and maps it:

   ```
   resolveRole(roleMetadata) {
     if (roleMetadata === 'ISSUER') return ClerkRole.Issuer;   // literal only
     return ClerkRole.Creator;                                  // everything else
   }                                                            // webhooks.service.ts
   ```

   Only the **literal string `'ISSUER'`** maps to Issuer; **any other value (or
   none) falls through to Creator.** Creator is the effective default, and the
   `user.clerk_role` column also defaults to `creator`.

The DB `user` row is only created once Clerk metadata carries `termsAreAccepted`
+ `termsLink`; until then the webhook defers creation to a later `user.updated`
event (see the terms-acceptance gate in §4).

### No backend role guard

There is **no role guard or `@Roles()` decorator** in the backend. `AuthGuard`
only confirms a DB user exists for the Clerk session and attaches it as
`req.user`. Each handler then **re-checks `user.clerkRole` manually** and throws
`NotFoundException('This api is only for ...')` on a mismatch. Role enforcement
is therefore per-handler and easy to miss – flag it as a hardening item (a
shared role guard would centralize it).

---

## 3. What each actor can do

Capability lists are summaries; the endpoint-level detail is in
`08-api-reference.md` and the VC-level detail in
`03-verifiable-credentials-catalog.md`.

### Host (the platform)

- Generates a per-user self-signed X.509 cert + did:key on provisioning.
- Self-signs platform VCs as `did:web:liccium.com` (the default "platform"
  signing source; issuers can switch to an imported external eIDAS cert – see
  `06-signing-and-trust-model.md`).
- Runs the automated verification pollers (DNS TXT, did:web) and the eIDAS trust
  store. It is machinery, not a user surface.

### Creator

- Self-verify: email (auto-issued from the Clerk-verified primary email),
  domain (DNS TXT), external EC keypair, and bind a Liccium did:key via the
  cross-app bridge.
- Browse issuers (`/creator/issuers`) and request a connection to one.
- Request VCs from a connected issuer (Member / Data Supplier / Liccium Data
  Supplier) and hold the resulting credentials (`/creator/credentials`).

### Issuer

- Self-verify the org: email, domain (DNS TXT), did:web
  (`.well-known/did.json`), and import an external eIDAS QSeal/QSig cert
  (cert-challenge).
- Define which VC types it offers (`credentialsToIssue`).
- Manage creator connections: accept / reject / **revoke** connection requests
  (`/issuer/creators/...`).
- Issue VCs to creators via the request → accept → **cert-signed
  verify-signature** state machine (`/issuer/credentials/...`).

> **Revocation caveat.** An issuer "revoke" flips the connection status to
> `REVOKED` (or deletes a credential row); it does **not** invalidate an
> already-issued VC. A W3C Bitstring Status List revocation model is specified
> in [`future/revocation-bitstring-status-list.md`](../future/revocation-bitstring-status-list.md) but is **Planned / not
> implemented**. See `03-verifiable-credentials-catalog.md` and
> `06-signing-and-trust-model.md`.

---

## 4. The terms-acceptance gate

Terms acceptance is a hard precondition for existing as a user:

- The UI captures acceptance during signup (a ToS checkbox sets the
  `pendingTosLink` cookie: `config.CREATOR_TERMS_URL` or `config.ISSUER_TERMS_URL`),
  and `withAuth` promotes it into Clerk `publicMetadata.termsAreAccepted` +
  `termsLink`.
- The backend webhook **will not create a `user` row** until the Clerk metadata
  carries both `termsAreAccepted` and `termsLink`; otherwise it defers to a
  later `user.updated` (`webhooks.service.ts`). The accepted terms link is
  stored on `user.terms_link`.

So a Clerk account without accepted terms has no backend user row and cannot use
any guarded `/v1` route.

---

## 5. No Verifier surface in-app

The older docs describe a third conceptual party, the **Verifier** (who checks a
presented VC). **There is no verifier surface in this application** – no public
verify/resolve page, no presentation endpoint. Verification of an issued VC
happens **out-of-app** (a relying party inspects the VC's signature / `x5c`
chain themselves). Treat any in-app Verifier flow as **Planned / not
implemented**; the current product covers only the Issuer and Creator surfaces
plus the platform (Host) machinery.
