# 🚨 ORef Alerts - OpenClaw Native

Real-time Israeli Home Front Command (פיקוד העורף) alert system.
**No Home Assistant dependency. No extra WhatsApp binary. OpenClaw handles everything.**

---

## Architecture

```
Pikud Ha-Oref API
       ↓
Docker: dmatik/oref-alerts (port 49000)   ← HTTP proxy for oref API
       ↓
oref_native.py — systemd service (polls every 5s)
       ↓
📱 openclaw message  → WhatsApp group
🔊 SSH → HA API     → TTS speaker  (via Tailscale tunnel)
```

---

## Quick Install

```bash
git clone https://github.com/shaike1/openclaw-skill-red-alert
cd openclaw-skill-red-alert
bash install.sh
```

The installer will:
- Pull and start the `dmatik/oref-alerts` Docker proxy
- Install Python dependencies (`requests`)
- Install the systemd service (`oref-native.service`)
- Enable it to start on boot

---

## Systemd Service

The monitor runs as a persistent systemd service:

```bash
# Status
systemctl status oref-native

# Logs
journalctl -u oref-native -f
tail -f /var/log/oref_native.log

# Restart
systemctl restart oref-native
```

Service file: `/etc/systemd/system/oref-native.service`

---

## Configuration

All config is passed via systemd `Environment=` directives in the service file.
Edit `/etc/systemd/system/oref-native.service`, then `systemctl daemon-reload && systemctl restart oref-native`.

| Variable | Default | Description |
|----------|---------|-------------|
| `OREF_API_URL` | `http://localhost:49000/current` | oref-alerts Docker proxy URL |
| `OREF_POLL_INTERVAL` | `5` | Polling interval in seconds |
| `OREF_COOLDOWN` | `60` | Min seconds between duplicate alerts |
| `MONITORED_AREAS` | `הרצליה,הרצליה - גליל ים ומרכז` | Comma-separated Hebrew area names |
| `WHATSAPP_GROUP_JID` | — | WhatsApp group JID (e.g. `...@g.us`) |
| `WHATSAPP_OWNER` | — | Your phone in E.164 (e.g. `+972...`) |
| `OPENCLAW_BIN` | `/usr/bin/openclaw` | Path to openclaw binary |
| `HASS_TOKEN` | — | Home Assistant long-lived access token |
| `HA_SSH_HOST` | `100.64.0.15` | HA host Tailscale IP for SSH tunnel |
| `HA_SSH_USER` | `root` | SSH user on HA host |
| `HA_SSH_PASS` | — | SSH password for HA host |
| `HA_LOCAL_URL` | `http://172.30.32.1:8443` | HA internal URL (reachable from HA host) |
| `HA_TTS_SPEAKER` | `media_player.home_assistant_voice_...` | HA media player entity ID |
| `CX3_ENABLED` | `false` | Enable 3CX outbound calls |

---

## Alert Routing

| Type | WhatsApp | 🔊 Speaker |
|------|----------|------------|
| 🚀 Rockets (cat=1) | ✅ | ✅ |
| ✈️ Aircraft (cat=2) | ✅ | ✅ |
| 🔴 Infiltration (cat=10) | ✅ | ✅ |
| ✅ All clear (cat=13) | ✅ | ✅ |
| ⚠️ Pre-alert (cat=14) | ❌ | ✅ |

---

## Home Assistant TTS — SSH Tunnel

The VPS cannot reach HA directly over the internet.
TTS is triggered by SSHing into the HA host (via Tailscale) and calling the HA local API from there:

```
VPS → SSH → HA host (100.64.0.15)
                  ↓
         curl → http://172.30.32.1:8443/api/services/tts/speak
```

Uses the `tts/speak` service with `tts.google_translate_iw_com` engine (required for HA 2025+).

---

## OpenClaw Bot Integration

The skill registers with OpenClaw and allows the WhatsApp bot to check alert status on demand.

When a user asks **"בדוק התרעות"** or **"סטטוס"**, the bot:
1. Runs `grep` on `/var/log/oref_native.log` via the `exec` tool
2. Returns a formatted Hebrew summary of recent alerts

### Requirements for bot exec

OpenClaw gateway must have exec enabled:

```json
// /root/.openclaw/openclaw.json  →  tools section
{
  "tools": {
    "exec": {
      "host": "gateway",
      "security": "allowlist",
      "ask": "off"
    }
  }
}
```

Allowlisted binaries in `/root/.openclaw/exec-approvals.json`:
`/usr/bin/curl`, `/usr/bin/systemctl`, `/usr/bin/grep`, `/usr/bin/tail`, `/usr/bin/python3`, `/usr/bin/sshpass`, `/usr/bin/docker`

### Model requirement

The default agent model must support tool use. Use either:
- `copilot-proxy/claude-sonnet-4-5` — requires the [tool-use patch](https://github.com/shaike1/ClaudeCode-Copilot-Proxy)
- `zai/glm-4.7` — works out of the box via OpenAI function calling

---

## Docker (oref-alerts proxy)

```bash
# Check status
docker ps | grep oref
docker logs oref-alerts --tail 20

# Restart
docker restart oref-alerts

# Full redeploy
docker stop oref-alerts && docker rm oref-alerts
docker run -d -p 49000:9001 --name oref-alerts --restart unless-stopped \
  -e TZ="Asia/Jerusalem" dmatik/oref-alerts:latest
```

---

## Requirements

- [OpenClaw](https://openclaw.ai) gateway running
- Docker (for `dmatik/oref-alerts` proxy)
- Python 3.8+
- `sshpass` (for HA TTS via SSH)
- Home Assistant with Google Translate TTS (optional, for speaker)
