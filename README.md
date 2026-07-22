# BBTSL Blade Ball Top Spender List

BBTSL is a public Blade Ball item site served by a Cloudflare Worker. It provides a searchable Top Spender list, media-backed item details, Discord sign-in, and role-gated staff tools.

## Architecture

- `public/` contains the static site, including the item list and public team directory.
- `src/worker.js` serves the site and API, handles Discord OAuth, sessions, authorization, R2-backed site state and media delivery, limited D1 access, and security headers.
- Public assets use the stable entrypoints `public/app.js`, `public/team.js`, and `public/styles.css`. Self-hosted fonts live under `public/fonts/`.
- Cloudflare R2 stores the site state, public item catalogue, media metadata, and rate-limit state. Cloudflare D1 stores user, role, and audit-log data. Sessions are signed cookies.
- `scripts/generate-secret-token.mjs` generates a high-entropy value for a Worker secret.

## Requirements

- Node.js 20 or newer
- npm
- Cloudflare access only when deploying or using remote resources

## Local development

```powershell
npm install
npm run cf:dev
```

The local Worker listens on the URL printed by Wrangler. It uses local Wrangler state for D1 and R2 unless Wrangler is explicitly instructed otherwise.

## Validation

```powershell
npm run check
npx wrangler deploy --dry-run
npm audit
git diff --check
```

## Configuration and secrets

Copy the placeholders in [.env.example](./.env.example) into a local `.dev.vars` file for local development. Never commit `.dev.vars`, `.env`, real credentials, session material, owner keys, or OAuth secrets.

The Worker requires these values:

- `ADMIN_SESSION_SECRET`
- `DISCORD_CLIENT_ID`
- `DISCORD_CLIENT_SECRET`
- `DISCORD_REDIRECT_URI` - an HTTPS Discord OAuth callback URL
- `V1_API_CLIENT_SECRETS` - JSON object of integration client ids to long random base secrets

Public runtime values such as `PUBLIC_SITE_URL` and `SITE_NAME` belong in `wrangler.jsonc`. Put sensitive Worker values in Cloudflare secrets for deployments.

Keep local scratch material out of commits. That includes `.dev.vars`, `.env`, Wrangler state, local SQLite files, one-off SQL helpers, export dumps, temporary JSON snapshots, and migration scratch files that only exist to support a local operation.

Generate a new random secret value with:

```powershell
npm run secrets:generate
```

## Private v1 API

`/api/v1/*` is a private integration API for approved server-side clients. The public website does not call it from the browser.

| Endpoint | Purpose |
| --- | --- |
| `GET /api/v1/health` | Service health and API version. |
| `GET /api/v1/swords` | Paginated item records. |
| `GET /api/v1/swords/%23ABC123` | One item by immutable card ID. URL-encode the `#`. |
| `GET /api/v1/team` | Active public team directory. |

### Authentication

Each client id in `V1_API_CLIENT_SECRETS` gets a rotating 128-bit daily access key derived from its base secret.

Send these headers with every `/api/v1/*` request:

- `x-bbtsl-api-client: <client-id>`
- `x-bbtsl-api-date: <UTC date in YYYY-MM-DD>`
- `authorization: Bearer <32 hex chars>`

The daily bearer key is derived as:

- `HMAC-SHA-256(baseSecret, "bbtsl-v1:<clientId>:<yyyy-mm-dd>")`
- take the first 32 hex characters of that digest

Generate the current key locally with:

```powershell
npm run api:v1-key -- owner
```

`/api/v1/swords` accepts optional `category`, `badge`, `demand`, `trend`, `cardId`, `search`, `sort`, `limit`, and `offset` parameters. Valid sort values are `value-desc`, `value-asc`, `name-asc`, `updated-desc`, `count-desc`, `count-asc`, `demand-desc`, `demand-asc`, and `trend-rank`.

List responses use:

```json
{
  "data": [],
  "meta": {
    "version": "v1",
    "generatedAt": "2026-07-22T00:00:00.000Z",
    "clientId": "owner",
    "total": 0,
    "limit": 50,
    "offset": 0,
    "hasMore": false,
    "filters": {
      "category": null,
      "demand": null,
      "trend": null,
      "cardId": null,
      "search": null,
      "sort": "value-desc"
    }
  }
}
```

Item responses return named fields such as `cardId`, `name`, `category`, `value`, `demand`, `trend`, `count`, `updatedAt`, `description`, `media`, and `flags`.

The private API is rate-limited to 180 requests per minute per client and IP combination, does not emit permissive CORS headers, and should only be called from trusted server-side code.

The staff site routes and mutation endpoints remain separate from the private integration API. They require Discord sign-in, role checks, same-origin requests, and the Worker request header checks enforced by `src/worker.js`.

## Contributor guidance

Read [CONTRIBUTING.md](./CONTRIBUTING.md) before opening a pull request. Use the issue forms for public bugs, feature requests, and data corrections. Do not report vulnerabilities publicly. Follow [SECURITY.md](./SECURITY.md).

## Public links

- Repository: [BBTS-Value-List/BBTSL-Website](https://github.com/BBTS-Value-List/BBTSL-Website)
- Live site: [bbtsl.lol](https://bbtsl.lol)
- Blade Ball experience: [Roblox](https://www.roblox.com/games/13772394625/Blade-Ball)

## Legal

BBTSL is an unofficial fan project and is not affiliated with Roblox, Blade Ball, or their owners. Read the live [Privacy Policy](https://bbtsl.lol/privacy), [Terms of Service](https://bbtsl.lol/terms), and [legal notice](./LEGAL.md) for data handling, terms, and rights concerns.

## Support and security

For public bugs, feature work, and data corrections, use the repository issue forms. For security, rights, or sensitive account concerns, use the contact paths in [SECURITY.md](./SECURITY.md) and [SUPPORT.md](./SUPPORT.md).
