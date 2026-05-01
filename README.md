# analytics.flndrn.com — Umami

Self-hosted [Umami](https://umami.is/) deployment for the flndrn-network.
Runs at <https://analytics.flndrn.com>, deployed via Dokploy as a Compose
service in the `flndrn` project, fronted by Dokploy's Traefik with
Let's Encrypt.

## Why Umami

Picked over PostHog (initial choice — see git history for the PostHog
attempt) because Umami covers the Google-Analytics-style needs of the
flndrn-network — pageviews, sessions, top pages, referrers, devices,
real-time — without PostHog's operational weight (no ClickHouse, no
Kafka, no Redis cluster). MIT licensed, easy to rebrand, ~300 MiB RAM
footprint vs PostHog's ~5 GiB peak.

## Stack

| Service | Image | Role |
|---|---|---|
| `umami` | `ghcr.io/umami-software/umami:postgresql-latest` | Next.js app, port 3000 |
| `db` | `postgres:16-alpine` | Persistent storage (volume `umami-db-data`) |

The `umami` container attaches to both the project default network (talks
to `db`) and `dokploy-network` (so Traefik can reach it). Dokploy writes
the Traefik routing rule when the domain is configured on the service.

## Deploy from scratch

1. Push this repo to `flndrn-dev/analytics` (or your fork).
2. In Dokploy, create a Compose service:
   - **Project:** `flndrn`
   - **Source:** GitHub → `flndrn-dev/analytics`, branch `main`
   - **Compose path:** `docker-compose.yml`
3. Environment tab — paste the keys from `.env.example`, then generate
   real values on a trusted machine (never paste in chat):

   ```sh
   echo "DB_PASSWORD=$(openssl rand -hex 16)"
   echo "APP_SECRET=$(openssl rand -hex 32)"
   ```

4. Add a domain on the `umami` service:
   - **Host:** `analytics.flndrn.com`
   - **Port:** `3000`
   - **HTTPS:** ✅ Let's Encrypt
5. Deploy.

First boot defaults: log in as **`admin` / `umami`** then immediately
change the password (top-right avatar → Profile → Change Password).

## Adding a site to track

Inside Umami: **Settings → Websites → Add website**. Enter the friendly
name and the public domain, save. Umami generates a `<script>` tag —
paste it into each app's HTML `<head>`:

```html
<script async defer
        src="https://analytics.flndrn.com/script.js"
        data-website-id="REPLACE_WITH_PER_SITE_ID"></script>
```

Each site gets its own `data-website-id`. Pageviews and clicks start
flowing within seconds.

The 21 flndrn-network domains we want to track:

```
flndrn.com           agecheckup.com       cyberbear.sh
krypco.eu            loowii.com           waypoints.tech
videodj.studio       mavifinans.sh        mavifinans.eu
pay.mavifinans.sh    card.mavifinans.sh   wallet.mavifinans.sh
dealdroppr.com       briven.cloud         ghostbot.dev
typr.tech            pandit.sh            handlr.sh
cyclingtravel.cc     murphus.eu           supportedby.sh
```

## Operations

- **Logs:** Dokploy → flndrn → umami → Logs
- **DB shell:** `docker exec -it <db-container> psql -U umami umami`
- **Backups:** enable Dokploy volume backup on `umami-db-data` (Postgres
  dump). Daily is plenty.
- **Upgrade:** bump the image tag in `docker-compose.yml`, push, redeploy
  in Dokploy. Umami publishes a major version every few months — read
  release notes for breaking schema changes.
- **Telemetry:** disabled via `DISABLE_TELEMETRY=1` in compose.
