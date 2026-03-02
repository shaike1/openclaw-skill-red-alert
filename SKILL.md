---
name: oref-native
description: >
  Israeli Home Front Command (פיקוד העורף) real-time alert system.
  Polls oref-alerts Docker proxy every 5s. Sends WhatsApp notifications
  and triggers HA speaker via SSH tunnel. Runs as systemd service.
---

# ORef Native - OpenClaw Only 🚨

## Architecture

```
Pikud Ha-Oref API
       ↓
Docker: dmatik/oref-alerts (port 49000)
       ↓
oref_native.py - systemd service (polls every 5s)
       ↓
openclaw message → 📱 WhatsApp group
SSH → HA API    → 🔊 Speaker (tts.google_translate_iw_com)
```

## Deployment

Runs as a systemd service (auto-starts on boot):

```bash
# Status
systemctl status oref-native

# Restart
systemctl restart oref-native

# Logs
tail -f /var/log/oref_native.log
```

Service file: `/etc/systemd/system/oref-native.service`

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| OREF_API_URL | http://localhost:49000/current | oref-alerts proxy |
| MONITORED_AREAS | הרצליה,הרצליה - גליל ים ומרכז | Comma-separated areas |
| WHATSAPP_GROUP_JID | - | WhatsApp group JID |
| WHATSAPP_OWNER | - | Personal WhatsApp number |
| OPENCLAW_BIN | openclaw | Path to openclaw binary |
| HASS_TOKEN | - | HA long-lived access token |
| HA_SSH_HOST | 100.64.0.15 | HA host Tailscale IP |
| HA_SSH_USER | root | HA SSH user |
| HA_SSH_PASS | - | HA SSH password |
| HA_LOCAL_URL | http://172.30.32.1:8443 | HA internal URL (from HA host) |
| HA_TTS_SPEAKER | media_player.home_assistant_voice_09a069_media_player | Speaker entity |

## Alert Routing

| Type | WhatsApp | TTS |
|------|----------|-----|
| 🚀 Rockets (cat=1) | ✅ | ✅ |
| ✈️ Aircraft (cat=2) | ✅ | ✅ |
| 🔴 Infiltration (cat=10) | ✅ | ✅ |
| ✅ All clear (cat=13) | ✅ | ✅ |
| ⚠️ Pre-alert (cat=14) | ❌ | ✅ |
