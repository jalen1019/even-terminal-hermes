# Even Hermes Terminal — Run Instructions

## What This Repo Does

`even-hermes-terminal` is a TypeScript bridge that makes Hermes Agent available as a provider in **Even Realities G2 Terminal Mode**, alongside the built-in Claude Code and Codex options.

It exposes the `@evenrealities/even-terminal` HTTP/SSE contract and forwards agent work to a local (or remote) Hermes Agent API server.

---

## Prerequisites

1. **Hermes Agent** installed and configured (`/Users/jalenadatan/.hermes/hermes-agent`)
2. **Node.js >= 20** (check with `node --version`)
3. **npm** (check with `npm --version`)
4. **OpenRouter API key** configured in Hermes (already set in this profile)
5. **Even G2 glasses** in Terminal Mode, or the Even desktop app in terminal mode

---

## Config Requirements

### Hermes Config (`~/.hermes/config.yaml` or profile config)

The Hermes API server must be enabled. This was added in the research profile:

```yaml
gateway:
  api_server:
    enabled: true
    max_concurrent_runs: 10
```

This can also be set via the CLI:
```bash
hermes config set gateway.api_server.enabled true
hermes config set gateway.api_server.max_concurrent_runs 10
```

### Bridge Auth

Set a bridge auth token:
```bash
BRIDGE_TOKEN=<your-bridge-token>
```

Use the same Hermes API key for both the gateway and the bridge:
```bash
API_SERVER_KEY=<your-64-char-hex-key>
HERMES_API_KEY=<your-64-char-hex-key>
```

> **Note:** The first time you set up a new Hermes installation, generate a strong key:
> ```bash
> openssl rand -hex 32
> ```
> The Hermes API server refuses to start with keys shorter than 16 characters for security reasons.

---

## Step-by-Step Startup

### Step 1 — Start the Hermes API Server

```bash
API_SERVER_ENABLED=true \
API_SERVER_KEY=d4b4379123cd8ca8ca8eb6a3e1e0dc1ff315ddbf2719424e312dc9519c79fb8ff74 \
API_SERVER_PORT=8642 \
API_SERVER_HOST=127.0.0.1 \
/Users/jalenadatan/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run --accept-hooks
```

You should see:
```
┌─────────────────────────────────────────────────────────┐
│           ⚕ Hermes Gateway Starting...                 │
└─────────────────────────────────────────────────────────┘
```

Verify it's running:
```bash
curl -s http://127.0.0.1:8642/health
# Expected: {"status":"ok","platform":"hermes-agent","version":"0.20.0"}

curl -s http://127.0.0.1:8642/v1/models \
  -H "Authorization: Bearer <your-64-char-hex-key>"
# Expected: list with your configured model (e.g. nvidia/nemotron-3-ultra-550b-a55b:free)
```

### Step 2 — Start the Bridge

```bash
cd /Users/jalenadatan/source/repos/even-terminal-hermes

BRIDGE_TOKEN=<your-bridge-token> \
HERMES_API_KEY="<your-64-char-hex-key>" \
npm start -- --host 0.0.0.0 --port 3456 --hermes-url http://127.0.0.1:8642
```

You should see:
```
Even Hermes Terminal v0.1.0
Local:  http://localhost:3456
LAN:    http://192.168.68.68:3456
Hermes: http://127.0.0.1:8642
Token:  <your-bridge-token>
```

Verify the bridge is running:
```bash
curl -s http://127.0.0.1:3456/ -H "Authorization: Bearer <your-bridge-token>"
# Expected: JSON with name, version, endpoints, messageTypes
```

### Step 3 — Connect from Even G2 / Even App

In the Even app's Terminal Mode, add a new provider pointing to:

```
http://192.168.68.68:3456?token=<your-bridge-token>&defaultProvider=codex&name=Hermes+Agent
```

Replace `192.168.68.68` with your computer's LAN IP if the glasses are on a different device.

To find your LAN IP:
```bash
ifconfig | grep "inet " | grep -v 127.0.0.1 | head -1 | awk '{print $2}'
```

---

## Testing the Full Flow

Send a test prompt from the Even app. To verify from command line:

```bash
# 1. Send a prompt
curl -s -X POST http://127.0.0.1:3456/api/prompt \
  -H "Authorization: Bearer <your-bridge-token>" \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello, this is a test"}'

# 2. Get the sessionId from the response, then watch events:
curl -s -N --max-time 60 \
  "http://127.0.0.1:3456/api/events?sessionId=hermes-<id>" \
  -H "Authorization: Bearer <your-bridge-token>"
```

You should see a stream of SSE events including `text_delta`, `result`, and `status` messages.

To check bridge-side state without SSE:
```bash
curl -s "http://127.0.0.1:3456/api/messages?sessionId=hermes-<id>" \
  -H "Authorization: Bearer <your-bridge-token>"

curl -s "http://127.0.0.1:3456/api/debug/status/hermes-<id>" \
  -H "Authorization: Bearer <your-bridge-token>"
```

---

## Checking if Processes Are Running

```bash
# Hermes API server
curl -s http://127.0.0.1:8642/health

# Bridge
curl -s http://127.0.0.1:3456/ -H "Authorization: Bearer <your-bridge-token>"

# Process list (macOS)
ps aux | grep -E "(hermes|node.*even-hermes)" | grep -v grep
```

---

## Stopping the Services

### Stop the bridge
Find the bridge PID and kill it, or use `process(action='kill', session_id='...')` if started via Hermes terminal tooling.

### Stop the Hermes gateway
```bash
hermes gateway stop
```
Or find the PID and kill it directly:
```bash
ps aux | grep "hermes_cli.main gateway" | grep -v grep | awk '{print $2}' | xargs kill
```

---

## Common Issues

### `API_SERVER_KEY is a placeholder or too short (<16 chars)`
Generate a proper key:
```bash
openssl rand -hex 32
```

Then set it in `~/.hermes/config.yaml` under `gateway.api_server.key` or export as `API_SERVER_KEY` before starting the gateway.

### `EADDRINUSE` on port 3456
Another bridge instance is already running. Kill it or use a different port:
```bash
npm start -- --host 0.0.0.0 --port 3457 --hermes-url http://127.0.0.1:8642
```

### No SSE events received on `/api/events`
- Check the session is actually `busy` via `/api/debug/status/<sessionId>`
- Check Hermes gateway logs for model errors (e.g. invalid model ID)
- Ensure both Hermes API server and bridge are running on the ports shown in their startup banners

### `hermes-agent is not a valid model ID`
This was a bug in an older version of the bridge that hardcoded `model: "hermes-agent"` in `/v1/runs` requests. It has been fixed in this repo — rebuild with `npm run build` if you pulled changes before this fix.

---

## Repo Structure

```
even-terminal-hermes/
├── src/
│   ├── bridge.ts          # Main HTTP/SSE bridge server
│   ├── cli.ts             # CLI entry point
│   ├── config.ts          # CLI flag / env var parsing
│   ├── constants.ts       # Defaults (port 3456, port 8642, etc.)
│   ├── hermes/
│   │   ├── client.ts      # Hermes API client (startRun, streamRun, approval, stop)
│   │   ├── event-handler.ts  # Translates Hermes events → Even messages
│   │   ├── mapping.ts     # Event type normalization
│   │   └── types.ts       # Hermes event types
│   ├── http.ts            # JSON helpers, CORS, auth
│   ├── sse.ts             # SSE parser + Even SSE client attachment
│   ├── session/
│   │   ├── session.ts     # Session model (ring buffer, push, history)
│   │   └── state-store.ts # Persistence
│   └── types.ts           # Shared TypeScript types
├── dist/                  # Built output (run via `npm start`)
├── test/
│   └── contract.ts        # Contract tests
└── README.md
```

---

## Quick Reference: One-Liner Startup

```bash
# Terminal 1: Hermes API server
API_SERVER_ENABLED=true API_SERVER_KEY=<your-64-char-hex-key> API_SERVER_PORT=8642 API_SERVER_HOST=127.0.0.1 /Users/jalenadatan/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run --accept-hooks

# Terminal 2: Bridge
cd /Users/jalenadatan/source/repos/even-terminal-hermes && BRIDGE_TOKEN=<your-bridge-token> HERMES_API_KEY="<your-64-char-hex-key>" npm start -- --host 0.0.0.0 --port 3456 --hermes-url http://127.0.0.1:8642
```

---

## Last Updated

2026-09-02 — after fixing the `model: "hermes-agent"` bug, enabling the Hermes API server in config, and verifying end-to-end SSE delivery.
