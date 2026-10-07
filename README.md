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
  [Google Cloud Console](https://console.cloud.google.com/). Google is the
  sign-in method this guide sets up for your users.
- **An SMTP server** the backend can reach. The instance is invite-only and
  invitations go out by email; you enter the server in the admin UI after the
  first login. The same transport enables sign-in by emailed code.
- **An HTTPS URL on a real domain**, if this is going to be used by anyone
  other than you. See [Behind a reverse proxy with TLS](#behind-a-reverse-proxy-with-tls)
  and [Does it have to be on the internet?](#does-it-have-to-be-on-the-internet)

### Ports

| Port | Where | Purpose |
| --- | --- | --- |
| 3000 | published on the host | The gateway: the web interface, plus the two backend routes of Google sign-in. This is the only port published, and it binds to `127.0.0.1` by default. Change with `MAINTMODE_HTTP_PORT`. |
| 8000 | compose network only | Backend API. Called by the frontend server. A browser reaches only `/auth/api/v1/login/oauth/{provider}/start` and `/callback`, through the gateway; every other backend route stays internal. |
| 8001 | compose network only | Backend health and readiness. Never published, not even through the gateway. |
| 5432 | compose network only | Postgres. Deliberately not published. |
| 6379 | compose network only | Valkey. Deliberately not published. |

Only port 3000 needs to be free on your host.

---

## Set up Google OAuth

**Do this first.** It is the step most installs stall on.

You need two values — a client ID and a client secret — and you need to
register one exact redirect URI.

How sign-in works, because it explains every value below: the "Sign in with
Google" button sends the browser to your instance's backend, which redirects it
to Google; Google sends the browser back to the backend's callback, and the
backend exchanges the code with Google directly, using the client secret. The
backend is a *confidential* client: the secret lives on your server, never in a
browser.

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
  <MAINTMODE_APP_BASE_URL>/auth/api/v1/login/oauth/google/callback
  ```

  Substituting the URL your users will actually type. For a local trial:

  ```
  http://localhost:3000/auth/api/v1/login/oauth/google/callback
  ```

  Behind TLS on your own domain:

  ```
  https://maintmode.example.com/auth/api/v1/login/oauth/google/callback
  ```

  It must match **exactly** — scheme, host, port, and path. `http` vs `https`,
  a trailing slash, `localhost` vs `127.0.0.1`, and a missing port are all
  mismatches. Google compares the string, not the destination.

  Note the `/auth` prefix. The backend itself serves
  `/api/v1/login/oauth/...`; the gateway puts it under `/auth` and strips the
  prefix on the way in. Google and the browser see the **external** form, with
  `/auth`, and that is the one you register. Registering the internal form
  fails with `redirect_uri_mismatch` on Google's side, which never reaches
  MaintMode's logs.

  You may register several URIs on one client, which is handy if you want to
  test locally and run in production against the same client.

- **Authorized JavaScript origins** can be left empty. MaintMode's login is a
  server-side redirect flow; nothing calls Google from browser JavaScript.

### 4. Copy the credentials

Google shows a **Client ID** (ends in `.apps.googleusercontent.com`) and a
**Client secret**. Both go to the backend:

- the client ID into `config/app.config.yaml`, at
  `oauth_providers.providers.google.client_id`;
- the client secret into `config/app.secrets.yaml`, under
  `auth_provider/google/client_secret`. The config file only references it
  (`<secret:auth_provider/google/client_secret>`); never paste the secret into
  the config file itself.

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

Six random values. Run each command and keep the output — you will paste them
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

# Break-glass password (bootstrap/password) — you will type this one
openssl rand -base64 24
```

Generate each one separately; do not reuse a single value across them.

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

`MAINTMODE_APP_BASE_URL` must be the start of the redirect URI you registered
with Google. If you registered
`http://localhost:3000/auth/api/v1/login/oauth/google/callback`, this is
`http://localhost:3000`.

### 5. Create the backend secrets file

```bash
cp config/app.secrets.example.yaml config/app.secrets.yaml
chmod 644 config/app.secrets.yaml
```

Open it and replace every `REPLACE_ME`:

| Key | Value |
| --- | --- |
| `db/dsn` | the same database password as `POSTGRES_PASSWORD`, inside the connection string |
| `auth_provider/google/client_secret` | the Client secret from Google Cloud Console |
| `jwt/issuer_private_key` | the 64-hex-char signing key |
| `jwt/issuer_kid` | the 32-hex-char key ID |
| `crypto/kek/selfhost-1` | the 64-hex-char encryption key |
| `bootstrap/password` | the break-glass password (at least 20 characters) |

The database password appears in both files and **must** match exactly; a
mismatch fails at startup with an authentication error.

Keep the break-glass password somewhere safe — a password manager, not this
host only. On a fresh instance it is the only way to sign in as the first
administrator.

The `chmod 644` matters: the backend container runs as an unprivileged user and
cannot read a `600` file owned by you.

### 6. Review the backend config

`config/app.config.yaml` needs three edits; everything else works unchanged:

| Key | Value |
| --- | --- |
| `app.frontend_url` | the same value as `MAINTMODE_APP_BASE_URL` |
| `oauth_providers.providers.google.client_id` | the Client ID from Google Cloud Console |
| `oauth_providers.providers.google.redirect_uri` | exactly the redirect URI you registered with Google |

`app.frontend_url` is where the backend sends a browser at the end of sign-in;
a stale value drops users somewhere unexpected after an otherwise working
login. Leave `app.oauth_callback_path` and `app.oauth_cookie_path` as they are:
they are fixed by the frontend and the gateway.

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
ready, then the frontend and the gateway start. First start takes a minute or
two.

### 8. Open it

<http://localhost:3000>

Then read the next section before you do anything else.

---

## First login

A fresh instance is invite-only and has no users, so nobody can sign in with
Google yet — an unknown account without an invitation is refused. The first
administrator is the **break-glass** account, which signs in with the
`bootstrap/password` from `config/app.secrets.yaml` and nothing else.

1. Open `<MAINTMODE_APP_BASE_URL>/login/recovery` — for a local trial,
   <http://localhost:3000/login/recovery>.
2. Enter the break-glass password. You are now signed in as the break-glass
   administrator.
3. In the admin UI, set up the email integration (your SMTP server).
   Invitations are delivered only by email.
4. Set up Slack or Telegram under **Administration → Integrations**, then
   create at least one notification channel under **Channels**. Every
   maintenance must notify at least one channel, so until one exists nobody can
   create a maintenance. Email does not count: it delivers invitations and
   sign-in codes only, and a channel cannot be created on it.
5. Invite yourself — your own Google address — with the admin role, and invite
   your colleagues.
6. Sign out, open the link from your invitation email, and sign in with Google.
7. *Only then* expose the instance (reverse proxy, or
   `MAINTMODE_BIND_ADDRESS=0.0.0.0`).

`compose.yaml` binds port 3000 to `127.0.0.1` by default, so a fresh
`docker compose up` is not exposed to your network until you deliberately
change that.

Keep the break-glass password afterwards. It is the way back in when the usual
sign-in methods are turned off or broken — a revoked Google client, say — and
`/login/recovery` keeps working regardless. Anyone who has it is an
administrator, so treat it like a root password; to retire break-glass
entirely, set `bootstrap/password` to `""` and restart.

`allow_open_signup: false` in `config/app.config.yaml` is what keeps the
instance invite-only. Setting it to `true` lets any Google account create an
account.

---

## Behind a reverse proxy with TLS

For anything beyond a local trial, put a reverse proxy in front and terminate
TLS there. MaintMode does not terminate TLS itself.

Four things must agree, and the login flow breaks if any one of them drifts:

1. `MAINTMODE_APP_BASE_URL` in `.env` — the public HTTPS URL
2. `app.frontend_url` in `config/app.config.yaml` — the same value
3. The Authorized redirect URI in Google Cloud Console —
   `<that URL>/auth/api/v1/login/oauth/google/callback`
4. `oauth_providers.providers.google.redirect_uri` in
   `config/app.config.yaml` — the same string as 3

Note the external form in 3 and 4: with the `/auth` prefix, as the browser sees
it. Update all four together whenever the public URL changes, then
`docker compose up -d` to apply.

The proxy sends **everything** to the gateway on port 3000; the gateway decides
what goes to the backend. Keep the gateway bound to `127.0.0.1` (the default)
when the proxy runs on the same host. The proxy reaches it over loopback;
nothing else can.

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

        # Conventional; the frontend does not depend on them (it takes its
        # public URL from MAINTMODE_APP_BASE_URL), and the gateway replaces
        # X-Forwarded-For with the address that connected to it.
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

### Rate limiting at the edge

Google sign-in puts two backend routes on your public surface. The backend
caps its sign-in routes per client (30 requests a minute,
`api_server.rate_limiter`), keyed on the address that connected to the gateway.
The gateway trusts no incoming `X-Forwarded-For` and hands the application
exactly one client address, so a client cannot fake its way into a fresh
budget — but behind your own proxy every client arrives as that proxy and
shares one bucket, unless you trust the proxy's exact address in
`gateway/Caddyfile` (`trusted_proxies static <address>/32`; the comment there
explains it). Trust only that address, never `private_ranges`: trusting your
LAN lets anyone on it choose the address they are limited by. Either way a
per-client limit at your proxy, where the real address is known, stops a flood
before it reaches the application.

Two rules make it work:

- **Key on the connection's IP, not on `X-Forwarded-For`.** A client sets that
  header to whatever it likes; a limit keyed on it is bypassed by sending a new
  "address" with every request.
- **If something sits in front of your proxy** (Cloudflare, a cloud load
  balancer), every client arrives from that front's address and shares one
  bucket. Trust it explicitly and key on the address it reports: in Caddy,
  `trusted_proxies` plus `key {client_ip}` instead of `{remote_host}`; in
  nginx, `set_real_ip_from` plus `real_ip_header X-Forwarded-For` (or the
  front's own header, such as `CF-Connecting-IP`).

nginx has `limit_req` built in:

```nginx
# In the http {} block:
limit_req_zone $binary_remote_addr zone=maintmode_auth:10m rate=10r/s;

# In the server {} block, next to location /:
location /auth/ {
    limit_req zone=maintmode_auth burst=100 nodelay;
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Caddy needs the third-party
[`rate_limit`](https://github.com/mholt/caddy-ratelimit) module, which the
official image does not include. Build Caddy with it
(`xcaddy build --with github.com/mholt/caddy-ratelimit`), then:

```caddyfile
{
	order rate_limit before reverse_proxy
}

maintmode.example.com {
	rate_limit {
		zone maintmode_auth {
			match {
				path /auth/*
			}
			key {remote_host}
			events 100
			window 10s
		}
	}
	reverse_proxy 127.0.0.1:3000
}
```

`{remote_host}` is the connection's address, not a header. If you would rather
not build Caddy, the backend's shared cap is what remains.

### Does it have to be on the internet?

No. Google never connects to your instance: the redirect to the callback is
made by the user's **browser**, so the instance only has to be reachable from
your users' browsers, which can be an internal network. What it does need:

- **Outbound HTTPS to Google** from the backend container (`accounts.google.com`,
  `oauth2.googleapis.com`, `www.googleapis.com`), to exchange the code and
  check the result.
- **A redirect URI Google accepts.** Google requires `https` unless the host is
  `localhost`, and a domain name rather than an IP address — one ending in a
  public top-level domain. An internal name like
  `maintmode.corp.example.com` that only your network resolves works; a bare
  `10.0.0.5` or a `.local` name does not.

If neither is possible, Google sign-in is not for that network. Sign-in by
emailed code works without Google, given an email server the backend can reach,
and break-glass works anywhere.

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

**Taking a newer `config/app.config.yaml`?** Compare your
`config/app.secrets.yaml` with `config/app.secrets.example.yaml` and add any
key you are missing: every `<secret:...>` the config references must exist, or
the backend refuses to start. For example, the `custom` sign-in provider
references `auth_provider/custom/client_secret`, which can stay `""` while that
provider is off.

To roll back, set the previous tag and `up -d` again — but note that a
migration applied by the newer version is *not* undone, and an older backend
may not tolerate a newer schema. Restoring from backup is the reliable path.

### Upgrading from v0.1.x

v0.1.x let the frontend run Google sign-in; newer versions run it in the
backend, behind the gateway. The configuration changed with it, and a newer
backend refuses to start on an old `config/app.config.yaml` (unknown keys are
an error, not ignored). Pull this repository's current files, then:

1. Take `compose.yaml`, `gateway/Caddyfile`, and `config/app.config.yaml` from
   this repository, and carry over your own values (`frontend_url` and
   anything you had changed). Then set
   `oauth_providers.providers.google.client_id` (the client ID, which used to
   live in the secrets file) and `oauth_providers.providers.google.redirect_uri`
   (see [step 6](#6-review-the-backend-config)).
2. In `config/app.secrets.yaml`, remove `oauth/google/client_id` and add
   `auth_provider/google/client_secret` (your Google client secret, which used
   to live in `.env`) and `bootstrap/password`
   (see `config/app.secrets.example.yaml`).
3. Remove `MAINTMODE_GOOGLE_OAUTH_CLIENT_ID` and
   `MAINTMODE_GOOGLE_OAUTH_CLIENT_SECRET` from `.env`.
4. In Google Cloud Console, add the new redirect URI
   (`<MAINTMODE_APP_BASE_URL>/auth/api/v1/login/oauth/google/callback`). Remove
   the old `/api/auth/callback/google` one once sign-in works.
5. `docker compose up -d`.

**Existing users lose their Google link.** The upgrade's migrations clear the
table that ties an account to its Google identity, so after it every user's
Google sign-in is refused as an unknown account. Accounts, roles and data are
kept; only the link is gone, and inviting an existing address is refused. To
restore access:

1. Sign in through [break-glass](#first-login) and set up the email
   integration.
2. Turn on sign-in by emailed code under the admin UI's authentication
   settings.
3. Each user signs in with an emailed code, then links Google again from their
   profile's sign-in methods.

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
- The path — it is `/auth/api/v1/login/oauth/google/callback`, with the
  `/auth` prefix
- `redirect_uri` in `config/app.config.yaml` differs from the registered one —
  it is the URI the backend actually sends

After correcting either side, `docker compose up -d`. Google's changes can take
a few minutes to propagate. This error never shows up in MaintMode's logs:
Google rejects the request on its side before your instance is involved.

### The frontend container exits immediately

Almost always a missing or invalid auth variable. `MAINTMODE_AUTH_SECRET`,
`MAINTMODE_APP_BASE_URL`, and `MAINTMODE_AUTH_PUBLIC_BASE_URL` (which
`compose.yaml` derives from `MAINTMODE_APP_BASE_URL`) are validated when the
auth module loads — before any page renders. If one is missing, nothing serves,
including `/login`. You get a container that starts and dies rather than a site
with a broken login page.

```bash
docker compose logs ui
```

The error names the offending variable. Check:

- Both are present in `.env` with no empty values
- `MAINTMODE_AUTH_SECRET` is at least 32 characters
- `MAINTMODE_APP_BASE_URL` is a full URL including the scheme
- No stray quotes around values — `.env` is not shell, so `KEY="value"` makes
  the quotes part of the value

Compose validates these upfront, so a missing one usually surfaces as an error
from `docker compose up` naming the variable, before anything starts.

### Login fails after Google accepts you

Google authenticated you, but the backend did not sign you in. Check the
backend log first — unlike `redirect_uri_mismatch`, these reach it:

```bash
docker compose logs maintmode | grep -i oauth
```

- **No invitation.** The account has no invitation and `allow_open_signup` is
  `false`, so signup is refused by design — on a fresh instance too. Sign in
  through [break-glass](#first-login) and invite the account; the invitee must
  sign in through the link in the invitation email.
- **Wrong client secret.** The code exchange with Google fails. Compare
  `auth_provider/google/client_secret` in `config/app.secrets.yaml` with the
  secret in Google Cloud Console, and `client_id` in `config/app.config.yaml`
  with the client ID.
- **Sign-in ends with a `state_invalid` error.** The short-lived sign-in
  cookie did not come back to the callback: `app.oauth_cookie_path` was
  changed, or a proxy in front rewrites paths. Restore
  `/auth/api/v1/login/oauth` and make sure the proxy passes paths through
  unchanged. (It also appears when the sign-in took longer than ten minutes;
  just try again.)

Fix and `docker compose up -d`.

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
- `has invalid keys` while reading the config — a key the backend no longer
  reads, usually a config from an older version. See
  [Upgrading from v0.1.x](#upgrading-from-v01x)

### Starting completely over

```bash
docker compose down -v
```

Deletes the containers **and the volumes**, so all data is gone. `.env` and
`config/app.secrets.yaml` survive, since they are files in your working tree.
The next `up -d` starts a fresh instance with no users: sign in through
[break-glass](#first-login) again.

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

- **Google** (`accounts.google.com`, `oauth2.googleapis.com`,
  `www.googleapis.com`), during sign-in: the code exchange and the public keys
  that verify its result. It carries nothing beyond the sign-in itself.
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
| `compose.yaml` | The stack: Postgres, Valkey, migrations, backend, frontend, gateway |
| `gateway/Caddyfile` | The gateway: routes the two sign-in routes to the backend, everything else to the frontend |
| `.env.example` | Template for `.env` — database, session, and public URL |
| `config/app.config.yaml` | Backend configuration, mounted read-only. Committed; holds no secrets |
| `config/app.secrets.example.yaml` | Template for `config/app.secrets.yaml` — signing key, encryption key, database DSN, Google client secret, break-glass password |
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
