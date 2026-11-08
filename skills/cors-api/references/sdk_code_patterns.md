# Looker TypeScript SDK Code Patterns for CORS OAuth 2.0 PKCE

This reference provides production-ready TypeScript patterns for persisting OAuth 2.0 PKCE tokens in `sessionStorage`, initializing `Looker40SDK`, and generating user-delegated embed URLs (`create_embed_url_as_me`).

---

## 1. Subpath Imports for Bundlers (Vite / esbuild / Webpack 5)

To avoid TypeScript interface re-export errors in modern bundlers, import `Looker40SDK` and model types from `@looker/sdk/lib/4.0/*`:

```typescript
import { Looker40SDK } from '@looker/sdk/lib/4.0/methods';
import type { IWriteQuery, IEmbedUrlResponse } from '@looker/sdk/lib/4.0/models';
import {
  AuthToken,
  BrowserServices,
  DefaultSettings,
  OAuthSession,
  iauthToken,
} from '@looker/sdk-rtl';
```

---

## 2. `PersistentOAuthSession` (`sessionStorage` Token Persistence)

By default, `@looker/sdk-rtl`'s `OAuthSession` keeps `activeToken` and `code_verifier` only in memory. Subclass `OAuthSession` so page refreshes and route transitions preserve the authenticated session:

```typescript
const TOKEN_STORAGE_KEY = 'looker_oauth_token';

export class PersistentOAuthSession extends OAuthSession {
  constructor(config: {
    baseUrl: string;
    apiUrl: string;
    clientId: string;
    redirectUri: string;
  }) {
    const settings = {
      ...DefaultSettings(),
      base_url: config.apiUrl,
      verify_ssl: true,
       Looker_url: config.baseUrl,
      client_id: config.clientId,
      redirect_uri: config.redirectUri,
    };
    super(new BrowserServices({ settings }));

    const saved = sessionStorage.getItem(TOKEN_STORAGE_KEY);
    if (saved) {
      try {
        const parsed: iauthToken = JSON.parse(saved);
        this.activeToken.setToken(parsed);
      } catch {
        sessionStorage.removeItem(TOKEN_STORAGE_KEY);
      }
    }
  }

  override async login(code?: string): Promise<AuthToken> {
    const token = await super.login(code);
    if (this.isAuthenticated()) {
      sessionStorage.setItem(TOKEN_STORAGE_KEY, JSON.stringify(token));
    }
    return token;
  }

  override async logout(): Promise<boolean> {
    sessionStorage.removeItem(TOKEN_STORAGE_KEY);
    return super.logout();
  }
}
```

---

## 3. OAuth Callback Component (SPA State Navigation)

Redeem the authorization `code` in your `/callback` route and navigate using client-side router state (`navigate(..., { replace: true })`) rather than `window.location.replace`:

```tsx
import React, { useEffect } from 'react';
import { useNavigate, useSearchParams } from 'react-router-dom';
import { authSession } from './lookerClient';

export const OAuthCallback: React.FC = () => {
  const [searchParams] = useSearchParams();
  const navigate = useNavigate();

  useEffect(() => {
    const code = searchParams.get('code');
    if (!code) return;

    authSession
      .login(code)
      .then(() => {
        const returnTo = sessionStorage.getItem('looker_return_to') || '/';
        sessionStorage.removeItem('looker_return_to');
        navigate(returnTo, { replace: true });
      })
      .catch((err) => {
        console.error('Looker OAuth PKCE redemption failed:', err);
      });
  }, [searchParams, navigate]);

  return <div>Completing sign-in...</div>;
};
```

---

## 4. Generating Embed URLs as the PKCE-Authenticated User (`create_embed_url_as_me`)

Once authenticated via OAuth 2.0 PKCE (`cors_api`), a browser application can generate a single-use, 5-minute TTL embed URL for a Looker dashboard or Explore scoped to the logged-in user without a backend embed secret:

```typescript
export async function getUserDelegatedEmbedUrl(
  sdk: Looker40SDK,
  lookerBaseUrl: string,
  dashboardId: string,
  theme?: string
): Promise<string> {
  const params = new URLSearchParams({ sdk: '2', _theme: theme || '' });
  const targetUrl = `${lookerBaseUrl}/embed/dashboards/${dashboardId}?${params.toString()}`;

  const response = await sdk.ok<IEmbedUrlResponse>(
    sdk.create_embed_url_as_me({
      target_url: targetUrl,
      session_length: 3600,
      force_logout_login: true,
    })
  );

  if (!response.url) {
    throw new Error('Looker did not return an embed URL from create_embed_url_as_me');
  }
  return response.url;
}
```
