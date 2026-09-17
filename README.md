# IOM ITB SSO

This folder is a self-contained Keycloak for IOM ITB SSO.

> [!NOTE]
> Please refer to [iom-itb-sso-poc](https://github.com/l0stplains/iom-itb-sso-poc) and [integration guide](https://github.com/l0stplains/iom-itb-sso-poc/blob/main/docs/SSO-INTEGRATION-GUIDE.md) for technical explanations.

Contents:
- keycloak/realm-export.json: realm definition
- keycloak/themes/iom-itb: login theme
- postgres/init.sql: postgres initialization for Keycloak DB
- docker-compose.local.yml: local stack for development
- docker-compose.prod.yml: production-oriented stack
- .env.example: environment template
- WORKFLOW.md: operational guide for typical SSO changes and Coolify deployment

## Local development quick start

1. Copy env template:

```bash
cp .env.example .env
```

2. Start local stack:

```bash
docker compose -f docker-compose.local.yml up --build
```

3. Open:
- Keycloak: http://localhost:8080
- Admin user: KEYCLOAK_ADMIN / KEYCLOAK_ADMIN_PASSWORD from .env

## Production with Docker Compose

1. Prepare .env with strong passwords and public URL.
2. Run:

```bash
docker compose -f docker-compose.prod.yml up -d
```

3. Put Keycloak behind reverse proxy/platform (Coolify in target setup).

## Notes

> [!IMPORTANT]
> Production SSO runs on Coolify at `https://sso-ng.iom-itb.id`
> (issuer `https://sso-ng.iom-itb.id/realms/iom-itb-sso`, set `KEYCLOAK_PUBLIC_URL=https://sso-ng.iom-itb.id`).
> Client `frontend-1` in `realm-export.json` still uses the staging `*.kirisame.jp.net` values.
> Before moving to a new server or production domain, update all related URLs and settings, including:
> - `keycloak/realm-export.json` client URLs (`redirectUris`, `webOrigins`, `rootUrl`, `baseUrl`, `post.logout.redirect.uris`)
> - `.env` / Coolify variables (especially `KEYCLOAK_PUBLIC_URL`)
> - application-side OIDC config in frontend/backend repos (issuer/authority, redirect/logout URIs)

- The imported realm is iom-itb-sso.
- When changing realm-export.json, re-import behavior depends on your Keycloak state and existing DB data.
- For repeatable environments, keep realm JSON and env contract versioned.

## Clients registered in the realm

| Client ID | Type | Used by |
|---|---|---|
| `frontend-1` | public (PKCE) | iom-itb-app-NG |
| `backend-api` | bearer-only | iom-itb-api-NG (token audience) |
| `bankes-app` | confidential | BanKes-OTA-NG / Bankes (NextAuth), callback `<BANKES_PUBLIC_URL>/api/auth/callback/keycloak` |
| `ota-ku-app` | confidential | BanKes-OTA-NG / OTA-KU, callback `<BANKES_PUBLIC_URL>/ota/integrations/keycloak/callback` |
| `iom-itb-api-admin` | service account (`manage-users`) | iom-itb-api-NG `POST /auth/register` |

Client secrets and `BANKES_PUBLIC_URL` are injected into `realm-export.json` from environment
variables (`${VAR}` placeholders), see `.env.example`.

> [!WARNING]
> `--import-realm` only imports the realm when it does **not** exist yet. The realm on
> `sso-ng.iom-itb.id` already exists, so changes to `realm-export.json` are ignored there.
> Add the clients through the Admin Console instead (see below).

### Adding the BanKes-OTA-NG / API clients to the running realm (sso-ng.iom-itb.id)

1. Open `https://sso-ng.iom-itb.id/admin` → realm **iom-itb-sso** → *Realm settings* →
   *Action* (top right) → **Partial import**.
2. Upload `keycloak/partial-import-bankes-ota-clients.json`, tick **Clients**,
   choose *If a resource exists* → **Skip**, then **Import**.
3. For `bankes-app`, `ota-ku-app` and `iom-itb-api-admin`: *Clients* → client →
   *Credentials* → **Regenerate** and copy the secret into the apps:
   - `bankes-app` → BanKes-OTA-NG `BANKES_KEYCLOAK_CLIENT_SECRET`
   - `ota-ku-app` → BanKes-OTA-NG `KEYCLOAK_CLIENT_SECRET`
   - `iom-itb-api-admin` → iom-itb-api-NG `KEYCLOAK_ADMIN_CLIENT_SECRET`
4. `iom-itb-api-admin` → *Service account roles* → *Assign role* → filter by clients →
   `realm-management`: `manage-users`, `view-users`, `query-users`, `view-realm`.
5. Check *Client scopes* of `bankes-app` and `ota-ku-app`: `roles`, `email`, `profile`
   must be **Default** (the apps read `realm_access.roles` from the access token).
6. Redeploy the BanKes-OTA-NG and iom-itb-api-NG resources in Coolify so they pick up the secrets.
