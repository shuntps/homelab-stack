# Homelab Config

Production-oriented Docker Compose configuration for a private homelab core stack.

## Repository

```bash
git clone https://github.com/shuntps/homelab-stack.git
```

## Overview

This repository contains the public-safe configuration templates for a homelab core stack:

- Cloudflare Tunnel for inbound HTTP routing.
- Cloudflare DDNS for direct DNS records.
- Traefik as the internal reverse proxy.
- Authelia for authentication.
- Valkey as Redis-compatible session storage.
- Docker Socket Proxy for constrained Docker API access.

The real local configuration is intentionally kept out of Git. Public files use safe examples, while private files are ignored by `.gitignore`.

## Layout

```text
unraid/
  compose.yml
  .env.example
  authelia/
    configuration.yml
    users_database.example.yml
  cloudflared/
    config.example.yml
    credentials.example.json
  traefik/
    traefik.yml
    config/
```

## Private Files

Create these files locally from their examples before starting the stack:

```bash
cd unraid
cp .env.example .env
cp cloudflared/config.example.yml cloudflared/config.yml
cp cloudflared/credentials.example.json cloudflared/credentials.json
cp authelia/users_database.example.yml authelia/users_database.yml
```

Then replace all example values with real local values.

Create `authelia/data/` in your configured appdata path and make it writable by `UID:GID` from `.env`.

Never commit:

- `unraid/.env`
- `unraid/cloudflared/config.yml`
- `unraid/cloudflared/credentials.json`
- `unraid/authelia/users_database.yml`
- Authelia database, notification, secret, and runtime data files

Authelia runtime data is expected under `authelia/data/` in your configured appdata path.

## Usage

Start the core stack from the `unraid/` directory:

```bash
docker compose up -d
```

Validate the rendered Compose configuration:

```bash
docker compose config --quiet
```

## Validation

Authelia configuration:

```bash
docker run --rm --entrypoint authelia \
  -v "$PWD/authelia/configuration.yml:/config/configuration.yml:ro" \
  -v "$PWD/authelia/users_database.example.yml:/config/users_database.yml:ro" \
  --tmpfs /data \
  -e DOMAIN=example.com \
  -e AUTHELIA_IDENTITY_VALIDATION_RESET_PASSWORD_JWT_SECRET=replace_with_random_reset_jwt_secret \
  -e AUTHELIA_SESSION_SECRET=replace_with_random_session_secret \
  -e AUTHELIA_STORAGE_ENCRYPTION_KEY=replace_with_random_storage_encryption_key \
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
  -v "$PWD/cloudflared/config.example.yml:/etc/cloudflared/config.yml:ro" \
  -v "$PWD/cloudflared/credentials.example.json:/etc/cloudflared/credentials.json:ro" \
  cloudflare/cloudflared:2026.6.1 \
  tunnel --config /etc/cloudflared/config.yml ingress validate
```

## Security Notes

- Do not expose Traefik, Authelia, Redis, or Docker Socket Proxy directly on host ports.
- Keep Docker socket access behind Docker Socket Proxy.
- Keep Cloudflare tunnel credentials private.
- Replace all example passwords, secrets, users, tunnel IDs, and DNS values before production use.
- Rotate any secret that was ever committed or shared.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
