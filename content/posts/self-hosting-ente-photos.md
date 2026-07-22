---
author: ["Thariq Shah"]
title: "Self-Hosting Ente Photos Behind a Cloudflare Tunnel: Preserving Request Integrity Through the Proxy Chain"
date: 2026-07-22T02:56:00+05:30
draft: false
ShowToc: true
cover:
  image: /self-hosting-ente-photos/cover/cover.png
categories: ["homelab", "self-hosting"]
tags: ["ente", "cloudflare-tunnel", "docker-compose", "minio", "s3", "caddy", "webauthn", "passkeys"]
series: ["Home labs"]
searchHidden: true
---

## Summary

This is a complete account of standing up [Ente](https://ente.com) — an end-to-end encrypted photo and video backup service — on a home server, exposed to the internet through a Cloudflare Tunnel, with mobile and web clients connecting to it from arbitrary networks. The straightforward part of this project was running `docker compose up`. The interesting part, and the reason this write-up exists, is a single architectural constraint that determines almost every other decision in the stack: **Ente's storage layer issues AWS SigV4 pre-signed URLs directly to the client, and pre-signed URL validation is byte-for-byte sensitive to header mutation along the request path.** Any reverse proxy sitting between the client and the object store is a candidate for silently breaking uploads and downloads — and Cloudflare's edge, which every hostname routed through a Cloudflare Tunnel passes through, is exactly this kind of proxy.

The rest of this document walks through the full system: the service topology, the specific header-mutation failure mode and how it was diagnosed and mitigated, the complete configuration for every component, client onboarding for mobile and web, and the administrative tooling required to run the instance day to day (including a few rough edges in Ente's own CLI that aren't documented anywhere else).

## Why Self-Host a Photo Backup Service

Consumer cloud photo storage is a recurring subscription tied to a company's roadmap, pricing decisions, and continued existence. Ente's pitch is specifically interesting for a self-hoster: the client encrypts every file, its metadata, and even the folder structure before it ever leaves the device. The server — in this case, a service we run and administer entirely ourselves — never has access to plaintext data or to the keys that would decrypt it. This changes the risk calculus for self-hosting considerably. A self-hosted Nextcloud or a self-hosted Immich instance holds your data in the clear; if the box is compromised, so is your photo library. A self-hosted Ente instance, by contrast, holds only ciphertext. This makes it one of the few self-hostable services where the operator (us) is architecturally prevented from being a liability to the thing being protected.

The trade-off is that we take on full operational responsibility: uptime, TLS, backups, and — as this document covers in depth — making sure nothing in the network path corrupts the cryptographic protocol the client and server use to talk to each other.

## System Architecture

The instance is composed of seven containers, deployed as a single Docker Compose project, sitting behind a dedicated Cloudflare Tunnel connector:

```
                                   Internet
                                       |
                                       v
                          ┌─────────────────────────┐
                          │   Cloudflare Edge (CDN)  │
                          │  TLS termination, DNS,   │
                          │  HTTP proxy layer        │
                          └────────────┬─────────────┘
                                       |  (outbound-only
                                       |   QUIC tunnel)
                          ┌────────────v─────────────┐
                          │   cloudflared connector   │
                          │  (this host, container)   │
                          └────────────┬─────────────┘
                                       |  http://caddy:80
                          ┌────────────v─────────────┐
                          │           Caddy           │
                          │   host-based routing to    │
                          │   internal services        │
                          └──┬─────────┬──────────┬───┘
                             |         |          |
                    ┌────────v──┐ ┌────v────┐ ┌───v────┐
                    │  museum   │ │   web   │ │ minio  │
                    │ (backend  │ │ (photos │ │  (S3   │
                    │   API)    │ │ + albums│ │ storage│
                    └─────┬─────┘ │  apps)  │ └────────┘
                          |       └─────────┘
                    ┌─────v─────┐
                    │ postgres  │
                    └───────────┘
```

| Service | Image | Role | Reachability |
|---|---|---|---|
| `postgres` | `postgres:15` | Application database — users, collections, file metadata, subscriptions | Container network only |
| `minio` | `minio/minio` | S3-compatible object storage for encrypted file blobs and thumbnails | Container network for administrative operations; public hostname for client-issued pre-signed URLs |
| `museum` | `ghcr.io/ente/server` | Ente's backend API (Go) | Container network for internal calls; public hostname for client API traffic |
| `web` | `ghcr.io/ente/web` | Photos web app and Public Albums web app (two apps, one image, different ports) | Public hostnames |
| `caddy` | `caddy:2` | Internal reverse proxy; the single point that receives all tunnel-routed traffic and dispatches it by Host header | Container network only, reached by the tunnel connector |
| `cloudflared` | `cloudflare/cloudflared` | Outbound-only tunnel connector; the only component with an internet-facing role | N/A (makes outbound connections only) |

Four public hostnames are involved, each carrying distinct traffic characteristics that matter later in this document:

- `photos.example.com` → the Photos web app
- `albums.example.com` → the Public Albums web app (used for shared-link viewing by people without an account)
- `api.example.com` → museum, the backend API
- `storage.example.com` → MinIO, the object store

## The Core Challenge: Header Integrity Across a Multi-Hop Proxy Chain

### How Ente's clients talk to storage

Most self-hosted apps that sit behind a reverse proxy only need the proxy to forward requests and responses byte-for-byte-ish — a dropped or renamed header rarely matters for a typical CRUD API. Object storage is different. When a client needs to upload or download a file, museum does not proxy the file bytes itself. Instead, museum generates a **pre-signed URL**: a URL with an embedded cryptographic signature (AWS Signature Version 4) that grants time-limited, scoped access to a specific object, without requiring the client to hold any storage credentials. The client then makes an HTTP request directly to that URL — which means the request goes to MinIO, not to museum, and it goes through whatever sits between the client and MinIO.

The SigV4 signature is computed over a canonical form of the request that includes select headers. If any component in the path between the client and MinIO adds, removes, or rewrites a header that's part of that canonical form — or changes the request body encoding in a way that shifts what the server sees — the signature MinIO independently recomputes on arrival will not match the one embedded in the URL, and the request is rejected. This is by design: it's exactly the mechanism that makes pre-signed URLs safe to hand to an untrusted client in the first place. But it means any proxy in the path has to be transparent in a very specific, narrow sense.

### Why "Cloudflare Tunnel" doesn't sidestep this

It's tempting to assume that routing traffic through a Cloudflare Tunnel avoids Cloudflare's usual proxying behavior, since a tunnel is often pitched as a way to expose a service "directly," without the traditional orange-cloud DNS proxy. This isn't accurate. A Cloudflare Tunnel's Public Hostname feature works by creating a CNAME record that resolves into Cloudflare's own network, and inbound HTTP requests for that hostname are received and processed by the same edge infrastructure that handles ordinary proxied DNS records — the tunnel only replaces how the connection reaches the origin (an outbound connection initiated by the connector, instead of an open inbound port), not how the edge itself handles the HTTP request.

Concretely, Cloudflare's edge:

- Normalizes and can rewrite the `Accept-Encoding` header as part of negotiating compression with the origin.
- Can inject additional response headers (e.g. `X-Content-Type-Options` and other security headers) via "Managed Transforms," if enabled on the zone.
- Applies caching logic by default, which interacts poorly with signed, time-limited URLs.

None of this is unique to MinIO — it's a documented, recurring issue for anyone running S3-compatible storage behind Cloudflare. The `Accept-Encoding` mutation in particular breaks any AWS SDK-based client that includes that header in its signature calculation (this affects tools like `rclone` against Cloudflare-proxied buckets as well, not just browser or mobile clients talking to a self-hosted museum).

### The constraint this creates for our topology

There's a second, more structural issue, independent of Cloudflare specifically. The pre-signed URL is generated by museum and consumed by the client — which means the *hostname baked into that URL* has to be resolvable and reachable by whatever machine the client is running on. If museum is configured with an internal-only storage endpoint (a Docker service name, a LAN IP), that works for local testing but breaks the moment the client is anywhere other than the same Docker network — which, for a photo backup app whose entire point is availability from a phone on a cellular network, is the normal case, not an edge case.

This is a real, currently open limitation in Ente's self-hosting story: museum uses a single endpoint configuration value both for its own internal calls to the storage bucket and for the hostname embedded in client-facing pre-signed URLs. There is no clean way to tell museum "use the internal address for yourself, but sign URLs with the external one." (See [ente-io/ente#5689](https://github.com/ente-io/ente/issues/5689) for the same limitation reported independently in a Kubernetes context.) The practical consequence: the storage endpoint has to be public, museum's own traffic to storage round-trips out through the tunnel and back in, and the header-integrity problem above has to be solved for real, not architected around.

## Design Decisions

With the constraint established, four decisions shape the rest of the implementation.

**1. A locally-managed Cloudflare Tunnel, not a dashboard-managed one.** Cloudflare Tunnels can be created two ways: through the Zero Trust dashboard, which issues a single run token and configures routing entirely through the UI, or through the `cloudflared` CLI, which produces a tunnel ID, a credentials file, and a `config.yml` that lives with the rest of the deployment's configuration. Both support per-hostname origin request overrides (the dashboard exposes them under "Additional application settings" on each Public Hostname route), so this isn't strictly a functional requirement — but keeping the ingress rules, including the explicit `httpHostHeader` override for each hostname, as version-controllable YAML alongside the rest of the compose project keeps the whole deployment reproducible from one directory, rather than split between a config file and a web UI.

**2. A local reverse proxy (Caddy) between the tunnel connector and every backend service.** The tunnel connector could point directly at each container's port. Routing everything through a single internal Caddy instance instead means Host-header-based dispatch happens in one well-understood, well-tested piece of software, and every backend service only ever sees traffic from one internal caller with a consistent request shape — regardless of which public hostname it arrived on. This also happens to be Ente's own recommended pattern for self-hosters running behind any reverse proxy.

**3. Accept that the storage hostname round-trips through Cloudflare, and mitigate at the edge.** Given the constraint above, the storage endpoint has to be public. The mitigation for the `Accept-Encoding` and header-injection issues is applied where the problem originates — Cloudflare's zone-level configuration — rather than attempted anywhere downstream, where it can't actually be fixed.

**4. Everything that doesn't need to leave the Docker network, doesn't.** `museum → postgres`, and MinIO's own bucket-provisioning calls at startup, all use Docker service names on the internal compose network. The only traffic that touches the public hostnames is traffic that is, by the nature of the client, coming from outside the network in the first place.

## Implementation

### Directory layout

```
~/docker/ente/
├── compose.yaml
├── .env                      # PUID/PGID/TZ, Postgres and MinIO credentials
├── museum.yaml               # Ente backend configuration
├── data/
│   ├── postgres/
│   ├── minio/
│   └── museum/
├── caddy/
│   └── Caddyfile
└── cloudflared/
    ├── cert.pem              # origin certificate, from `cloudflared tunnel login`
    ├── <tunnel-id>.json      # tunnel credentials, from `cloudflared tunnel create`
    └── config.yml            # ingress rules
```

### Generating credentials

Every secret is generated locally rather than left as a placeholder, following the same pattern Ente's own quickstart script uses:

```sh
gen_password () { head -c 21 /dev/urandom | base64 | tr -d '\n'; }
gen_key ()      { head -c 32 /dev/urandom | base64 | tr -d '\n'; }
gen_hash ()     { head -c 64 /dev/urandom | base64 | tr -d '\n'; }
gen_jwt_secret (){ head -c 32 /dev/urandom | base64 | tr -d '\n' | tr '+/' '-_'; }

pg_pass=$(gen_password)
minio_user="minio-user-$(head -c 6 /dev/urandom | base64 | tr -dc 'a-zA-Z0-9')"
minio_pass=$(gen_password)
museum_key=$(gen_key)
museum_hash=$(gen_hash)
museum_jwt_secret=$(gen_jwt_secret)
```

These feed into `.env` (consumed by `compose.yaml`) and directly into `museum.yaml` (which is not variable-substituted by Compose, since it's mounted as a file into the `museum` container rather than read by Compose itself).

`.env`:

```env
PUID=1000
PGID=1000
TZ=America/Los_Angeles
PG_PASS=<generated>
MINIO_USER=<generated>
MINIO_PASS=<generated>
```

### compose.yaml

```yaml
services:
  postgres:
    image: postgres:15
    container_name: ente-postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=pguser
      - POSTGRES_PASSWORD=${PG_PASS}
      - POSTGRES_DB=ente_db
    healthcheck:
      test: pg_isready -q -d ente_db -U pguser
      start_period: 40s
      start_interval: 1s
    volumes:
      - ./data/postgres:/var/lib/postgresql/data

  minio:
    image: minio/minio
    container_name: ente-minio
    restart: unless-stopped
    environment:
      - MINIO_ROOT_USER=${MINIO_USER}
      - MINIO_ROOT_PASSWORD=${MINIO_PASS}
    command: server /data --address ":3200" --console-address ":3201"
    volumes:
      - ./data/minio:/data
    post_start:
      - command: |
          sh -c '
          while ! mc alias set h0 http://minio:3200 ${MINIO_USER} ${MINIO_PASS} 2>/dev/null
          do
            echo "Waiting for minio..."
            sleep 0.5
          done
          cd /data
          mc mb -p ente-photos
          '

  museum:
    image: ghcr.io/ente/server
    container_name: ente-museum
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./museum.yaml:/museum.yaml:ro
      - ./data/museum:/data:ro
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/ping"]
      interval: 60s
      timeout: 5s
      retries: 3
      start_period: 120s

  web:
    image: ghcr.io/ente/web
    container_name: ente-web
    restart: unless-stopped
    environment:
      ENTE_API_ORIGIN: https://api.example.com

  caddy:
    image: caddy:2
    container_name: ente-caddy
    restart: unless-stopped
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
    depends_on:
      - museum
      - web
      - minio

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: ente-cloudflared
    restart: unless-stopped
    command: tunnel --config /etc/cloudflared/config.yml run
    volumes:
      - ./cloudflared:/etc/cloudflared:ro
    depends_on:
      - caddy
```

A few things worth calling out that aren't obvious from reading the file alone:

- No service other than `caddy` needs a host port published. The entire public surface is reached through the tunnel; the tunnel reaches everything through Caddy; Caddy reaches everything by container name. There's no reason for `museum`, `minio`, or `web` to bind anything on the Docker host itself in normal operation — port publishing is reintroduced only temporarily, for administrative CLI access (covered later).
- The `web` image serves multiple distinct apps (Photos, Public Albums, Accounts, Auth, and others) from different ports inside the same container. Only the ports actually in use need a corresponding Caddy route; there's no need to enumerate or expose the rest.
- `museum`'s `/data` mount is unrelated to MinIO's storage — it's a local scratch/cache directory museum expects, distinct from the object storage backend, and is fine to leave as an empty read-only bind mount unless a specific feature requires otherwise.

### museum.yaml

This is the file that decides how museum talks to storage, and it's where the architectural constraint discussed earlier becomes concrete configuration:

```yaml
db:
    host: postgres
    port: 5432
    name: ente_db
    user: pguser
    password: <PG_PASS>

s3:
    are_local_buckets: false
    use_path_style_urls: true
    b2-eu-cen:
        key: <MINIO_USER>
        secret: <MINIO_PASS>
        endpoint: storage.example.com
        region: eu-central-2
        bucket: ente-photos

apps:
    photos: https://photos.example.com
    public-albums: https://albums.example.com

internal:
    trusted-client-ip-header: CF-Connecting-IP

key:
    encryption: <museum_key>
    hash: <museum_hash>

jwt:
    secret: <museum_jwt_secret>
```

Two flags in the `s3` block deserve explanation, because their names undersell what they actually do:

- **`are_local_buckets`** is not really about whether the bucket is "local" in a physical sense — it's a bundle of workarounds for talking to an unencrypted, locally-reachable MinIO instance: it disables SSL for the connection, forces path-style addressing, and skips storage-class headers MinIO doesn't support. Left `true` (the default when pointing at a bare `minio:3200` container address), museum generates pre-signed URLs with an `http://` scheme — which is wrong for a public endpoint sitting behind Cloudflare's TLS termination. Setting it `false` is what makes museum sign URLs with `https://`, matching what the client will actually receive.
- **`use_path_style_urls`** controls whether the bucket name appears in the path (`https://storage.example.com/ente-photos/...`) or as a subdomain (`https://ente-photos.storage.example.com/...`). Since there is exactly one public hostname in front of MinIO here, not a wildcard subdomain per bucket, path-style is the only option that works without additional DNS and Caddy configuration.

The `internal.trusted-client-ip-header` setting is a small piece of header hygiene that matters specifically because of the proxy chain: without it, museum falls back to checking `X-Forwarded-For`, `X-Real-IP`, and the raw connection address, in that order, none of which reflect the real client IP once Caddy is in front of every request. Cloudflare's edge sets `CF-Connecting-IP` to the actual client IP on every request it forwards, regardless of how many hops happen after that inside the tunnel, so pointing museum at that header directly is more reliable than hoping the intermediate hops preserve `X-Forwarded-For` correctly.

### Caddyfile

```
{
	auto_https off
}

http://photos.example.com {
	reverse_proxy web:3000
}

http://albums.example.com {
	reverse_proxy web:3002
}

http://api.example.com {
	reverse_proxy museum:8080
}

http://storage.example.com {
	reverse_proxy minio:3200
}
```

`auto_https off` is not optional here. Caddy's default behavior is to provision a Let's Encrypt certificate for every hostname in its config, using an HTTP-01 or TLS-ALPN challenge that requires the certificate authority to reach Caddy directly over the public internet. Caddy in this deployment is never reachable from the public internet — only `cloudflared`, on the internal Docker network, ever talks to it — so the challenge would fail indefinitely and Caddy would retry it forever in the background for no benefit. TLS for public traffic is entirely Cloudflare's responsibility here; Caddy only ever needs to speak plain HTTP to the tunnel connector sitting next to it on the same Docker network.

### The Cloudflare Tunnel

A locally-managed tunnel is created once, interactively, from the host:

```sh
docker run --rm \
  -v ~/docker/ente/cloudflared:/home/nonroot/.cloudflared \
  cloudflare/cloudflared:latest tunnel login
```

This prints a URL. Opening it in a browser and authorizing against the target Cloudflare account writes an origin certificate (`cert.pem`) back into the mounted directory — this is the one interactive step in the entire setup that can't be scripted, since it requires an actual human logging into Cloudflare.

One operational detail worth flagging explicitly: the `cloudflare/cloudflared` image runs as a non-root user (UID `65532`, conventionally named `nonroot` in distroless-style images). If the host directory being mounted in is owned by a different UID — which it will be, by default, if created by whatever user is running Docker — the container will fail to write `cert.pem` with a permission error. The fix is a one-time `chown` of the host directory to `65532:65532` before running any `cloudflared` command against it, rather than trying to override the container's user (overriding to an arbitrary UID with no corresponding entry in the image's `/etc/passwd` breaks `$HOME` resolution inside the container in its own way, trading one permission error for a different one).

With the certificate in place, the tunnel itself is created, which produces a tunnel ID and a credentials JSON file:

```sh
docker run --rm \
  -v ~/docker/ente/cloudflared:/home/nonroot/.cloudflared \
  cloudflare/cloudflared:latest tunnel create ente-homelab
```

`config.yml`:

```yaml
tunnel: <tunnel-id>
credentials-file: /etc/cloudflared/<tunnel-id>.json

ingress:
  - hostname: photos.example.com
    service: http://caddy:80
    originRequest:
      httpHostHeader: photos.example.com
  - hostname: albums.example.com
    service: http://caddy:80
    originRequest:
      httpHostHeader: albums.example.com
  - hostname: api.example.com
    service: http://caddy:80
    originRequest:
      httpHostHeader: api.example.com
  - hostname: storage.example.com
    service: http://caddy:80
    originRequest:
      httpHostHeader: storage.example.com
  - service: http_status:404
```

The `credentials-file` path is where the most common configuration mistake in this setup happens, and it's worth being explicit about why: the path used during `tunnel login` / `tunnel create` (`/home/nonroot/.cloudflared/...`, matching how those one-off commands were invoked) has no relationship to the mount point used by the long-running service in `compose.yaml` (`/etc/cloudflared`, as declared above). The credentials file physically exists at the same place on the host disk either way — but the path *inside the container* that `config.yml` references has to match whatever mount point that particular container invocation uses. Copy the ingress config above verbatim and this isn't an issue, but if the credentials-file path is ever adjusted, it needs to track the compose service's mount point, not whatever path a one-off CLI invocation happened to use.

Finally, DNS records for all four hostnames are created (as CNAMEs pointing at `<tunnel-id>.cfargotunnel.com`) either automatically by adding each hostname as a Public Hostname route in the Zero Trust dashboard, or explicitly via:

```sh
docker run --rm \
  -v ~/docker/ente/cloudflared:/home/nonroot/.cloudflared \
  cloudflare/cloudflared:latest tunnel route dns ente-homelab photos.example.com
# ...repeated for each hostname
```

### Mitigating the storage-hostname header issue at the Cloudflare zone level

This is the step that directly addresses the failure mode described earlier, and it happens entirely in the Cloudflare dashboard rather than in any file in this deployment:

1. **Cache Rule**, scoped to `storage.example.com`: Cache Level set to Bypass. Pre-signed URLs are time-limited and single-purpose by design; caching a response against one is both incorrect (the URL may not be valid on a subsequent request) and unnecessary.
2. **Configuration Rule**, scoped to the same hostname: Auto Minify, Rocket Loader, and the "Add security headers" managed transform all disabled. These are all response-mutating features aimed at HTML-serving origins; applied to binary object storage responses, they're pure downside.

Neither rule fully guarantees the `Accept-Encoding` normalization issue is eliminated for every possible client SDK — this is genuinely a case where "verify, don't assume" applies. The definitive test is an actual upload and download performed from a network that isn't the same LAN as the server (a phone on cellular data, not home Wi-Fi), since that's the only way to exercise the exact code path — client → Cloudflare edge → tunnel → Caddy → MinIO — that a same-network test would silently skip past.

If that end-to-end test still surfaces signature-mismatch errors in museum's logs after applying both rules, the documented fallback is to make the storage hostname **DNS-only** (grey-clouded) in Cloudflare — a plain A record pointing at the server's public IP, with the port forwarded on the router and Caddy handling its own certificate — which removes Cloudflare's HTTP-layer processing from that one hostname entirely, while `photos`, `albums`, and `api` remain behind the tunnel. This trades away Cloudflare's DDoS protection and IP-hiding for that specific hostname, which is a reasonable trade for a hostname that only ever serves opaque, pre-authorized binary blobs.

## Deployment and Verification

```sh
cd ~/docker/ente
docker compose pull
docker compose up -d
docker compose ps
```

Expected state: `postgres` and `museum` report healthy via their configured healthchecks; the rest report running. A quick verification pass:

```sh
# From inside the Docker network, bypassing the tunnel entirely:
docker exec ente-museum wget -qO- http://localhost:8080/ping

# From the public internet, through the full chain:
curl -s https://api.example.com/ping
# {"id":"...","message":"pong"}

curl -s -o /dev/null -w '%{http_code}\n' https://photos.example.com
curl -s -o /dev/null -w '%{http_code}\n' https://albums.example.com
curl -s -o /dev/null -w '%{http_code}\n' https://storage.example.com
# A 403 here from MinIO on an unauthenticated request is the *correct* result —
# it confirms MinIO is reachable through the full chain and is correctly
# rejecting an unsigned request, rather than the request failing to arrive at all.
```

## Connecting Client Applications

Every Ente client — the Android app, the iOS app, and the web apps — supports pointing at a custom server instead of Ente's own hosted service. In the mobile apps, this is exposed as a "Sign in" flow option (tapping the logo a set number of times, or a dedicated "custom server" entry point depending on app version) that prompts for an API endpoint. That endpoint is the museum public hostname configured above — `https://api.example.com` — not the Photos web app hostname. Once set, the mobile app performs its own email-verification signup flow against that endpoint exactly as it would against Ente's production service; there's no difference in client-side behavior between talking to a self-hosted instance and talking to the hosted one, which is the entire point of the client/server split.

The web apps don't need this step at deployment time, since `ENTE_API_ORIGIN` is baked in as a build/runtime environment variable in `compose.yaml`, pointing the Photos and Public Albums apps at the correct API origin automatically.

## Administrative Operations

Running the instance day to day surfaces a few things that aren't covered in Ente's own self-hosting documentation.

### Account verification without outbound email

Ente's signup and login flow sends a one-time verification code by email. Self-hosted museum, with no SMTP relay configured, does not silently fail this — it logs the code instead of attempting to send it:

```
INFO email.go:111 Send Skipping sending email to user@example.com: Verification code: 483250
```

The web app's copy ("we've sent you a code") is generic client-side text and doesn't reflect whether an email actually went anywhere; on an instance with no SMTP configured, nothing does, and no third-party mail service is ever contacted. For a single-operator instance, reading the code out of `docker logs ente-museum` after triggering it from the client is a perfectly workable flow. For anything more frequent, `internal.hardcoded-ott` in `museum.yaml` can pin a fixed, known code to a specific address or an entire domain suffix, removing the log-reading step entirely — documented directly in Ente's own `local.yaml` configuration reference, under the comment "useful for logging in when developing."

### The admin model

There is no separate admin account or admin login. Admin privileges are a property assigned to a regular user's ID, configured via `internal.admin` (single admin) or `internal.admins` (a list) in `museum.yaml`. If neither is set, museum falls back to treating the very first user ever created on the instance as the admin — which, for a freshly deployed single-operator instance, means the first signup is the admin by default, with no further configuration required.

Registration can also be closed off entirely once the intended set of users has signed up, via `internal.disable-registration: true` — subsequent signup attempts get an unauthorized error at the client, while existing users continue to log in normally.

### Storage quotas and the CLI

Every account on a self-hosted instance still defaults to Ente's "Free" plan storage limit, since museum's subscription logic has no special-cased "this is self-hosted, grant unlimited storage" behavior — the plan and quota system exists independently of whether real billing (Stripe) is configured. Raising a user's limit is an explicitly supported admin operation, via Ente's separate CLI tool rather than through `museum.yaml` or the API directly:

```sh
ente admin update-subscription -a admin@example.com -u user@example.com --no-limit=true
```

Reading the CLI's own source is more informative than its `--help` output: `--no-limit` does exactly one thing — it writes a literal 100TB value into the target user's storage quota and pushes the subscription's expiry out by 100 years. There is no separate "unlimited" sentinel value or display mode anywhere in the data model; the web and mobile clients always render whatever numeric quota is stored, and a sufficiently large number is Ente's version of "unlimited." (A quota this large can render as an unexpected-looking figure in client UI — 100 TiB formatted through a decimal-TB label shows as "102.4 TB," for instance — which is a display quirk in unit conversion, not a sign that a smaller limit was actually applied. The number in the database is the authority; if in doubt, that's what to check.)

Two setup issues are worth documenting because they're not obvious from the CLI's error messages:

**The CLI defaults to storing its session in the OS keyring**, which doesn't exist in a headless SSH session. The error (`error getting password from keyring: failed to unlock correct collection`) is resolved by setting `ENTE_CLI_SECRETS_PATH` to a file path before running any `ente` command — the CLI then manages its own encrypted local credential store at that path instead of depending on a desktop secret-service daemon.

**The CLI's `~/.ente/config.yaml` endpoint has to point at wherever museum is actually reachable from the CLI's own perspective**, which is not necessarily `localhost:8080` by default — if that port is already bound by something else on the host (a torrent client's web UI is a common collision on a homelab box that runs several services), the CLI will connect successfully to the *wrong* service, get back an unexpected response, and fail with an opaque error rather than a connection refusal. Since museum has no port published to the host in this deployment's normal operating mode (traffic reaches it only through Caddy), a temporary, loopback-only port mapping (`127.0.0.1:18080:8080` in `compose.yaml`, never exposed beyond the host itself) is the cleanest way to give the CLI something to talk to without touching the tunnel-facing configuration at all.

Direct database edits to the `subscriptions` table can achieve the same result as `--no-limit` without needing the target user's password (the CLI's `account add` step authenticates as a real user, including their real credentials) — useful in situations where an admin needs to grant storage to an account whose password they don't have and shouldn't ask for, but the CLI path is the supported one and is what to reach for by default.

## Two-Factor Authentication and a Hidden Dependency on Time Synchronization

Enabling TOTP-based two-factor authentication on the Ente account itself — as distinct from Ente Auth, the separate authenticator app in Ente's product line — is a standard, protocol-level TOTP enrollment: Settings → Security in the Photos app produces a QR code and a fallback secret, either of which can be scanned or entered into any RFC 6238-compliant authenticator (Google Authenticator, Microsoft Authenticator, 1Password, Bitwarden, or Ente Auth itself). The one caveat specific to Ente's product design is that if the account being protected is the same account used to log into Ente Auth, storing that account's own 2FA secret inside Ente Auth creates a circular dependency — Auth's own access is gated by the same login, so the code needed to get back in can end up locked inside the app that required it in the first place. Using a separate authenticator for the Ente account's own secret avoids this entirely.

On this instance, enabling 2FA initially failed in a way that pointed away from the obvious suspects: every code produced by two independent, unrelated authenticator apps (Google Authenticator and Microsoft Authenticator, on different underlying implementations) was rejected as incorrect. A single app producing bad codes usually means a mis-scanned secret. Two independent apps failing identically means the problem isn't the secret, or the client — it means whatever both clients are being checked against is wrong.

TOTP is, by construction, not really "time-based" so much as "clock-agreement-based": the client and the server independently compute a code from the shared secret and the current Unix time, divided into fixed windows (30 seconds, standardly), and the two computations only produce the same code if both parties' clocks agree closely enough to land in the same window. The client side of that agreement is generally solid — phones sync their clocks aggressively by default. The server side is not guaranteed at all, and is worth checking explicitly before assuming a code-generation bug:

```sh
timedatectl status
```

```
System clock synchronized: no
              NTP service: n/a
```

Two flags, together, are the signature of this failure mode: `System clock synchronized: no` means the kernel doesn't believe its own clock is accurate, and `NTP service: n/a` means there is nothing running that could correct it — the host had no `systemd-timesyncd`, no `chronyd`, nothing. Without an active time-sync daemon, a system clock only ever drifts, gradually, for as long as it runs, with nothing pulling it back — and the moment a corrective sync did run, the actual drift became visible:

```
# before enabling NTP:
Local time: Wed 2026-07-22 07:34:14 IST

# after `apt-get install systemd-timesyncd && systemctl enable --now systemd-timesyncd`:
Local time: Wed 2026-07-22 02:11:23 IST
```

Just over five hours of accumulated drift, on a machine that otherwise looked and behaved completely normally — serving media, running background jobs, answering health checks — because none of that other traffic cares what time it is in any way that would surface a five-hour error as a visible symptom. TOTP validation is one of the few things a server does that fails hard and immediately the moment the clock is wrong, which is what made this instructive: the 2FA setup flow didn't malfunction, it worked exactly as designed and correctly rejected codes computed against a clock that was, from its perspective, telling the truth about a wrong time.

The fix doesn't require touching Ente at all — it's entirely a host-level concern, and once corrected, applies to everything on the box, not just museum:

```sh
sudo apt-get install -y systemd-timesyncd
sudo systemctl enable --now systemd-timesyncd
sudo timedatectl set-ntp true
```

No container restart is required afterward. Museum reads system time live, per request, rather than caching it at process start, so the correction takes effect for the very next login attempt.

The broader lesson generalizes past TOTP specifically: any self-hosted service whose correctness depends on clock agreement with an external party — TOTP, JWT expiry validation, TLS certificate validity windows, cron-scheduled jobs coordinating with anything off-box — is quietly relying on the host having a working NTP client. It's cheap enough to verify unconditionally, on any freshly provisioned box, before it becomes the answer to a confusing bug report: `timedatectl status`, check for `synchronized: yes`, move on.

## Addendum: Passkey Support Requires a Fifth Public Hostname

Some months into running this instance, adding a passkey from the Android app — Settings → Security → Passkey, meant to sit alongside the TOTP 2FA described above and let a fingerprint or Face ID unlock stand in for typing a code — kept redirecting to `accounts.ente.com`, Ente's own hosted service, instead of anything resolving to this instance. Worth being precise about what this feature actually is before chasing the bug: on Ente, a passkey is a **second factor**, not a password replacement. It doesn't change how the account's email+password login works; it gives WebAuthn (fingerprint, Face ID, a hardware key) as an alternative to typing a TOTP code at that second step.

### Why a fifth hostname, and not just a flag

Ente's client apps ship several distinct frontends from the same `ghcr.io/ente/web` image — Photos and Public Albums, already routed in this deployment, plus Accounts, Auth, Cast, and a handful of others, each on its own internal port. Passkey enrollment specifically only exists in the Accounts frontend; if it isn't deployed and reachable, there is no code path that can create or verify a passkey at all, on any client, regardless of what else is configured correctly. `museum.yaml` has an `apps.accounts` key for exactly this, defaulting to `https://accounts.ente.com` — Ente's own hosted instance — which is what the mobile app was actually reaching. Nothing was misconfigured so much as never configured; the key was simply absent.

The fix doesn't require a new container. The `web` image is already running the Accounts frontend internally, unused, on port 3001 — it just needed the same three-piece wiring already used for the other hostnames:

`caddy/Caddyfile`, one more block alongside the existing four:

```
http://accounts.example.com {
	reverse_proxy web:3001
}
```

`cloudflared/config.yml`, one more ingress rule, and a corresponding Public Hostname / DNS CNAME pointed at the tunnel, same pattern as every other hostname in this deployment:

```yaml
  - hostname: accounts.example.com
    service: http://caddy:80
    originRequest:
      httpHostHeader: accounts.example.com
```

`museum.yaml`, telling museum to actually send clients to the new hostname instead of its own default:

```yaml
apps:
    photos: https://photos.example.com
    public-albums: https://albums.example.com
    accounts: https://accounts.example.com
```

A `docker compose up -d` doesn't pick any of this up on its own — none of these three files are read by Compose itself, they're bind-mounted straight into their respective containers, so Compose sees no service-definition change and leaves all three containers running untouched. Caddy, cloudflared, and museum each needed an explicit `docker restart` before the new route existed anywhere. After that, the mobile app's passkey button correctly landed on the self-hosted Accounts page instead of Ente's own service — and enrollment still failed, silently, on every attempt.

### The second bug, underneath the first

"Silently" is doing real work in that sentence, and it's what made this one worth writing up separately from the routing fix above. The Accounts page loaded, the fingerprint prompt appeared, and then — nothing. No error toast, no red text, just a spinner that gave up. `docker logs` on the Accounts frontend showed the page and its assets serving normally. The interesting log was museum's:

```
req_method=POST req_uri=/passkeys/registration/begin status_code=200 ...
```

Called twice (one retry), both times a clean `200`. No corresponding `/passkeys/registration/finish` ever showed up — not a failed one, not one at all. The request that would carry the actual signed credential back to the server never left the browser. That split is the tell: the server handed back a valid registration challenge; whatever happened next happened entirely inside the phone's browser, before it ever tried to talk to museum again.

WebAuthn — the browser API behind every passkey prompt — binds every credential it creates to a **Relying Party ID**: a domain the browser will enforce for the lifetime of that credential, checked against the page's actual origin at creation time and again at every subsequent use. The server includes the RP ID it expects in the registration challenge; if that ID isn't equal to (or a registrable parent of) the page's real origin, the browser refuses to create the credential at all, client-side, before any network call reports the failure back — which is exactly the silent, non-erroring dead end this instance was hitting.

`museum.yaml` had no `webauthn` section anywhere. Ente's own default configuration, absent any override, is:

```yaml
webauthn:
    rpid: localhost
    rporigins:
        - http://localhost:3001
```

Sensible for a developer running the stack unmodified on their own machine; silently wrong for a self-hosted instance with the Accounts app on its own real hostname. Every registration challenge museum issued was scoped to `localhost`, which the phone's browser — correctly seeing `accounts.example.com` in its address bar — rejected outright. The fix is the same file, one more section:

```yaml
webauthn:
    rpid: accounts.example.com
    rporigins:
        - https://accounts.example.com
```

`rpid` is the domain the credential is scoped to; `rporigins` is the explicit whitelist of full origins (scheme included) the server will accept a WebAuthn ceremony from. Both need to agree with wherever the Accounts app is actually being served — not the Photos app's hostname, not the API hostname, the Accounts one specifically, since that's the origin the browser's `navigator.credentials.create()` call is actually running from. A `docker restart` on museum alone was enough to pick this up; enrollment succeeded on the next attempt, first try.

The general shape of this bug is worth remembering past Ente specifically: any self-hosted service that does its own WebAuthn/passkey handling, rather than delegating to an external identity provider, has some equivalent of an RP ID setting, and it defaults to whatever's convenient for local development — `localhost`, a bare IP, a placeholder domain. It fails exactly like this one did: no error surfaced to the user, no error logged past the initial challenge, just a request that mysteriously never gets past the browser. If a WebAuthn ceremony issues a valid-looking first request and then goes quiet, the RP ID configuration is the first thing worth checking, before assuming the bug is anywhere in the network path that got the request there in the first place.

## Operational Considerations

**Backup.** Every object MinIO stores is already client-side encrypted ciphertext — Ente's entire security model depends on the server never seeing plaintext. This means the object store can be mirrored to an untrusted, even fully public, storage destination without weakening the system's privacy guarantees at all. A scheduled `mc mirror` (or `rclone sync`) job from the local MinIO bucket to an off-site S3-compatible destination, alongside a nightly `pg_dump` of Postgres (which holds the wrapped per-file keys, and is required alongside the blobs, not optional), is a complete backup strategy that doesn't require any additional encryption step on top.

**Migration.** Every piece of persistent state in this deployment lives under one directory (`~/docker/ente`, including the bind-mounted Postgres and MinIO data). Moving to new hardware is `rsync` that directory to the new host and `docker compose up -d` there. Because the tunnel connector is an outbound-only process authenticated by a credentials file, the exact same tunnel — no DNS changes, no dashboard reconfiguration — can run on the new host immediately after the move.

**Storage growth.** Object storage for a photo/video library grows indefinitely and without an obvious ceiling; it's worth deliberately choosing a data volume location with headroom to spare from day one; retrofitting this after the fact means either resizing a filesystem live or migrating potentially hundreds of gigabytes of already-uploaded object data.

## Closing Notes

The bulk of the effort in this project wasn't wiring together seven containers — Docker Compose makes that close to mechanical. It was recognizing, early, that one specific architectural property of the application (client-issued pre-signed URLs) interacted with one specific, easy-to-overlook property of the network path (Cloudflare's edge always sits in the HTTP path, tunnel or not) to produce a failure mode that wouldn't show up in a same-network smoke test, and would only surface once a real client, on a real remote network, tried to actually use the thing. Solving for that up front — rather than discovering it after onboarding real users — is the difference between a home lab project that works in a demo and one that works when someone's phone tries to back up their camera roll from a train.
