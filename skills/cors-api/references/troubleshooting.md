# Troubleshooting Looker CORS API, OAuth 2.0 PKCE, & Embedded OAuth Connections

---

## 1. Resolving `accounts.google.com` CSP `403` Errors on Embedded BigQuery OAuth Dashboards

### Symptom
A parent application authenticates the user via Google OAuth, completes the Looker OAuth 2.0 PKCE (`cors_api`) flow, and mounts an embedded Looker dashboard using `create_embed_url_as_me` (`POST /api/4.0/embed/token_url/me`). When the dashboard loads against a **BigQuery OAuth connection**, tiles fail with a "Log In" prompt; clicking it attempts to navigate the iframe to `https://accounts.google.com/o/oauth2/v2/auth`, which fails with an HTTP `403` / browser error (`Refused to frame 'https://accounts.google.com/' because an ancestor violates the following Content Security Policy directive: "frame-ancestors ..."`).

### Root Cause: Two Independent OAuth Layers in Looker
Looker maintains two separate authentication states:
1. **Application Session (`:embed_user` Warden scope)**: Populated by `cors_api` + `create_embed_url_as_me` (`GET /login/embed?t=<token>`).
2. **Database Connection OAuth State (`Looker::Model::OAuthUserState`)**: Stores the per-user Google Cloud `access_token` and `refresh_token` required to run BigQuery jobs under the viewer's identity.

Authenticating via Looker OAuth PKCE only establishes Layer 1. When a tile executes against a BigQuery OAuth connection without an active token in `OAuthUserState`, Looker raises `Looker::DBOAuthLoginRequiredError` and renders an in-tile login redirect (`/external_oauth/authenticate/<id>`), which either fails under the `:embed_user` path allowlist or redirects the iframe to `accounts.google.com` (blocked by Google's `X-Frame-Options: DENY` and `CSP: frame-ancestors`).

### Architectural Resolution: Headless Token Seeding (`create_oauth_application_user_state`)
Because the parent application already authenticates the user with Google OAuth before running the Looker client flow:
1. **Share the Google OAuth 2.0 Client ID**: Configure the Looker BigQuery connection to use the **same custom Google Cloud OAuth 2.0 Client ID and Secret** as the parent web application.
2. **Request BigQuery Scopes at Parent Login**: Include `https://www.googleapis.com/auth/bigquery` (and `access_type=offline` if refresh tokens are needed) during the parent application's top-level Google login.
3. **Seed `OAuthUserState` Before Mounting the Iframe**: From your backend (or via the Looker API), push the user's Google `access_token` and `refresh_token` directly into Looker using `create_oauth_application_user_state` (`POST /api/4.0/external_oauth_applications/user_state`) before loading the embedded dashboard:

```python
import datetime
import looker_sdk
from looker_sdk import models40 as models

sdk = looker_sdk.init40()

def seed_bigquery_oauth_user_state(
    looker_user_id: str,
    external_oauth_app_id: str,
    google_access_token: str,
    google_refresh_token: str,
    expires_in_seconds: int = 3500,
):
    expires_at = (
        datetime.datetime.now(datetime.timezone.utc)
        + datetime.timedelta(seconds=expires_in_seconds)
    ).strftime("%Y-%m-%dT%H:%M:%SZ")

    sdk.create_oauth_application_user_state(
        body=models.WriteOAuthUserStateRequest(
            user_id=looker_user_id,
            external_oauth_application_id=external_oauth_app_id,
            access_token=google_access_token,
            refresh_token=google_refresh_token,
            access_token_expires_at=expires_at,
        )
    )
```

> **Alternative (Popup Window for `/external_oauth/authenticate`)**: If headless token seeding is not used, open Looker's `/external_oauth/authenticate/<app_id>` flow in a top-level popup window (`window.open`) prior to mounting the iframe so `accounts.google.com` never loads inside an iframe.

---

## 2. Common OAuth 2.0 PKCE & CORS Errors

| Error / Symptom | Root Cause | Resolution |
| :--- | :--- | :--- |
| **`400 Bad Request` on `/auth` or `/api/token` (`Invalid redirect_uri`)** | `redirect_uri` contains a wildcard (`*`), trailing slash mismatch, dynamic subpath, or query string. | Register exact callback URLs in `redirect_uris: string[]` and pass dynamic return routes via the OAuth `state` parameter or `sessionStorage`. |
| **CORS Preflight (`OPTIONS`) Fails with `403 Forbidden`** | Frontend origin (`https://host:port`) is missing from Looker's Embedded Domain Allowlist. | Add the exact protocol, hostname, and port (no trailing slash) to **Admin > Embed > Embedded Domain Allowlist**. |
| **`window.crypto.subtle is undefined` during PKCE challenge generation** | Browser disables Web Crypto API on non-secure (`http://`) origins other than `localhost`/`127.0.0.1`. | Serve local and staging environments over `https://` (`HTTPS=true` or Vite `@vitejs/plugin-basic-ssl`). |
| **`401 Unauthorized` after page reload** | Default `OAuthSession` stores tokens in volatile JS memory only. | Use `PersistentOAuthSession` backed by `sessionStorage` (see `sdk_code_patterns.md`). |
