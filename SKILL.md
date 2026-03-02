---
name: oref-native
description: >
  Israeli Home Front Command (פיקוד העורף) real-time alert system.
  Polls oref-alerts Docker proxy every 5s. Sends WhatsApp notifications
  and triggers HA speaker via SSH tunnel. Runs as systemd service.
  Triggers on: (1) checking alert status, (2) checking service status,
  (3) viewing recent alerts, (4) restarting the monitor, (5) testing the speaker.
---

# ORef Native - OpenClaw Only 🚨

## IMPORTANT - Already Running as Systemd Service

This skill is deployed and running as a systemd service. Do NOT try to run
oref_native.py manually. Always use systemctl commands below.

## Quick Commands

### Check if service is running
```bash
systemctl status oref-native --no-pager
```

### Check active alerts right now
```bash
curl -s http://localhost:49000/current | python3 -m json.tool
```

### View recent alert history
```bash
grep "🚨\|✅ WhatsApp\|✅ Speaker\|⚠️ Speaker" /var/log/oref_native.log | tail -30
```

### Restart the monitor
```bash
systemctl restart oref-native && systemctl status oref-native --no-pager
```

### Test speaker (TTS via HA)
```bash
sshpass -p 'Tr1C0late' ssh -o StrictHostKeyChecking=no root@100.64.0.15 \
  "curl -sk -o /dev/null -w '%{http_code}' -X POST http://172.30.32.1:8443/api/services/tts/speak \
   -H 'Authorization: Bearer \$HASS_TOKEN' \
   -H 'Content-Type: application/json' \
   -d '{\"entity_id\":\"tts.google_translate_iw_com\",\"media_player_entity_id\":\"media_player.home_assistant_voice_09a069_media_player\",\"message\":\"בדיקת מערכת פיקוד העורף\",\"language\":\"iw\"}'"
```

### View live logs
```bash
tail -f /var/log/oref_native.log
```

## Architecture

```
Pikud Ha-Oref API
       ↓
Docker: dmatik/oref-alerts (port 49000)
       ↓
oref_native.py - systemd service (polls every 5s)
       ↓
openclaw message → 📱 WhatsApp group (120363417492964228@g.us)
SSH → HA API    → 🔊 Speaker (media_player.home_assistant_voice_09a069_media_player)
```

## Infrastructure

| Component | Value |
|-----------|-------|
| oref-alerts API | http://localhost:49000/current |
| HA host (Tailscale) | 100.64.0.15 |
| HA internal URL | http://172.30.32.1:8443 |
| TTS engine | tts.google_translate_iw_com |
| Speaker | media_player.home_assistant_voice_09a069_media_player |
| WhatsApp group | 120363417492964228@g.us |
| Monitored areas | הרצליה, הרצליה - גליל ים ומרכז |
| Service file | /etc/systemd/system/oref-native.service |
| Log file | /var/log/oref_native.log |

## Alert Routing

| Type | WhatsApp | TTS |
|------|----------|-----|
| 🚀 Rockets (cat=1) | ✅ | ✅ |
| ✈️ Aircraft (cat=2) | ✅ | ✅ |
| 🔴 Infiltration (cat=10) | ✅ | ✅ |
| ✅ All clear (cat=13) | ✅ | ✅ |
| ⚠️ Pre-alert (cat=14) | ❌ | ✅ |
