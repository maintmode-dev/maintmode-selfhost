# Self-hosting MaintMode

Docker Compose setup and step-by-step instructions for running your own
MaintMode instance.

**MaintMode is a maintenance calendar for engineering teams.** Several teams
share the same infrastructure; two changes touch the same database in
overlapping windows and nobody notices until something breaks. MaintMode makes
that visible beforehand:

- One calendar of planned technical work — week and month views
- Planned window vs. actual execution time, tracked separately
- Shared resources: services, databases, clusters
- Conflict detection when work overlaps on the same resource
- Approval flows for changes that need sign-off
- Notifications to Slack, Telegram, and email
- Role-based access control and invitation-based user management
- Audit log

**Self-hosting is free and unlimited.** No seat counting, no licence check, no
phoning home. See [Licensing and telemetry](#licensing-and-telemetry) for what
that means concretely in the code.

Source repositories:

- Backend (Go) — [maintmode-dev/maintmode](https://github.com/maintmode-dev/maintmode)
- Frontend (Next.js) — [maintmode-dev/maintmode-ui](https://github.com/maintmode-dev/maintmode-ui)

Both are AGPL-3.0, as is this repository.

---

## Images

```
ghcr.io/maintmode-dev/maintmode:${MAINTMODE_VERSION}
ghcr.io/maintmode-dev/maintmode-ui:${MAINTMODE_UI_VERSION}
ghcr.io/maintmode-dev/maintmode-migrations:${MAINTMODE_VERSION}
```

Published to GHCR for `linux/amd64` and `linux/arm64`, so an Apple Silicon Mac
runs them natively. They are public — no `docker login` needed.

The version comes from `MAINTMODE_VERSION` / `MAINTMODE_UI_VERSION` in your
`.env`. CI tags every build three ways: by release tag (`v0.1.0`), by branch
(`main`) and by commit (`sha-abc1234`). **Pin a release tag in production** —
`main` moves whenever you happen to run `docker compose pull`, which is rarely
what you want. Keep `maintmode` and `maintmode-migrations` on the same version:
they ship as a pair, and the migrations image carries exactly the schema that
backend build expects.

---

## Prerequisites

- **Docker Engine 20.10+ with Compose v2.** Check with `docker compose version`
  (a space, not a hyphen). Docker Desktop on macOS and Windows includes it.
- **RAM:** 2 GB is enough for a small team. Postgres and the two application
  containers idle at roughly 700 MB together; the rest is headroom for the
  Next.js server under load. 4 GB is comfortable.
- **Disk:** 5 GB for images and a young database. Growth is driven almost
  entirely by the audit log, which is pruned to 365 days by default.
- **A Google account** with access to
  [Google Cloud Console](https://console.cloud.google.com/). Google OAuth is
  currently the only sign-in method — an instance without it has no login at
  all.
- **A public HTTPS URL**, if this is going to be reachable by anyone other than
  you. See [Behind a reverse proxy with TLS](#behind-a-reverse-proxy-with-tls).

### Ports

| Port | Where | Purpose |
| --- | --- | --- |
| 3000 | published on the host | The web interface. This is the only port published, and it binds to `127.0.0.1` by default. Change with `MAINTMODE_HTTP_PORT`. |
| 8000 | compose network only | Backend API. Reached by the frontend, never by a browser. |
| 8001 | compose network only | Backend health and readiness. |
| 5432 | compose network only | Postgres. Deliberately not published. |
| 6379 | compose network only | Valkey. Deliberately not published. |

Only port 3000 needs to be free on your host.

---

## Set up Google OAuth

**Do this first.** It is the step most installs stall on, and the stack will
not start without its output.

You need two values — a client ID and a client secret — and you need to
register one exact redirect URI.

### 1. Create or pick a project

Open [Google Cloud Console](https://console.cloud.google.com/) and create a
project (or select an existing one). The project is just a container; nothing
about it is billed for OAuth.

### 2. Configure the consent screen

Under **APIs & Services → OAuth consent screen**:

- **User type:** choose **Internal** if your Google Workspace organisation is
  the only group that will use MaintMode — it restricts sign-in to your own
  domain, which is a useful extra fence. Choose **External** otherwise.
- Fill in the app name, a support email, and a developer contact email. These
  are shown to users on the Google consent screen.
- **Scopes:** the defaults are enough. MaintMode reads only the basic profile —
  email, name, and the stable Google account identifier. It never requests
  access to Gmail, Drive, or Calendar.
- If you chose **External** and leave the app in *Testing*, only accounts you
  add as test users can sign in, and their consent expires after seven days.
  For anything beyond a trial, publish the app.

### 3. Create the OAuth client

Under **APIs & Services → Credentials → Create Credentials → OAuth client ID**:

- **Application type: Web application.** Not "Desktop app", not "TVs and
  limited-input devices" — those client types cannot use the redirect flow and
  produce confusing failures later.
- Give it a name (only you see it).
- **Authorized redirect URIs → Add URI.** This is the part that matters:

  ```
  <MAINTMODE_APP_BASE_URL>/api/auth/callback/google
  ```

  Substituting the URL your users will actually type. For a local trial:

  ```
  http://localhost:3000/api/auth/callback/google
  ```

  Behind TLS on your own domain:

  ```
  https://maintmode.example.com/api/auth/callback/google
  ```

  It must match **exactly** — scheme, host, port, and path. `http` vs `https`,
  a trailing slash, `localhost` vs `127.0.0.1`, and a missing port are all
  mismatches. Google compares the string, not the destination.

  You may register several URIs on one client, which is handy if you want to
  test locally and run in production against the same client.

- **Authorized JavaScript origins** can be left empty. MaintMode's login is a
  server-side redirect flow; nothing initiates it from browser JavaScript.

### 4. Copy the credentials

Google shows a **Client ID** (ends in `.apps.googleusercontent.com`) and a
**Client secret**. You will paste both into `.env`, and the client ID again
into the backend secrets file.

The client ID is not sensitive — it travels in every browser redirect. The
client secret is: it stays on your server and is never sent to a browser.

---

## Install

### 1. Clone

```bash
git clone https://github.com/maintmode-dev/maintmode-selfhost.git
cd maintmode-selfhost
```

### 2. Create the environment file

```bash
cp .env.example .env
```

### 3. Generate secrets

Four random values. Run each command and keep the output — you will paste them
in the next two steps.

```bash
# Database password
openssl rand -hex 32

# Frontend session secret (MAINTMODE_AUTH_SECRET)
openssl rand -hex 32

# Backend token signing key (jwt/issuer_private_key)
openssl rand -hex 32

# Encryption master key (crypto/kek/selfhost-1)
openssl rand -hex 32

# Signing key ID (jwt/issuer_kid) — shorter
openssl rand -hex 16
```

Generate each one separately; do not reuse a single value across all five.

A note on the signing key: it is a raw 32-byte P-256 scalar, hex-encoded, so
`openssl rand -hex 32` is exactly right. Do **not** use
`openssl ecparam -genkey ... | xxd` — that produces a DER structure rather than
a raw scalar, and the backend rejects it at startup. The backend also rejects a
key made of one repeated byte, so a placeholder fails loudly instead of quietly
becoming a guessable signing key.

### 4. Fill in `.env`

Open `.env` and set:

| Variable | Value |
| --- | --- |
| `POSTGRES_PASSWORD` | the database password you generated |
| `MAINTMODE_AUTH_SECRET` | the session secret (min 32 chars) |
| `MAINTMODE_APP_BASE_URL` | the URL users will type, no trailing slash |
| `MAINTMODE_GOOGLE_OAUTH_CLIENT_ID` | from Google Cloud Console |
| `MAINTMODE_GOOGLE_OAUTH_CLIENT_SECRET` | from Google Cloud Console |

`MAINTMODE_APP_BASE_URL` must match the redirect URI you registered with
Google. If you registered `http://localhost:3000/api/auth/callback/google`,
this is `http://localhost:3000`.

### 5. Create the backend secrets file

```bash
cp config/app.secrets.example.yaml config/app.secrets.yaml
chmod 644 config/app.secrets.yaml
```

Open it and replace every `REPLACE_ME`:

| Key | Value |
| --- | --- |
| `db/dsn` | the same database password as `POSTGRES_PASSWORD`, inside the connection string |
| `oauth/google/client_id` | the same client ID as in `.env` |
| `jwt/issuer_private_key` | the 64-hex-char signing key |
| `jwt/issuer_kid` | the 32-hex-char key ID |
| `crypto/kek/selfhost-1` | the 64-hex-char encryption key |

Two values appear in both files and **must** match exactly: the database
password and the Google client ID. A mismatched password fails at startup with
an authentication error; a mismatched client ID fails later, at login, with an
audience validation error that is much harder to read.

The `chmod 644` matters: the backend container runs as an unprivileged user and
cannot read a `600` file owned by you.

### 6. Review the backend config

`config/app.config.yaml` works unchanged, with one thing worth checking:
`app.frontend_url` should equal `MAINTMODE_APP_BASE_URL`. It is where the
backend sends a browser after a successful OAuth exchange, and a stale value
drops users somewhere unexpected at the end of an otherwise working login.

### 7. Start

```bash
docker compose up -d
```

Watch it come up:

```bash
docker compose ps
docker compose logs -f
```

The order is enforced by health checks: Postgres and Valkey become healthy, the
migration job runs to completion and exits, the backend starts and reports
ready, then the frontend starts. First start takes a minute or two.

### 8. Open it

<http://localhost:3000>

Then read the next section before you do anything else.

---

## First login

> **⚠️ The first person to sign in becomes the administrator.**
>
> On a fresh installation with zero administrators, the first successful Google
> login is granted the admin role automatically. There is no setup token, no
> invitation, and no lock on this — it is decided purely by who arrives first.
>
> **Log in yourself, immediately after starting the stack, before the instance
> is reachable by anyone else.** If a stranger reaches your login page before
> you do, they become the administrator of your instance.

This is why `compose.yaml` binds port 3000 to `127.0.0.1` by default: a fresh
`docker compose up` is not exposed to your network until you deliberately
change that.

The safe order is:

1. `docker compose up -d`
2. Open the instance and sign in with Google — you are now the administrator
3. Verify your account shows the admin role
4. *Only then* expose it (reverse proxy, or `MAINTMODE_BIND_ADDRESS=0.0.0.0`)

After bootstrap, the instance is invite-only: `allow_open_signup: false` in
`config/app.config.yaml` means an unknown Google account with no invitation is
rejected. Add your colleagues from the admin UI, which sends them invitations.

If you accidentally let someone else bootstrap, the fastest fix on an instance
with no real data is to start over:

```bash
docker compose down -v   # ⚠️ deletes the database
docker compose up -d
```

---

## Behind a reverse proxy with TLS

For anything beyond a local trial, put a reverse proxy in front and terminate
TLS there. MaintMode does not terminate TLS itself.

Three things must agree, and the login flow breaks if any one of them drifts:

1. `MAINTMODE_APP_BASE_URL` in `.env` — the public HTTPS URL
2. `app.frontend_url` in `config/app.config.yaml` — the same value
3. The Authorized redirect URI in Google Cloud Console —
   `<that URL>/api/auth/callback/google`

Update all three together whenever the public URL changes, then
`docker compose up -d` to apply.

Keep the frontend bound to `127.0.0.1` (the default) when the proxy runs on the
same host. The proxy reaches it over loopback; nothing else can.

### Caddy

Caddy obtains and renews a certificate automatically. The whole config:

```caddyfile
maintmode.example.com {
	reverse_proxy 127.0.0.1:3000
}
```

Point your domain's A record at the host first — Caddy needs to answer an ACME
challenge on port 80 before it can issue the certificate.

### nginx

```nginx
server {
    listen 443 ssl;
    server_name maintmode.example.com;

    ssl_certificate     /etc/letsencrypt/live/maintmode.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/maintmode.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        # The frontend builds absolute URLs from these. Without them it
        # generates http:// links against an https:// site and the OAuth
        # redirect fails.
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

server {
    listen 80;
    server_name maintmode.example.com;
    return 301 https://$host$request_uri;
}
```

Certificates come from certbot or your own CA; nginx will not obtain them for
you the way Caddy does.

---

## Updating

```bash
docker compose pull
docker compose up -d
```

`pull` fetches new images, `up -d` recreates only the containers whose image
changed. Expect under a minute of downtime.

**Migrations run automatically.** The `migrations` service runs before the
backend on every `up`, applies any new migrations, and exits. Already-applied
migrations are skipped, so running it repeatedly is safe. If it fails, the
backend does not start at all — deliberately, since a backend running against a
schema it does not expect corrupts data in ways that are far worse than
downtime.

**Back up before updating.** See below. A schema migration is not reversible by
`docker compose down`.

**Pin your versions.** CI tags images by release tag (`v1.2.3`), by branch
(`main`) and by commit (`sha-abc1234`), and points `latest` at the newest
release. Set `MAINTMODE_VERSION` to a release tag in production: `main` and
`latest` both move whenever you happen to run `pull`, which is rarely what you
want. Keep the backend and
migrations images on the same version — they ship as a pair, and the migrations
image contains exactly the schema that backend build expects.

To roll back, set the previous tag and `up -d` again — but note that a
migration applied by the newer version is *not* undone, and an older backend
may not tolerate a newer schema. Restoring from backup is the reliable path.

---

## Backup and restore

Two things need backing up, and **a database dump alone is not enough**:

1. **The database** — all your data.
2. **`config/app.secrets.yaml`** — specifically `crypto/kek/selfhost-1`. It is
   the key that decrypts your stored integration credentials. Restore a
   database without it and MaintMode comes up with your Slack and SMTP
   integrations listed but their credentials unreadable, and you have to enter
   them again.

Store them separately: a backup holding both an encrypted dump and the key that
decrypts it offers little protection.

Valkey does not need backing up — it holds only rate-limit counters and
short-lived locks, all of which regenerate.

### Back up the database

```bash
docker compose exec -T postgres \
  pg_dump -U maintmode -d maintmode -Fc \
  > maintmode-$(date +%F).dump
```

`-Fc` is Postgres's custom format: compressed, and restorable with
`pg_restore`. Verify the file is non-trivial in size, then copy it off the
host — a backup on the same disk as the database is not a backup.

For a nightly cron job:

```bash
0 3 * * * cd /path/to/maintmode-selfhost && docker compose exec -T postgres pg_dump -U maintmode -d maintmode -Fc > /backups/maintmode-$(date +\%F).dump
```

(`%` must be escaped as `\%` in a crontab.)

### Restore

Restoring replaces existing data, so stop the applications first — leaving them
running against a database being rewritten produces inconsistent results.

```bash
# 1. Stop the apps, keep the database running
docker compose stop ui maintmode

# 2. Drop and recreate the schema
docker compose exec -T postgres \
  psql -U maintmode -d maintmode \
  -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"

# 3. Restore
docker compose exec -T postgres \
  pg_restore -U maintmode -d maintmode --no-owner \
  < maintmode-2026-01-15.dump

# 4. Start back up
docker compose up -d
```

Make sure `config/app.secrets.yaml` holds the **same** `crypto/kek/selfhost-1`
value it had when the dump was taken, before starting back up.

To restore onto a brand-new host, copy `.env` and `config/app.secrets.yaml`
across first, run `docker compose up -d` once to create the volumes and apply
migrations, then follow the steps above.

---

## Troubleshooting

Start here for anything:

```bash
docker compose ps          # which containers are up, healthy, or exited
docker compose logs ui     # frontend
docker compose logs maintmode
docker compose logs migrations
```

### `redirect_uri_mismatch` from Google

The most common failure. Google is comparing the redirect URI your instance
sent against the ones registered on the client, as exact strings.

The error page shows the URI that was actually sent. Compare it,
character by character, against **APIs & Services → Credentials → your client →
Authorized redirect URIs**. Usual culprits:

- `http` vs `https` — behind a proxy, `MAINTMODE_APP_BASE_URL` must be the
  `https` URL, not the internal `http` one
- A trailing slash on `MAINTMODE_APP_BASE_URL`
- `localhost` vs `127.0.0.1` — different strings to Google
- A missing or extra port
- The path — it is `/api/auth/callback/google`, nothing else

After correcting either side, `docker compose up -d`. Google's changes can take
a few minutes to propagate.

### The frontend container exits immediately

Almost always a missing or invalid auth variable. All four of
`MAINTMODE_AUTH_SECRET`, `MAINTMODE_APP_BASE_URL`,
`MAINTMODE_GOOGLE_OAUTH_CLIENT_ID`, and `MAINTMODE_GOOGLE_OAUTH_CLIENT_SECRET`
are validated when the auth module loads — before any page renders. If one is
missing, nothing serves, including `/login`. You get a container that starts
and dies rather than a site with a broken login page.

```bash
docker compose logs ui
```

The error names the offending variable. Check:

- All four are present in `.env` with no empty values
- `MAINTMODE_AUTH_SECRET` is at least 32 characters
- `MAINTMODE_APP_BASE_URL` is a full URL including the scheme
- No stray quotes around values — `.env` is not shell, so `KEY="value"` makes
  the quotes part of the value

Compose validates these upfront, so a missing one usually surfaces as an error
from `docker compose up` naming the variable, before anything starts.

### Login fails after Google accepts you

Google authenticated you, but the backend rejected the resulting token. Nearly
always the client ID in `config/app.secrets.yaml` (`oauth/google/client_id`)
differs from `MAINTMODE_GOOGLE_OAUTH_CLIENT_ID` in `.env`. The backend
validates the token's audience against its own copy, and a mismatch fails every
login.

```bash
grep GOOGLE_OAUTH_CLIENT_ID .env
grep client_id config/app.secrets.yaml
```

They must be identical. Fix and `docker compose up -d`.

The other possibility, once you are past bootstrap: the account has no
invitation and `allow_open_signup` is `false`, so signup is refused by design.
Invite the account from the admin UI.

### Migrations did not run

```bash
docker compose logs migrations
```

- **Authentication failure:** the password in `db/dsn` does not match
  `POSTGRES_PASSWORD`. Note that `POSTGRES_PASSWORD` only takes effect on the
  *first* start — if you changed it afterwards, Postgres still enforces the
  original. Either set `db/dsn` back to the original, or wipe the volume with
  `docker compose down -v` (destroys all data).
- **Cannot connect:** Postgres has not become healthy. `docker compose ps`, and
  check its logs.
- **A migration errored:** the schema is now partly applied. Restore from
  backup rather than improvising; the backend deliberately refuses to start.

The backend will not start until this job exits 0, so a stuck migration
presents as a backend that never starts.

### `denied`, `401 Unauthorized` or `not found` pulling images

The images are public, so this is almost always a stale credential rather than
a permission you are missing. Docker sends whatever it has stored for the
registry, and a token that no longer grants access to the package fails the
pull that would have succeeded anonymously:

```bash
docker logout ghcr.io
docker compose pull
```

If it still fails, check the tag exists — `MAINTMODE_VERSION` must name a
published release tag, a branch, a `sha-` tag or `latest`.

### Port already in use

```
Error starting userland proxy: listen tcp4 127.0.0.1:3000: bind: address already in use
```

Something else holds port 3000. Either stop it, or pick another port in `.env`:

```
MAINTMODE_HTTP_PORT=3001
```

If you are **not** behind a reverse proxy, the port is part of your public URL,
so also update `MAINTMODE_APP_BASE_URL`, `app.frontend_url` in
`config/app.config.yaml`, and the redirect URI in Google Cloud Console. Behind
a proxy, only the proxy's upstream needs changing.

### The backend never becomes healthy

```bash
docker compose logs maintmode
```

Startup validates configuration strictly and fails loudly rather than running
degraded:

- `jwt.issuer_private_key is unusable` — not 64 hex characters, or generated
  with `ecparam` instead of `rand -hex 32`
- `jwt.issuer_private_key is a placeholder` — all one repeated byte
- A missing secrets key — every `<secret:...>` reference in
  `app.config.yaml` must exist in `app.secrets.yaml`
- A permission error reading `/app/app.secrets.yaml` — the container's
  unprivileged user cannot read it; `chmod 644 config/app.secrets.yaml`

### Starting completely over

```bash
docker compose down -v
```

Deletes the containers **and the volumes**, so all data is gone. `.env` and
`config/app.secrets.yaml` survive, since they are files in your working tree.
The next `up -d` bootstraps a fresh instance — including a fresh first-login
admin grant.

---

## Licensing and telemetry

### Your instance sends nothing anywhere

**Self-hosted MaintMode is free, unlimited, and does not phone home.** No seat
counting, no licence check, no usage reporting, no update pings.

This is not a policy promise — it is how the code is structured, and you can
verify it. The backend contains licence-enforcement code because the same
binary also runs the paid hosted service. That code activates only when
**both** a console URL and an instance token are configured:

```go
func (c LicenseConfig) Enabled() bool {
	return c.URL != "" && c.InstanceToken != ""
}
```

A half-configured block stays off. `config/app.config.yaml` in this repository
has no `license:` section at all, so both are empty and the gate reports
disabled. In that state the heartbeat job is never registered, the licence HTTP
client is never constructed, no seat cap is applied, and no request leaves your
instance. There is no code path that reaches the licence server without those
two values.

The only outbound connections a running instance makes are:

- **Google's JWKS endpoint** (`googleapis.com`), to fetch the public keys that
  verify login tokens. Required for OAuth; it carries no data about you.
- **Whatever you configure yourself** — Slack, Telegram, or your SMTP server,
  once you set up notifications.

Nothing else.

### The code is AGPL-3.0

All three repositories — backend, frontend, and this one — are licensed under
the **GNU Affero General Public License v3.0**. The full text is in
[LICENSE](LICENSE).

What that means in practice:

- **Running it internally, modified or not, obliges you to nothing.** Use it for
  your team, change it, keep the changes private.
- **The network clause (section 13) applies if you offer a modified version to
  users over a network.** Run your own modified MaintMode as a service that
  other people use, and those users are entitled to the source of your modified
  version. Running an *unmodified* version, or running a modified one only for
  your own organisation's internal use, does not trigger this.
- **Redistributing it, modified or not, requires the same licence** and that
  you pass on the source.

This is a summary, not legal advice. Read [LICENSE](LICENSE) if the distinction
matters to you.

---

## What is in this repository

| File | Purpose |
| --- | --- |
| `compose.yaml` | The stack: Postgres, Valkey, migrations, backend, frontend |
| `.env.example` | Template for `.env` — database, session, and OAuth settings |
| `config/app.config.yaml` | Backend configuration, mounted read-only. Committed; holds no secrets |
| `config/app.secrets.example.yaml` | Template for `config/app.secrets.yaml` — signing key, encryption key, database DSN |
| `.gitignore` | Keeps `.env`, secrets, and dumps out of git |
| `LICENSE` | AGPL-3.0 |

## Getting help

- Bugs and questions about the backend or the API —
  [maintmode-dev/maintmode](https://github.com/maintmode-dev/maintmode/issues)
- Bugs in the web interface —
  [maintmode-dev/maintmode-ui](https://github.com/maintmode-dev/maintmode-ui/issues)
- Problems with this Compose setup or these instructions — open an issue here

When reporting a bring-up problem, include the output of `docker compose ps` and
the relevant `docker compose logs`, with secrets redacted.
