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

## ⚡ AGENT INSTRUCTIONS - READ FIRST

You have **direct exec access** to this server. When the user asks anything about
alerts, status, or this system — **use the exec tool immediately** to run the
commands below. Do NOT say you lack access. Do NOT ask the user to run commands.
Do NOT say "אין לי גישה". Just run it.

### User asks "מה ההתרעות האחרונות" / "check alerts" / "בדוק התרעות"
→ Run this immediately:
```bash
grep "🚨\|✅ WhatsApp\|✅ Speaker\|⚠️ Speaker" /var/log/oref_native.log | tail -20
```

### User asks "מה קורה עכשיו" / "are there active alerts"
→ Run this immediately:
```bash
curl -s http://localhost:49000/current | python3 -m json.tool
```

### User asks "סטטוס" / "status" / "האם עובד"
→ Run this immediately:
```bash
systemctl status oref-native --no-pager && curl -s http://localhost:49000/current | python3 -m json.tool
```

### User asks to restart
→ Run this immediately:
```bash
systemctl restart oref-native && systemctl status oref-native --no-pager
```

### User asks to test the speaker
→ Run this immediately:
```bash
HASS_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJiZGIwNmIyOTNjNjE0OGE3YTgzMDViNmQ5YzIwOWNkZiIsImlhdCI6MTc3MjQ2OTEzOSwiZXhwIjoyMDg3ODI5MTM5fQ.NftilQndevpqzfReHOJtRwUDjSpYjuCMB-6FniPhAj0"
sshpass -p 'Tr1C0late' ssh -o StrictHostKeyChecking=no root@100.64.0.15 \
  "curl -sk -o /dev/null -w '%{http_code}' -X POST http://172.30.32.1:8443/api/services/tts/speak \
   -H 'Authorization: Bearer $HASS_TOKEN' \
   -H 'Content-Type: application/json' \
   -d '{\"entity_id\":\"tts.google_translate_iw_com\",\"media_player_entity_id\":\"media_player.home_assistant_voice_09a069_media_player\",\"message\":\"בדיקת מערכת פיקוד העורף\",\"language\":\"iw\"}'"
```

---

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

## Docker Commands

```bash
# Check container
docker ps | grep oref
docker logs oref-alerts --tail 20

# Restart
docker restart oref-alerts

# Full redeploy
docker stop oref-alerts && docker rm oref-alerts
docker run -d -p 49000:9001 --name oref-alerts --restart unless-stopped \
  -e TZ="Asia/Jerusalem" dmatik/oref-alerts:latest
```

## Alert Routing

| Type | WhatsApp | TTS |
|------|----------|-----|
| 🚀 Rockets (cat=1) | ✅ | ✅ |
| ✈️ Aircraft (cat=2) | ✅ | ✅ |
| 🔴 Infiltration (cat=10) | ✅ | ✅ |
| ✅ All clear (cat=13) | ✅ | ✅ |
| ⚠️ Pre-alert (cat=14) | ❌ | ✅ |
