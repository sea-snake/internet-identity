# Implementation plan — `/mcp` delegation authorization page

Lets a **single, deploy-configured MCP server** obtain an Internet Identity
delegation that acts as the user **at an application** (a derivation origin),
through the same account-delegation machinery that powers `/cli`. App-mode only.

UI follows the Claude Design handoff *"II CLI authorize"* (the `authorize-mcp`
screens and the MCP settings card). Source of truth for copy/layout:
`mcp-screens.jsx`, `mcp-app.jsx`, `manage.jsx`, `authorize-mcp.css` in that
bundle.

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
| Target application | Optional (`domain` absent ⇒ generic II sign-in) | **Required, always present** — no generic mode |
| Account | `account_number = []` (anchor seed) | The user's **default account** for the app, fetched via `get_default_account` |
| Consent | Device gate (one-time) + Continue button | Device gate + **per-request "Allow MCP access" screen naming the app + MCP server** |

Confirmed design decisions:

- **Relying party:** a single MCP server **configured as a frontend-canister
  deploy arg** (`mcp_server_origin`, e.g. `https://mcp.id.ai`). Operator-trusted.
- **Transport:** a plain **redirect flow — GET in, top-level form POST out**,
  like `/cli`. The MCP server redirects the browser to `/mcp` with the request
  params (GET); the delegation is returned by a top-level form POST to the
  configured origin, which redirects back with `status`.
- **No postMessage / ICRC channel transport.** Both legs are ordinary HTTP
  navigations, so `/mcp` does **not** use `channelStore` /
  `PostMessageTransport` / the `icrc34_delegation` JSON-RPC handlers and does
  **not** run `validateDerivationOrigin`. It reuses the canister
  account-delegation methods directly. What carries over from "ICRC-34/95" is
  only the *concept*: an ICRC-34-style delegation derived for an application's
  ICRC-95 derivation origin.
- **App-mode only:** `app` (derivation origin) is always required; no generic
  path, no `CLI_GENERIC_DERIVATION_ORIGIN` equivalent. (Generic mode was
  explicitly removed from the design.)
- **Inbound params in the URL fragment** (`/mcp#…`), matching `/cli` — keeps the
  session key and callback out of II's server-side request logs.
- **Account:** the app's **default account**, not a hardcoded `[]`. Fetch it
  with `get_default_account(anchor, origin)` and pass the returned
  `account_number` (which is null for the unreserved default, or a real number
  if the user set one) into `prepare_account_delegation` /
  `get_account_delegation`. This makes MCP act as the *same* account a normal
  `/authorize` sign-in would, rather than the legacy anchor-seed principal that
  a blind `[]` produces. **No multi-account picker** on the screen.
- **No client name.** MCP dynamic client registration means the calling client
  (e.g. an AI assistant) is not trustworthy to name; the UI frames everything
  around the known **MCP server**, shown by hostname only (`mcp.id.ai` /
  "MCP server").
- **TTL:** FE default **60 min**; the caller may request longer via the `ttl`
  fragment param, clamped to 30 days by the backend (`MAX_EXPIRATION_PERIOD_NS`).
- **Scope:** the II-side `/mcp` page + the deploy arg + CSP wiring + the MCP
  settings card. The MCP server itself is a separate project.

## 2. Configurable MCP server (deploy arg)

Follows the existing `related_origins` / `feature_flags` path end-to-end:

1. **`InternetIdentityFrontendArgs`** (`src/internet_identity_interface/.../types.rs`)
   and `InternetIdentityFrontendInit` (`internet_identity_frontend.did`):
   ```rust
   /// Origin of the trusted MCP server, e.g. "https://mcp.id.ai" (no trailing
   /// slash). When unset, the /mcp delegation flow is disabled.
   pub mcp_server_origin: Option<String>,
   ```
   Mirrors the existing `backend_origin` convention. Regenerate the FE bindings
   (`$lib/generated/internet_identity_frontend_{types,idl}`).
2. **CSP** (`internet_identity_frontend/src/main.rs#get_content_security_policy`):
   when configured, append the origin to `form-action`, so the top-level form
   POST to the MCP server is permitted — analogous to how `related_origins`
   feeds `frame-ancestors` / `frame-src`. `form-action` stays
   `'self' http://127.0.0.1:*` plus the configured MCP origin; never `https:`.
3. **Frontend** reads `frontendCanisterConfig.mcp_server_origin` (decoded from
   `document.body.dataset.canisterConfig` in `globals.ts#initGlobals`) to:
   - validate `new URL(callback).origin === new URL(mcp_server_origin).origin`
     (reject ⇒ invalid screen — belt-and-suspenders with the CSP), and
   - render the MCP server hostname on the authorize screen.
   If `mcp_server_origin` is unset, `/mcp` is effectively disabled (every
   request is invalid).

The config stores the **origin**, not a full callback URL: it matches the
granularity `form-action` can enforce (`scheme://host:port`, no path), lets the
MCP server choose/version its own callback path without an II redeploy, and
keeps `/mcp` structurally parallel to `/cli`. Generalises to
`Option<Vec<String>>` later if multiple trusted servers are ever needed.
`https` required (the configured origin and the callback); `http://127.0.0.1`
accepted only under `dev_csp`.

## 3. End-to-end flow

1. The MCP server generates an ephemeral session key pair and redirects the
   browser to `https://id.ai/mcp#public_key=…&callback=…&app=…&state=…&ttl=…`
   (params in the fragment).
2. `/mcp/+page.ts` parses/validates the fragment; `+page.svelte` drives the
   phase machine.
3. The user signs in (`AuthWizard` / `AuthLastUsedFlow`) and lands on the
   **"Allow MCP access"** screen (§5). The chosen identity must have **MCP
   access enabled on this device** (gate); otherwise the access-disabled screen
   shows.
4. On "Allow access", `mcp/utils.ts#mcpAuthorize`:
   - `get_default_account(identityNumber, effectiveOrigin)` → `accountNumber`
     (`opt AccountNumber`),
   - generates a fresh **non-extractable** browser ephemeral key,
   - `prepare_account_delegation(identityNumber, effectiveOrigin, accountNumber, ephemeralPubKey, [ttlNanos])`,
   - `get_account_delegation(…)` → canister-signed `DelegationChain`,
   - sub-delegates ephemeral key → the **MCP server's session public key** from
     the fragment (chain expires as a whole),
   - **top-level form POST** of `{ delegation: chain.toJSON(), state }` to the
     configured-origin `callback`.
5. The MCP server validates `state`, stores the chain against the private key it
   holds, then **redirects the browser back to `/mcp#status=success`** (or
   `status=error`).
6. `/mcp` reads `status` on reload and shows the **success / error** screen
   (reusing the `/cli` redirect-back-with-`status` machinery).

The canister **never signs to the session public key from the fragment** (it is
attacker-controllable); it signs to the browser ephemeral key and the page
sub-delegates — the same invariant `/cli` relies on.

## 4. Security model

The relying party is **one operator-configured MCP server**, so II only ever
delivers to the configured origin (enforced by both the `callback` origin check
and the `form-action` CSP).

- **Bound to the MCP server key.** Canister signs to the browser ephemeral key;
  the chain ends at the MCP server's session key, so interception in transit is
  insufficient to use the delegation. `state` binds CSRF.
- **No alternative-origins check** (per decision): II derives the principal for
  whatever `app` the request names. Trust rests on the device gate + the
  per-request "Allow MCP access" screen — same as `/cli` app mode, further
  bounded by the fixed configured delivery origin.
- **Mitigations:** per-device MCP-access gate (with an explicit risk-ack
  dialog), per-request authorize screen naming the app + MCP server, configured
  HTTPS callback origin, short default TTL (60 min).
- **For security review:** residual risk is a user with the gate enabled being
  socially engineered into approving access for a sensitive app — the authorize
  screen and the settings risk-ack are the last lines of defence.

## 5. Frontend — `/mcp` route

New directory `src/frontend/src/routes/(new-styling)/mcp/`, mirroring `cli/`.
Phases: `wizard | authorize | close | mcp-disabled | invalid | error`. The
identity-switcher header shows only during `wizard`/`authorize`.

- **`+page.ts`** — parse/validate the fragment into `McpParams` + `status`:
  - `public_key` (base64url DER, required),
  - `callback` (required; HTTPS; **origin must equal the configured MCP
    origin**; `http://127.0.0.1` only under dev),
  - `app` (required bare hostname; reuse `/cli`'s `parseDomain`),
  - `state` (required, echoed back — the `/mcp` analogue of `/cli`'s `nonce`),
  - `ttl` (optional; reuse `parseTtl` with a **60-min** default; backend clamps
    to 30 days),
  - `status` (`success | error`) for the redirect-back load.
- **`+page.svelte`** — phase machine + the onMount `status`/fragment-clear logic
  adapted from `/cli`.
- **`+layout.svelte`** — `/mcp`'s own identity-switcher header, following the
  pattern in `cli/+layout.svelte`. `/cli` is left untouched.
- **`utils.ts`** — self-contained `mcpAuthorize()` (default-account fetch →
  build chain → form POST). No shared helper with `/cli`.
- **`mcp-access.store.ts`** — device-local gate, structurally like
  `cli-access.store.ts`; new `storeLocalStorageKey.McpAccess`.
- **`views/`** + a small **`McpHero`** component.

### Screens (copy + layout from the design)

- **`McpHero`** (centerpiece of wizard + authorize): three nodes —
  app tile (dapp logo, else a globe) · a **lock chip** · the **MCP plug glyph**
  tile — with badges underneath (`oc.app` and `mcp.id.ai`). Conveys
  "your identity links this app to the MCP server" before any copy.
- **Wizard** (new user): `McpHero` + **"Choose method"** + subtitle
  **"to allow MCP access to {app}"**, then the existing `AuthWizard`
  (passkey + OpenID providers + SSO) and a "Lost access … Recover" row.
- **Authorize** (`McpAuthorizeView`, returning/after sign-in — the single
  combined authorize+allow surface): `McpHero` + title **"Allow MCP access"** +
  subtitle **"Let {mcp.id.ai} act as your {app} account"** + a full-width
  **"Allow access"** primary button (busy state: "Allowing access…"). **No
  multi-account picker** — default account only (§1). This is the per-request
  consent; shown every connection.
- **Close / success** (`McpCloseWindowView`): success check, **"You're signed
  in"** / **"You can close this window."** *(Screenshot shows a later variant
  "You're connected / You can return to your MCP client" — final wording is
  adjustable; keep it client-agnostic since the client is unknown.)*
- **mcp-disabled** (gate not enabled): lock icon, **"MCP access not enabled"** /
  "For security, Internet Identity blocks MCP clients from signing in to apps
  using your identity." / "Enable MCP access for this device, then try again." +
  a **"Manage your identity"** button → `/manage`.
- **Invalid**: **"Invalid request"** / "This connection link is missing
  information or has been changed." / "Start the connection again from your MCP
  client. You can close this window."
- **Error**: **"Something went wrong"** / "The MCP client couldn't finish
  connecting." / "Return to your MCP client and try again. You can close this
  window."

## 6. Settings — MCP access card

One settings page with **two cards: the existing CLI access card (unchanged)
and a new MCP access card**, each with its **own** dialog implementation. `/cli`
components (`CliAccessSection`, `CliConfirmDialog`) are **not** touched or
generalised.

- **`McpAccessSection.svelte`** — card with the MCP plug icon, title **"MCP
  access"**, subtitle **"Let AI assistants on this device sign in to apps using
  your identity."** (and when enabled: "AI assistants on this device can ask to
  sign you in to apps."), a `Badge` "Enabled" when on, and a `Toggle`. Toggling
  **on** opens the confirm dialog; toggling **off** is unguarded.
- **`McpConfirmDialog.svelte`** — warning icon, title **"Are you sure?"**, body
  **"Enabling this lets AI assistants on this device ask to sign you in to apps
  using your identity."**, two risk lines:
  - "It can send messages or move funds on your behalf."
  - "Like any AI, it can hallucinate and make mistakes."
  a checkbox **"I understand the risks."** gating the **"Enable MCP access"**
  button; Esc / backdrop close. (No install-command/verify row — that is
  CLI-specific.)
- **`mcp-access.store.ts`** backs the toggle (per-identity, per-device, in
  localStorage), independent of CLI access.

## 7. Backend changes

- **Deploy arg + CSP** as in §2 (the only required backend changes).
- **Candid / canister methods:** none new — reuse `get_default_account`,
  `prepare_account_delegation`, `get_account_delegation`.
- **No CORS, no `.well-known`** discovery needed (top-level navigation; the MCP
  server is configured with the `/mcp` URL out of band).

## 8. Analytics

`mcpAuthorizeFunnel.ts` mirroring `cliAuthorizeFunnel.ts`, constructed
`new Funnel("mcp-authorize", true)` (**`prefixEvents = true`**, short unprefixed
values). Events: `request-invalid`, `request-received`, `confirmed`,
`access-disabled`, `success`, `error`.

## 9. Testing

- **E2E (Playwright):** `tests/e2e-playwright/fixtures/mcp.ts` modelled on
  `fixtures/cli.ts`; a mock MCP server receives the form POST, asserts `state`
  and a valid delegation chain, and redirects back to `/mcp#status=success`.
  Spec: valid request → "Allow access" → delivered → success; access-disabled
  when the gate is off; invalid fragment; callback origin ≠ configured origin
  rejected; `status` redirect-back screens.
- **Unit:** `+page.ts` validation (callback origin must equal configured origin,
  required `app`/`state`, TTL default/cap); `mcp/utils.ts` (default-account
  fetch is passed through, not hardcoded `[]`); `mcp-access.store`.
- **Rust:** `get_content_security_policy` assertion — configured MCP origin
  appears in `form-action`, and `form-action` is never broadened to `https:`.

## 10. i18n and docs

- New strings via `$t` / `<Trans>`; **do not** hand-edit `lib/locales/*.po`
  (bot-managed).
- Document `/mcp`, the deploy arg, and the security model alongside `/cli`,
  including the explicit "no alternative-origins validation" note.

## 11. Open questions

1. **Success-screen wording** — "You're signed in" (final JSX) vs the
   screenshot's "You're connected"; keep client-agnostic.

## 12. Task breakdown

1. Add `mcp_server_origin` to `InternetIdentityFrontendArgs` + `.did`;
   regenerate FE bindings; feed it into `form-action` in
   `get_content_security_policy`; expose via `globals.ts`.
2. `mcp-access.store.ts` + `storeLocalStorageKey.McpAccess`.
3. `mcp/+page.ts` parsing/validation (incl. configured-origin callback check) +
   unit tests.
4. `mcp/utils.ts#mcpAuthorize` (default-account fetch + build chain + form POST)
   + unit test.
5. `McpHero` component.
6. `mcp/+page.svelte` phase machine + views (wizard via `AuthWizard`, authorize,
   close, mcp-disabled, invalid, error) + `status` redirect-back handling.
7. `mcp/+layout.svelte` (own identity-switcher header).
8. `McpAccessSection` + `McpConfirmDialog` (own dialog) added to the settings
   page next to the untouched CLI card.
9. `mcpAuthorizeFunnel.ts`.
10. E2E fixture + spec; Rust CSP assertion.
11. Docs/spec update; run formatter + linter before commit.
