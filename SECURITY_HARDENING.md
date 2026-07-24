# OmniRoute hardened home-LAN deployment

This branch is designed for a dedicated VM inside a trusted home VLAN. The goal is not to isolate OmniRoute from the network: authorized clients and the operational agent must be able to reach infrastructure services. The goal is to keep that access explicit, authenticated and separated from unrestricted host control.

## Security model

- Dashboard, API and WebSocket publish only on the address configured by `OMNIROUTE_BIND_IP`.
- `OMNIROUTE_BIND_IP` should be the VM/server address on the trusted management or services VLAN, never `0.0.0.0`.
- API-key authentication remains mandatory on the LAN.
- OmniRoute has outbound connectivity for model providers and authorized LAN services.
- Redis, Qdrant, Bifrost and CLIProxyAPI remain private to the internal Docker network.
- Containers drop Linux capabilities and use `no-new-privileges`.
- Cloud sync remains disabled; normal AI-provider connections remain available.
- No service receives `/var/run/docker.sock` or the host user's complete home directory.

## Modes

### Base

Use for clients such as Claude Code, Codex, Cursor and Cline running on other trusted machines. They connect to the OmniRoute LAN endpoint and authenticate with a client API key.

```bash
docker compose --profile base up -d
```

### Web

Use when browser-backed providers require Chromium/Playwright.

```bash
docker compose --profile web up -d
```

### Agent

Use when OmniRoute itself must execute CLIs and perform deployments.

```bash
mkdir -p data agent-workspace agent-home
chmod 700 data agent-workspace agent-home
docker compose --profile agent up -d
```

The agent can access the home LAN and Internet. Give it dedicated credentials inside `agent-home`, such as:

- a restricted SSH key for deployment targets;
- a kubeconfig bound to a least-privilege Kubernetes service account;
- scoped tokens for Proxmox, Coolify, Forgejo/GitHub and deployment APIs;
- known-host entries for SSH targets.

Do not copy the host's entire `~/.ssh`, `~/.claude`, `~/.codex`, `~/.cursor` or other personal configuration into this directory. Authenticate each required CLI inside the dedicated agent home.

The preferred deployment paths are SSH, HTTPS APIs, GitOps and the Kubernetes API. Direct access to the host Docker daemon is intentionally not provided because possession of the Docker socket is effectively root-equivalent on the host.

## First deployment

```bash
cp .env.security.example .env
chmod 600 .env
mkdir -p data
chmod 700 data
# Replace every REPLACE_* value.
# Set OMNIROUTE_BIND_IP to the server address on your trusted VLAN.
docker compose --profile base config
docker compose --profile base up -d
```

At the firewall, allow ports `20128`, `20129` and `20132` only from the VLANs or client addresses that need them. Block those ports from guest Wi-Fi, IoT VLANs and the Internet. Prefer internal DNS and TLS through the existing reverse proxy.

## Deployment authorization

Network reachability alone does not grant permission. Every infrastructure target should enforce its own identity and scope:

1. Create a dedicated `omniroute-agent` account or service account.
2. Allow only the deployment commands, namespaces, projects or stacks it needs.
3. Keep destructive operations behind separate credentials or manual approval.
4. Use SSH host-key verification and do not disable TLS verification.
5. Record deployments and credential use in the target systems.
6. Rotate agent credentials independently from personal administrator credentials.

## Features and compatibility

- External Claude Code/Codex/Cursor/Cline clients are supported through the LAN endpoint and API keys.
- Normal model providers are supported because OmniRoute joins an egress-capable Docker network.
- The `agent` profile restores packaged CLI execution without restoring Docker socket or host-home mounts.
- Qdrant, Bifrost and CLIProxyAPI are available to OmniRoute internally when their profiles are enabled.
- Cloud sync is disabled because the audited implementation may transmit provider and client credentials.
- MCP, plugins, MITM, VNC, Traffic Inspector and custom middleware may be enabled only when required and should receive separate review and credentials.

## Residual risk

This configuration reduces unsafe deployment defaults but does not make a privileged autonomous agent risk-free. An agent with SSH, Kubernetes or infrastructure API credentials can perform whatever those credentials permit. Least privilege, network segmentation, backups, audit logs and tested rollback remain mandatory. Code-level findings from the security assessment, including webhook SSRF and sensitive-data handling, still require remediation before Internet exposure or regulated workloads.
