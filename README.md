# Homelab Config

Production-oriented Docker Compose configuration for a private homelab stack.

## Repository

```bash
git clone https://github.com/shuntps/homelab-stack.git
```

## Overview

This repository contains public-safe configuration templates for a homelab stack:

- Cloudflare Tunnel for inbound HTTP routing.
- Cloudflare DDNS for direct DNS records.
- Traefik as the internal reverse proxy.
- Authelia for authentication.
- Valkey as Redis-compatible session storage.
- Docker Socket Proxy for constrained Docker API access.
- Vaultwarden for password management.
- Pi-hole for LAN DNS filtering.
- Servarr media automation with qBittorrent, Plex, xTeVe, Prowlarr, Radarr, Sonarr, Overseerr, and Profilarr.

The real local configuration is intentionally kept out of Git. Public files use safe examples, while private files are ignored by `.gitignore`.

Infrastructure services live in `core/compose.yml`; application services that depend on the shared edge network live in `apps/compose.yml`; media automation services live in `servarr/compose.yml`. Shared values live in `env/common.env`.

## Layout

```text
env/
  common.env.example
core/
  compose.yml
  .env.example
  config/
    authelia/
      configuration.yml
      users_database.example.yml
    cloudflared/
      config.example.yml
      credentials.example.json
    traefik/
      traefik.yml
      config/
apps/
  compose.yml
  .env.example
servarr/
  compose.yml
  .env.example
```

## Private Files

Create these files locally from their examples before starting the stack:

```bash
cp env/common.env.example env/common.env
cp core/.env.example core/.env
cp apps/.env.example apps/.env
cp servarr/.env.example servarr/.env
cp core/config/cloudflared/config.example.yml core/config/cloudflared/config.yml
cp core/config/cloudflared/credentials.example.json core/config/cloudflared/credentials.json
cp core/config/authelia/users_database.example.yml core/config/authelia/users_database.yml
```

Then replace all example values with real local values.

Create `authelia/data/` in your configured appdata path and make it writable by `UID:GID` from `env/common.env`.

Public hostnames are configured with `*_SUBDOMAIN` variables plus `DOMAIN`.

Set `PIHOLE_DNS_BIND_IP` to the LAN address of the host that should answer DNS queries.

Never commit:

- `env/common.env`
- `core/.env`
- `apps/.env`
- `servarr/.env`
- `core/config/cloudflared/config.yml`
- `core/config/cloudflared/credentials.json`
- `core/config/authelia/users_database.yml`
- Authelia database, notification, secret, and runtime data files

Authelia runtime data is expected under `authelia/data/` in your configured appdata path. Vaultwarden and Pi-hole persistent data are also expected under the configured appdata path and should stay out of Git.

Pi-hole Docker settings are managed through `FTLCONF_` environment variables in `apps/compose.yml`. Generated Pi-hole files such as `pihole.toml`, `dnsmasq.conf`, local hosts, and real list exports should stay private.

Servarr downloads and media paths should stay on the same storage pool when possible. This keeps Radarr and Sonarr imports fast and allows hardlinks instead of copy-heavy moves.

Before the first Servarr start, create the Servarr config, downloads, and media directories and make them writable by `UID:GID`.

xTeVe keeps its own container user and group through `XTEVE_UID` and `XTEVE_GID`. Keep its config directories writable by those numeric IDs rather than the shared LinuxServer `PUID` and `PGID`.

Plex is not routed through Traefik. It uses host networking so LAN discovery, companion, and DLNA behavior match a native Plex install.

## Usage

Start the core stack from the repository root first, then start the app stack:

```bash
docker compose --env-file env/common.env --env-file core/.env -f core/compose.yml up -d
docker compose --env-file env/common.env --env-file apps/.env -f apps/compose.yml up -d
docker compose --env-file env/common.env --env-file servarr/.env -f servarr/compose.yml up -d
```

Validate the rendered Compose configuration:

```bash
docker compose --env-file env/common.env.example --env-file core/.env.example -f core/compose.yml config --quiet
docker compose --env-file env/common.env.example --env-file apps/.env.example -f apps/compose.yml config --quiet
docker compose --env-file env/common.env.example --env-file servarr/.env.example -f servarr/compose.yml config --quiet
docker compose --env-file env/common.env.example --env-file core/.env.example --env-file apps/.env.example --env-file servarr/.env.example -f core/compose.yml -f apps/compose.yml -f servarr/compose.yml config --quiet
```

## Validation

Authelia configuration:

```bash
docker run --rm --entrypoint authelia \
  --env-file env/common.env.example \
  --env-file core/.env.example \
  -v "$PWD/core/config/authelia/configuration.yml:/config/configuration.yml:ro" \
  -v "$PWD/core/config/authelia/users_database.example.yml:/config/users_database.yml:ro" \
  --tmpfs /data \
  -e AUTHELIA_SESSION_REDIS_HOST=redis_valkey \
  -e AUTHELIA_SESSION_REDIS_PORT=6379 \
  -e AUTHELIA_SESSION_REDIS_PASSWORD=replace_with_strong_redis_password \
  authelia/authelia:4.39.15 \
  --config /config/configuration.yml \
  --config.experimental.filters template \
  config validate
```

Cloudflared example configuration:

```bash
docker run --rm \
  -v "$PWD/core/config/cloudflared/config.example.yml:/etc/cloudflared/config.yml:ro" \
  -v "$PWD/core/config/cloudflared/credentials.example.json:/etc/cloudflared/credentials.json:ro" \
  cloudflare/cloudflared:2026.6.1 \
  tunnel --config /etc/cloudflared/config.yml ingress validate
```

## Security Notes

- Do not expose Traefik, Authelia, Redis, or Docker Socket Proxy directly on host ports.
- Keep Docker socket access behind Docker Socket Proxy.
- Bind Pi-hole DNS only to the intended LAN host address.
- Run qBittorrent behind a VPN or firewall policy before using it for untrusted peer traffic.
- Do not expose Plex through Traefik unless you intentionally design that route.
- Keep Cloudflare tunnel credentials private.
- Replace all example passwords, secrets, users, tunnel IDs, and DNS values before production use.
- Rotate any secret that was ever committed or shared.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
