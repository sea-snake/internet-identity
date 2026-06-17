# Implementation plan — `/mcp` delegation authorization page

Lets a **single, deploy-configured MCP server** obtain an Internet Identity
delegation that acts as the user **at an application** (a derivation origin),
through the same account-delegation machinery that powers `/cli`. App-mode only.

## 1. Goal and how it differs from `/cli`

`/cli` already does the core of this in *app mode*: it sets
`effectiveOrigin = remapToLegacyDomain(domain)`, calls
`prepare_account_delegation` / `get_account_delegation`, sub-delegates to the
caller's session key, and form-POSTs the chain to the caller. `/mcp` is a close
sibling with these deliberate differences:

| Aspect | `/cli` | `/mcp` |
| --- | --- | --- |
| Relying party | Local CLI binary | **One deploy-configured MCP server** (HTTPS origin) |
| Delivery target | Top-level form POST to `http://127.0.0.1:*` loopback | Top-level form POST to the **configured MCP server** HTTPS callback |
| Target application | Optional (`domain` absent ⇒ generic II sign-in) | **Required, always present** — there is no generic mode |
| Consent | Device gate (one-time) + Continue button | Device gate + **per-request consent naming the application** |

Confirmed design decisions:

- **Relying party:** a single MCP server **configured as a frontend-canister
  deploy arg** (not arbitrary per request). Its origin is operator-trusted.
- **Transport:** a plain **redirect flow — GET in, top-level form POST out**,
  exactly like `/cli`. The MCP server redirects the browser to `/mcp` with the
  request params (a GET); the delegation is returned by a top-level form POST to
  the configured origin, which redirects back with `status`. Viable because the
  configured origin can be added to the CSP `form-action` directive statically.
  (No `fetch`, no CORS — a top-level form POST is a navigation, not a
  cross-origin subrequest.)
- **No postMessage / ICRC channel transport.** Because both legs are ordinary
  HTTP navigations, `/mcp` does **not** use the postMessage signer path
  (`channelStore`, `PostMessageTransport`, the `icrc34_delegation` JSON-RPC
  handlers) and does **not** run `validateDerivationOrigin`. It reuses the
  canister account-delegation methods directly, like `/cli`. What carries over
  from "ICRC-34/95" is only the *concept*: an ICRC-34-style delegation derived
  for an application's ICRC-95 derivation origin (`app` → `remapToLegacyDomain`
  → `prepare_account_delegation`).
- **Authorization model:** device gate + per-request consent, reusing the `/cli`
  model — **no `.well-known/ii-alternative-origins` validation**.
- **App-mode only:** `app` (derivation origin) is always required; no generic
  II sign-in path, no `CLI_GENERIC_DERIVATION_ORIGIN` equivalent.
- **Inbound params in the URL fragment** (`/mcp#…`), matching `/cli` — keeps the
  session key and callback out of II's server-side request logs.
- **Account:** default account only (`account_number = []`), matching `/cli`,
  which has no account selection.
- **Scope:** the II-side `/mcp` page + the deploy arg + CSP wiring. The MCP
  server itself is a separate project.

## 2. Configurable MCP server (deploy arg)

Follows the existing `related_origins` / `feature_flags` path end-to-end:

1. **`InternetIdentityFrontendArgs`** (`src/internet_identity_interface/.../types.rs`)
   and `InternetIdentityFrontendInit` (`internet_identity_frontend.did`): add
   `mcp_server_origin: Option<String>` (e.g. `"https://mcp.id.ai"`, no trailing
   slash). Regenerate the FE bindings
   (`$lib/generated/internet_identity_frontend_{types,idl}`).
2. **CSP** (`internet_identity_frontend/src/main.rs#get_content_security_policy`):
   when configured, append the origin to `form-action`, so the top-level form
   POST to the MCP server is permitted — analogous to how `related_origins`
   feeds `frame-ancestors` / `frame-src`. `form-action` stays
   `'self' http://127.0.0.1:*` plus the configured MCP origin; never `https:`.
3. **Frontend** reads `frontendCanisterConfig.mcp_server_origin` (decoded from
   `document.body.dataset.canisterConfig` in `globals.ts#initGlobals`) to:
   - validate the request `callback`'s origin equals the configured origin
     (reject ⇒ invalid screen — belt-and-suspenders with the CSP), and
   - show the MCP server identity on the consent screen.
   If `mcp_server_origin` is unset, `/mcp` is effectively disabled (every
   request is invalid).

*(Exact field name / whether to store full origin vs URL is part of the design
the operator is finalising; the above is the integration shape.)*

## 3. End-to-end flow

1. The MCP server generates an ephemeral session key pair and redirects the
   user's browser to
   `https://id.ai/mcp#public_key=…&callback=…&app=…&state=…&ttl=…`. Parameters
   go in the **URL fragment** (as `/cli` does) so the session key, callback, and
   target app stay out of II's server-side request logs.
2. `/mcp/+page.ts` parses/validates the fragment; `+page.svelte` drives the
   phase machine.
3. The user signs in (`AuthWizard` / `AuthLastUsedFlow`) and is shown a
   **consent screen naming the target application** (and the configured MCP
   server). The chosen identity must have **MCP access enabled on this device**
   (gate); otherwise the access-disabled screen is shown.
4. On approval, `mcp/utils.ts#mcpAuthorize`:
   - generates a fresh **non-extractable** browser ephemeral key,
   - `prepare_account_delegation(identityNumber, effectiveOrigin, [], ephemeralPubKey, [ttlNanos])`,
   - `get_account_delegation(…)` → canister-signed `DelegationChain`,
   - sub-delegates ephemeral key → the **MCP server's session public key** from
     the fragment (chain expires as a whole),
   - **top-level form POST** of `{ delegation: chain.toJSON(), state }` to the
     configured-origin `callback`.
5. The MCP server validates `state`, stores the chain against the private key it
   holds, then **redirects the browser back to `/mcp#status=success`** (or
   `status=error`).
6. `/mcp` reads `status` on reload and shows the **success / error screen**
   (reusing the `/cli` redirect-back-with-`status` machinery).

The canister **never signs to the session public key from the fragment** (it is
attacker-controllable); it signs to the browser ephemeral key and the page
sub-delegates — same invariant `/cli` relies on.

## 4. Security model

Because the relying party is **one operator-configured MCP server**, this is
much narrower than "any remote server": II only ever delivers to the configured
origin (enforced by both the `callback` origin check and the `form-action` CSP).

- **Bound to the MCP server key.** Canister signs to the browser ephemeral key;
  the chain ends at the MCP server's session key, so interception in transit is
  insufficient to use the delegation. `state` binds CSRF.
- **No alternative-origins check** (per decision): II derives the principal for
  whatever `app` the request names. Trust rests on the device gate + per-request
  consent — same as `/cli` app mode, and now further bounded by the fixed
  configured delivery origin.
- **Mitigations:**
  - **Device gate** (`mcpAccessStore`): per-identity, per-device opt-in with an
    explicit warning dialog, separate from CLI access.
  - **Per-request consent** screen prominently naming the **target application**
    (and the configured MCP server) plus the identity and TTL. Approved every
    time.
  - **Configured HTTPS callback origin** (http allowed only under dev CSP).
  - **Short default TTL** — recommend well below `/cli`'s 480 min for these
    delegations (e.g. 60 min), capped by `MAX_EXPIRATION_PERIOD_NS`.
- **For security review:** residual risk is a user with the gate enabled being
  socially engineered into approving a delegation for a sensitive app — the
  consent UI is the last line of defence.

## 5. Frontend implementation

New directory `src/frontend/src/routes/(new-styling)/mcp/`, mirroring `cli/`:

- **`+page.ts`** — parse/validate the fragment into `McpParams` + `status`:
  - `public_key` (base64url DER, required),
  - `callback` (required; HTTPS; **origin must equal the configured MCP
    origin**; allow `http://127.0.0.1` only under dev),
  - `app` (required bare hostname; reuse `/cli`'s `parseDomain`),
  - `state` (required, echoed back — the `/mcp` analogue of `/cli`'s `nonce`),
  - `ttl` (optional; reuse `parseTtl`, lower default),
  - `status` (`success | error`) for the redirect-back load (reuse `/cli`'s
    `parseStatus`, minus `identity-mismatch`).
- **`+page.svelte`** — phase machine `wizard | authorize | consent | success |
  error | access-disabled | invalid`, plus the onMount `status`/fragment-clear
  logic copied from `/cli`. The `consent` phase is new vs `/cli` (which folds
  consent into Continue) because consent must name the app every request.
- **`+layout.svelte`** — identity switcher header; factor the shared header out
  of `cli/+layout.svelte` into a common component rather than copy-paste.
- **`utils.ts`** — `mcpAuthorize()` (form POST to the configured callback) on
  top of the shared chain builder (§6).
- **`mcp-access.store.ts`** — device-local gate, structurally identical to
  `cli-access.store.ts`; new `storeLocalStorageKey.McpAccess`.
- **`views/`** — `McpConsentView`, `McpSuccessView`, `McpErrorView`,
  `McpAccessDisabledView` (+ authorize view if not folded into consent).

## 6. Shared refactor

`cli/utils.ts#cliAuthorize` and `mcp/utils.ts#mcpAuthorize` share the
prepare → get → sub-delegate sequence. Extract
`$lib/utils/delegation/buildAccountDelegationChain({ authenticated,
targetPublicKey, effectiveOrigin, ttlNanos })`, used by both; the differences
(loopback vs configured-origin form target) stay in each route's `utils.ts`.
Keeps the "never sign to the supplied key" invariant in one place.

## 7. Settings (device gate)

- `McpAccessSection.svelte` + `McpConfirmDialog.svelte` under
  `manage/(authenticated)/settings/components/`, mirroring the CLI pair with
  MCP-specific warning copy. Wire next to `CliAccessSection`. Keep MCP access a
  **separate** toggle from CLI access.

## 8. Backend changes

- **Deploy arg + CSP** as in §2 (the only required backend changes).
- **Candid / canister methods:** none — reuse `prepare_account_delegation` /
  `get_account_delegation`.
- **No CORS, no `.well-known`** discovery needed (top-level navigation; the MCP
  server is configured with the `/mcp` URL out of band).

## 9. Analytics

`mcpAuthorizeFunnel.ts` mirroring `cliAuthorizeFunnel.ts`, constructed
`new Funnel("mcp-authorize", true)` (**`prefixEvents = true`**, short unprefixed
values). Events: `request-invalid`, `request-received`, `confirmed`,
`access-disabled`, `success`, `error`.

## 10. Testing

- **E2E (Playwright):** `tests/e2e-playwright/fixtures/mcp.ts` modelled on
  `fixtures/cli.ts`; the mock MCP server receives the form POST, asserts `state`
  and a valid delegation chain, and redirects back to `/mcp#status=success`.
  Spec: valid request → consent → delivered → success; access-disabled when gate
  off; invalid fragment; callback origin ≠ configured origin rejected; `status`
  redirect-back screens.
- **Unit:** `+page.ts` validation (callback origin must equal configured origin,
  required `app`/`state`, TTL default/cap); shared chain builder;
  `mcp-access.store`.
- **Rust:** `get_content_security_policy` assertion — configured MCP origin
  appears in `form-action`, and `form-action` is never broadened to `https:`.

## 11. i18n and docs

- New strings via `$t` / `<Trans>`; **do not** hand-edit `lib/locales/*.po`
  (bot-managed).
- Document `/mcp`, the deploy arg, and the security model alongside `/cli`,
  including the explicit "no alternative-origins validation" note.

## 12. Open questions

1. **Deploy-arg shape:** field name and whether to store the bare origin vs a
   full callback URL (the operator's design, point 1).
2. **Default TTL** for these delegations (proposed 60 min).
3. **Consent copy** naming the configured MCP server (security-sensitive).

## 13. Task breakdown

1. Add `mcp_server_origin` to `InternetIdentityFrontendArgs` + `.did`;
   regenerate FE bindings; feed it into `form-action` in
   `get_content_security_policy`; expose via `globals.ts`.
2. Extract shared `buildAccountDelegationChain`; refactor `cli/utils.ts` onto it.
3. `mcp-access.store.ts` + `storeLocalStorageKey.McpAccess`.
4. `mcp/+page.ts` parsing/validation (incl. configured-origin callback check) +
   unit tests.
5. `mcp/utils.ts#mcpAuthorize` (form POST) + unit test.
6. `mcp/+page.svelte` phase machine + views + `status` redirect-back handling.
7. Shared layout header refactor + `mcp/+layout.svelte`.
8. `McpAccessSection` / `McpConfirmDialog` + settings wiring.
9. `mcpAuthorizeFunnel.ts`.
10. E2E fixture + spec; Rust CSP assertion.
11. Docs/spec update; run formatter + linter before commit.
