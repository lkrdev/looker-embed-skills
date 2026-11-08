---
name: cors-api
description: Architectural pattern, TypeScript SDK integration, and implementation standards for direct browser-to-Looker API calls via CORS using OAuth 2.0 with PKCE.
---

# Looker CORS API & OAuth 2.0 PKCE Pattern

This skill provides architectural standards, TypeScript SDK integration patterns, and best practices for building client-side data applications that communicate directly with the Looker API via **CORS** and **OAuth 2.0 with PKCE (Proof Key for Code Exchange)**.

---

## 1. Architectural Overview

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Browser Application (React / Vue / TypeScript)           │
│    • Handles OAuth 2.0 PKCE challenge generation            │
│    • Stores access tokens in sessionStorage via SDK session │
│    • Executes queries directly via @looker/sdk              │
└──────────────────────────────┬──────────────────────────────┘
                               │ CORS Bearer Token Requests
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Looker Instance (API 4.0 & Auth Server)                  │
│    • /auth -> User login & consent dialog                   │
│    • /api/token -> PKCE verification & token issuance       │
│    • /api/4.0/queries/run/json -> Governed query execution  │
│    • Enforces User Permissions, RLS, & LookML Caching       │
└─────────────────────────────────────────────────────────────┘
```

### Key Advantages:
* **Zero Backend**: Eliminates custom API proxies or backend token vaults.
* **Public Client Security**: PKCE removes the need for client secrets in the browser.
* **Governed Data Access**: Inherits the logged-in user's exact permissions, access filters, and user attributes.

---

## 2. Prerequisites & Configuration Checklist

Before initiating OAuth flows, verify these 4 instance settings:

| Setting | Location in Looker | Requirement |
| :--- | :--- | :--- |
| **Looker API CORS** | `Admin > Labs` | Enabled |
| **OAuth Client App** | `Admin > Platform > API Explorer` or Admin UI | Register `client_guid`, exact `redirect_uri`, `enabled: true` |
| **Embedded Domain Allowlist** | `Admin > Embed` | Add application origin (e.g. `https://localhost:3000` or production domain) |
| **HTTPS Protocol** | Application Server | Required for Web Crypto API (`window.crypto.subtle`) and PKCE validation |

---

## 3. Core Implementation Rules for Agents

When implementing or modifying Looker CORS OAuth applications, you **MUST** follow these directives:

### 1. Token Persistence Across Navigations (`PersistentOAuthSession`)
`@looker/sdk-rtl`'s default `OAuthSession` keeps tokens strictly in JavaScript memory. You **MUST** subclass `OAuthSession` to persist `access_token`, `refresh_token`, and `expiresAt` in `sessionStorage`. Otherwise, page refreshes or route changes will lose the active session.

### 2. Subpath Imports for `@looker/sdk`
Due to upstream TypeScript interface re-export issues in the root package, **DO NOT** import directly from `@looker/sdk` when using bundlers like Vite or esbuild. Always import from:
```typescript
import { Looker40SDK } from '@looker/sdk/lib/4.0/methods';
import type { IUser, IWriteQuery } from '@looker/sdk/lib/4.0/models';
import { BrowserServices, OAuthSession, AuthToken } from '@looker/sdk-rtl';
```

### 3. Client-Side SPA Navigation in Callback Handlers
After redeeming the authorization code in the `/callback` handler, transition to the target app view using **client-side state navigation** (`window.history.replaceState` or React Router state). **DO NOT** perform a hard browser reload (`window.location.replace`), which resets in-memory React state.

### 4. Declarative Semantic Query Binding
Use a custom React hook (such as `useLookerQuery`) that dispatches `sdk.run_inline_query({ result_format: 'json', body: queryPayload })` to bind Looker dimensions and measures dynamically to UI charts and tables.

### 5. Strict `redirect_uri` Validation (No Wildcards)
Looker OAuth client applications (`OauthClientApp`) enforce strict character-for-character equality (`==`) on `redirect_uri` during both `/auth` and `/api/token`. Wildcards (`*`), regexes, trailing slash discrepancies, and dynamic subpaths are rejected with `400 Bad Request` (only RFC 8252 loopback IP literals `127.0.0.1` / `[::1]` allow dynamic port matching). Register an array of static callbacks in `redirect_uris: string[]` or pass dynamic deep-link destinations via the OAuth `state` parameter / `sessionStorage`.

### 6. Two-Layer OAuth Boundary (`cors_api` vs BigQuery `OAuthUserState`)
Calling `create_embed_url_as_me` (`POST /api/4.0/embed/token_url/me`) with a PKCE token only establishes the Looker `:embed_user` session; it does **not** populate database OAuth tokens in `Looker::Model::OAuthUserState`. If the embedded dashboard queries a BigQuery OAuth connection, in-iframe redirects to `accounts.google.com` will be blocked by Google's `CSP: frame-ancestors` header (`403`). Seed the user's Google OAuth tokens into Looker via `create_oauth_application_user_state` (`POST /api/4.0/external_oauth_applications/user_state`) before mounting the iframe (see [troubleshooting.md](references/troubleshooting.md)).

---

## 4. Reference Guides & Code Templates

For complete code implementations and deep troubleshooting guides, see the following references:

* **[OAuth PKCE Architecture & Sequence](references/oauth_pkce_architecture.md)**: End-to-end token exchange diagrams and security considerations.
* **[TypeScript SDK Code Patterns](references/sdk_code_patterns.md)**: Production-ready code templates for `PersistentOAuthSession`, `LookerAuthProvider`, `OAuthCallback`, and `useLookerQuery`.
* **[Troubleshooting & Common Pitfalls](references/troubleshooting.md)**: Fixes for redirect URI mismatches, CORS preflight 403 errors, token loss, and bundler resolution bugs.
