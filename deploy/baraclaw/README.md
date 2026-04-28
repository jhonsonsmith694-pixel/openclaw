# Baraclaw deploy notes (jhonsonsmith694-pixel fork)

Personal deploy notes for the @baraclaw_bot Telegram instance running on a GCP
Compute Engine VM. These are fork-specific operator notes — NOT a general
OpenClaw deployment guide.

## Where it runs

- Host: GCP Compute Engine (asia-southeast1-a), Ubuntu 24.04, n4-standard-2
  (2 vCPU / 8GB RAM)
- Service: systemd user unit `openclaw-gateway.service` for user `u0_a375`
  (created by the npm-based `openclaw onboard` flow)
- Linger enabled (`loginctl enable-linger u0_a375`) so the service runs without
  an active login session and survives VM reboots
- `Restart=always`, `RestartSec=5` — auto-recovers from crashes

## Model layout (multi-tier fallback)

OpenClaw's `agents.defaults.model` is configured as a chain. If the primary
fails or hits rate limits, OpenClaw walks the fallback list in order and the
bot keeps replying.

| Tier | Model | Provider | Notes |
| --- | --- | --- | --- |
| Primary | `deepseek-v3-2-251201` | BytePlus ARK | Free pack, fastest, strongest stable |
| Fallback 1 | `gemini-2.5-flash` | Google AI Studio | Free tier, 1M tokens/day, very fast |
| Fallback 2 | `gpt-oss-120b-250805` | BytePlus ARK | Reasoning model, separate free pack |
| Fallback 3 | `gemma-4-31b-it` | Google AI Studio | Free tier |
| Fallback 4 | `qwen2.5:3b` | Local Ollama on the VM | Always-on safety net, slowest (~25s/reply) |

Provider endpoints in use:

- BytePlus international: `https://ark.ap-southeast.bytepluses.com/api/v3`
- Google AI Studio: `https://generativelanguage.googleapis.com` (native
  `google-generative-ai` API, not the OpenAI-compat shim)
- Ollama local: `http://127.0.0.1:11434`

## Components

- Channel: Telegram bot `@baraclaw_bot` (long polling, no inbound port needed)
- DM policy: `pairing` — first DM from a new user yields a pairing code that
  the operator must approve via CLI
- Bonjour mDNS plugin disabled on the VPS via `OPENCLAW_DISABLE_BONJOUR=1`

## Recipes

Status / health:

```
sudo -u u0_a375 XDG_RUNTIME_DIR=/run/user/1001 \
  systemctl --user status openclaw-gateway
```

Restart:

```
sudo -u u0_a375 XDG_RUNTIME_DIR=/run/user/1001 \
  systemctl --user restart openclaw-gateway
```

Live logs:

```
sudo -u u0_a375 XDG_RUNTIME_DIR=/run/user/1001 \
  journalctl --user -u openclaw-gateway -f
```

Approve a Telegram pairing code (replace `CODE`):

```
sudo -u u0_a375 -H bash -lc \
  'node /usr/lib/node_modules/openclaw/dist/index.js pairing approve telegram CODE'
```

Edit config (validated):

```
sudo -u u0_a375 -H bash -lc \
  'node /usr/lib/node_modules/openclaw/dist/index.js config set --merge --batch-json "[{\"path\":\"...\",\"value\":...}]"'
```

Pull a new local Ollama model:

```
sudo -u u0_a375 -H bash -lc 'ollama pull <model:tag>'
```

## Files of interest on the VM

- `/home/u0_a375/.openclaw/openclaw.json` — main config
- `/home/u0_a375/.config/systemd/user/openclaw-gateway.service` — base unit
- `/home/u0_a375/.config/systemd/user/openclaw-gateway.service.d/secrets.conf`
  — drop-in supplying provider API keys and `OPENCLAW_DISABLE_BONJOUR=1`
- `/tmp/openclaw-1001/openclaw-YYYY-MM-DD.log` — JSON log file (rotates daily)

## Templates

See `secrets.conf.example` and `openclaw-config.snippet.json` in this directory
for the shape of the secrets drop-in and the model/provider config block.

## Why not Docker

The VM was already onboarded with the npm install flow; switching to the
Docker compose layout means a port conflict (the npm-installed service binds
`127.0.0.1:18789`) and a re-onboard. The npm + systemd path is what is
deployed, so these notes target that.
