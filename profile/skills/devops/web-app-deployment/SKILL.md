---
name: web-app-deployment
title: Web App & Dashboard Installation (Hermes Workspace, Hermes Dashboard)
scope: class-level
status: stable
last_updated: 2026-05-01
version: 1.0.0
description: Install, configure, and verify web-based dashboards and workspaces (Hermes Workspace, Hermes Agent Dashboard) that integrate with a running Hermes Agent gateway. Covers prerequisites, environment setup, development server launch, remote/Tailscale deployment, and troubleshooting common pitfalls.
---

# Web App & Dashboard Installation

Deploy and configure web-based dashboards and workspaces (Hermes Workspace, Hermes Agent Dashboard, etc.) that integrate with a running Hermes Agent gateway.

## When to use

- Installing a web UI/dashboard that connects to a local or remote Hermes Agent
- Setting up development environments for Node.js-based web tools
- Verifying service health and connectivity after installation
- Debugging startup failures or connection issues

## Prerequisites checklist

- **Hermes Agent** installed and gateway running (default: http://127.0.0.1:8642)
  - Verify: `curl http://127.0.0.1:8642/health` → `{"status":"ok"}`
- **Hermes Dashboard** (if required by the app): `hermes dashboard --port 9119 --no-open`
  - Verify: `curl http://127.0.0.1:9119/api/status` → JSON metadata
- **Node.js 22+** installed (check: `node --version`)
- **pnpm** package manager available
  - Install if missing: `npm install -g pnpm`
  - If not in PATH, use full path (e.g. `~/.hermes/node/bin/pnpm`)
- **Ports** 3000 (workspace), 9119 (dashboard), 8642 (gateway) free on localhost
- **Environment variables** configured in the web app's `.env` file:
  - `HERMES_API_URL=http://127.0.0.1:8642` (or your gateway address)
  - `HERMES_DASHBOARD_URL=http://127.0.0.1:9119` (if the app needs the dashboard)
  - `HERMES_API_TOKEN=<key>` only if your gateway uses `API_SERVER_KEY` auth

## Installation flow

### 1. Clone & enter project directory

```bash
git clone https://github.com/outsourc-e/hermes-workspace.git ~/hermes-workspace
cd ~/hermes-workspace
```

### 2. Install dependencies

```bash
pnpm install
```

If `pnpm` command not found, use full path:
```bash
~/.hermes/node/bin/pnpm install
```

### 3. Configure environment

```bash
cp .env.example .env
# Edit .env to set HERMES_API_URL and HERMES_DASHBOARD_URL
```

Ensure `.env` contains:
```
HERMES_API_URL=http://127.0.0.1:8642
HERMES_DASHBOARD_URL=http://127.0.0.1:9119
```

### 4. Start development server

```bash
pnpm dev
```

Server starts at http://localhost:3000 by default (override with `PORT=4000 pnpm dev`).

### 5. Verify health

- Web app: `curl http://127.0.0.1:3000` → HTML response
- API connectivity: Open browser and complete onboarding; check that chat works
- Dashboard APIs: `curl http://127.0.0.1:9119/api/status`

## Common pitfalls & fixes

### "pnpm: command not found" during startup
**Cause:** pnpm installed globally but not in PATH for background processes.
**Fix:** Use absolute path: `~/.hermes/node/bin/pnpm dev`, or add npm global bin to PATH.

### Dashboard refuses to bind to 0.0.0.0
**Cause:** Hermes dashboard intentionally binds to 127.0.0.1 only (security — exposes API keys).
**Fix:** Use default `--host 127.0.0.1` or omit host; do NOT use `--insecure` on untrusted networks.

### Workspace fails to connect to gateway
**Checks:**
- Gateway health: `curl http://127.0.0.1:8642/health`
- Dashboard health: `curl http://127.0.0.1:9119/api/status`
- `.env` URLs point to the correct addresses (not 127.0.0.1 when remote)
- If using Tailscale/VPN, set both URLs to the Tailscale/LAN IP
- If gateway has `API_SERVER_KEY` set, add `HERMES_API_TOKEN=<same-value>` to workspace `.env`

### Build hangs or takes too long
**Cause:** First run compiles the dashboard UI (Vite/esbuild).
**Fix:** Wait 30-60 seconds; watch `~/hermes-workspace` terminal for build output. Subsequent starts are instant (uses `node_modules/.cache`).

### Port already in use
**Fix:** Find and kill the process using the port:
```bash
# Example for port 3000
ss -tlnp | grep 3000
kill <PID>
```

## Remote / Tailscale deployment

If workspace runs on a server and you access it from another device:

1. On the server, set `.env` to use the Tailscale/LAN IP:
   ```
   HERMES_API_URL=http://100.x.y.z:8642
   HERMES_DASHBOARD_URL=http://100.x.y.z:9119
   ```
2. Ensure Hermes gateway listens on all interfaces: in `~/.hermes/.env` add `API_SERVER_HOST=0.0.0.0` and restart gateway.
3. Access workspace via the server's Tailscale IP (not localhost).
4. You can update URLs live from Settings → Connection without restarting (saved to `~/.hermes/workspace-overrides.json`).

## Verification checklist

- [ ] Gateway responding: `curl http://127.0.0.1:8642/health` → `{"status":"ok"}`
- [ ] Dashboard responding (if used): `curl http://127.0.0.1:9119/api/status` → JSON
- [ ] Workspace server running on port 3000 (check `ss -tlnp | grep 3000`)
- [ ] Workspace HTML loads in browser at http://localhost:3000
- [ ] Onboarding flow completes and chat sends a test message
- [ ] No errors in workspace terminal output

## Related skills

- `hermes-agent` — configure and manage the Hermes Agent gateway itself
- `systematic-debugging` — general troublehooting methodology for connection and startup issues
- `github-repo-management` — cloning repositories, managing remotes
