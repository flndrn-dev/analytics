# analytics.flndrn.com — PostHog wrapper repo

Dokploy-managed deployment of a self-hosted PostHog instance for the
flndrn-network. Lives at `https://analytics.flndrn.com`.

This is a thin wrapper around upstream PostHog (vendored as a git
submodule at `posthog/`). The wrapper's job is to:

- Pin a specific PostHog version
- Provide a stripped-down "lean" docker-compose.yml suitable for a
  co-hosted hobby VPS (drops session replay, error tracking, Temporal,
  livestream, and other heavy services that aren't needed for plain
  analytics)
- Expose deployment to Dokploy via Git source

## Layout

```
analytics/
├── docker-compose.yml          # lean compose (services pruned for low-RAM hosts)
├── compose/                    # web container start scripts (mounted at /compose)
│   ├── start
│   └── wait
├── share/                      # mounted at /share inside plugins (GeoIP DB, etc.)
├── posthog/                    # PostHog source as git submodule (shallow)
├── .env.example                # template for Dokploy env tab
├── .gitignore
└── README.md
```

## What's in (kept services)

`db` `redis7` `clickhouse` `zookeeper` `kafka` `kafka-init` `objectstorage`
`web` `worker` `plugins` `ingestion-general` `capture` `asyncmigrationscheck`

Target footprint: ~3–4 GiB peak RAM.

## What's out (vs upstream hobby compose)

- `proxy` (Caddy) — replaced by Dokploy's Traefik
- `seaweedfs` — only used for session replay
- `recording-api`, `replay-capture`, `ingestion-sessionreplay` — session replay
- `ingestion-error-tracking`, `ingestion-logs`, `ingestion-traces` — error tracking, logs, traces
- `temporal`, `temporal-ui`, `temporal-admin-tools`, `temporal-django-worker`,
  `elasticsearch`, `cyclotron-janitor` — batch exports / data warehouse / scheduled HogQL
- `livestream` — real-time event UI
- `cymbal` — error stack symbolicator
- `property-defs-rs`, `feature-flags`, `hypercache-server` — Rust services that
  speed up specific paths; the Python web/plugins handle the same work natively

To re-enable any of these later, copy the matching service block back from
`posthog/docker-compose.hobby.yml`, restore its `depends_on` references in
`web` / `worker` / `plugins`, and restore any related env vars
(`SESSION_RECORDING_V2_*`, `RECORDING_API_URL`, `LIVESTREAM_HOST`).

## Deploying via Dokploy

1. **Generate secrets locally** (never share, never commit):

   ```sh
   echo "POSTHOG_SECRET=$(openssl rand -hex 32)"
   echo "ENCRYPTION_SALT_KEYS=$(openssl rand -hex 16)"
   ```

2. **Push this repo** to `github.com/flndrn-dev/analytics` (private).

3. **Dokploy → New Project → Compose**:

   - **Source:** Git → `https://github.com/flndrn-dev/analytics.git` (private; configure SSH key or PAT in Dokploy)
   - **Branch:** `main`
   - **Compose file:** `docker-compose.yml`
   - **Recurse submodules:** ✅ ON (so `posthog/` source is cloned)
   - **Environment:** paste `.env.example` and fill in real values
     (use the secrets generated above and your Resend API key)
   - **Domain:** add `analytics.flndrn.com` → service `web`, port `8000`,
     HTTPS via Let's Encrypt enabled

4. **Deploy.** First boot takes 5–10 minutes — ClickHouse and Kafka are slow
   to bootstrap and the `web` service runs `/compose/start` which waits on
   them, then runs migrations, then starts gunicorn.

5. **Verify.** `https://analytics.flndrn.com` should show PostHog's setup
   wizard. Register first account with `vancutsem@live.com` (becomes the
   org owner).

## After first boot

- **Project:** create one project named `flndrn-network` (single project
  with all apps, query cross-app journeys via `$current_url` filtering).
- **Test email delivery:** Settings → Email → "Send test email". Should
  land in your inbox within seconds.
- **Snapshot/backups:** in Dokploy, enable weekly volume backups for at
  minimum the `postgres-data` and `clickhouse-data` volumes.

## Upgrading PostHog

```sh
cd posthog
git fetch origin
git checkout <new-tag-or-commit>
cd ..
git add posthog
git commit -m "bump posthog to <new-tag>"
git push
```

Then in Dokploy → Redeploy. Read the upstream PostHog changelog before
each major bump — async migrations may need to be handled.

## Operations

- Logs: Dokploy log viewer per service, or `docker logs <container>` over SSH
- Resource check: `docker stats` over SSH if responses feel sluggish
- Restart a single service: Dokploy "Restart" button on the service
- Volumes live under Docker's volume directory on the host; back them up
  before any major change
