# OmniRoute hardened deployment

This branch changes the default Docker posture from convenience-first to security-first. It is designed for a dedicated VM or host and assumes access through a local reverse proxy or SSH tunnel.

## What is enforced

- Dashboard, API and live WebSocket ports publish only on `127.0.0.1`.
- Redis, Qdrant, Bifrost and CLIProxyAPI are not published on the host.
- The dangerous `cli` and `host` Compose profiles were removed.
- No Docker socket, source repository, AI CLI home or host credential directory is mounted.
- Containers drop all Linux capabilities and use `no-new-privileges`.
- The OmniRoute root filesystem is read-only, with explicit writable mounts only for data and temporary files.
- API-key authentication is forced in Compose.
- Cloud sync endpoints are forced empty in Compose.
- Qdrant authentication is mandatory before enabling the memory profile.

## First deployment

```bash
cp .env.security.example .env
chmod 600 .env
mkdir -p data
chmod 700 data
# Replace every REPLACE_* value before continuing.
docker compose --profile base config
docker compose --profile base up -d
```

Do not expose ports `20128`, `20129` or `20132` directly. Put a TLS reverse proxy on the same host or access through an SSH tunnel. The proxy must authenticate users, strip query strings from access logs and apply request-size and rate limits.

## Required operational controls

1. Run in a dedicated VM with encrypted storage and restricted backups.
2. Restrict outbound traffic to the exact model providers in use; block RFC1918, link-local and cloud metadata destinations.
3. Disable cloud sync in the database/dashboard as well as through environment variables.
4. Keep MCP, A2A, plugins, custom middleware, MITM, VNC, Traffic Inspector and auto-update disabled until each feature is separately reviewed.
5. Disable detailed call payloads and semantic caching for sensitive workloads; minimize retention.
6. Rotate all provider and client keys after any test using privileged profiles from another branch.
7. Re-run dependency, secret and container-image scanning on every update.

## Deliberately unsupported in this Compose file

The upstream `cli` and `host` profiles grant access to the Docker socket, repository and AI-client credentials. They are intentionally absent. Reintroducing any of those mounts should be treated as granting the application host-level control.

## Residual risk

This hardening reduces unsafe deployment defaults but does not prove the application safe. The codebase retains a large privileged surface, including outbound webhooks, local process execution, integrations and storage of sensitive model traffic. A dynamic pentest and runtime egress monitoring are still required before Internet exposure or use with regulated data.
