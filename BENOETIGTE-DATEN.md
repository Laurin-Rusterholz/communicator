# Benötigte Daten — Communicator-Fertigstellung

Diese Liste am Schluss ausgefüllt zurückgeben. **Teil A** trage ich danach in die
Dateien ein und pushe erneut. **Teil B/C** kannst nur du erledigen (ich habe keinen
SSH-/Google-Cloud-Zugriff).

---

## A) Daten, die ICH einsetze (dann Re-Push)

> Ausfüllen und zurückschicken — ich ersetze die Platzhalter und gebe die finale n8n-URL aus.

```
1) VPS öffentliche IPv4        :  ____________________   (z. B. 203.0.113.47)   ← MUSS
2) n8n-Containername (docker ps):  ____________________   (Default: n8n)
3) Deployment-Variante         :  [ ] A = Ports 80/443 frei → unser Caddy
                                  [ ] B = es läuft schon ein Reverse-Proxy (Caddy/Traefik/nginx)
4) n8n-Port (falls ≠ 5678)     :  ____________________   (Default: 5678)
5) Let's-Encrypt-E-Mail        :  ____________________   (aktuell: contact@laurin-rusterholz.ch)
```

**Wohin die Werte fließen (zur Info — mache ich automatisch):**
- IPv4 → `deploy/docker-compose.yml`, `deploy/.env.example`, `deploy/Caddyfile`,
  `index.html` (`CONFIG.n8nPublicUrl`) und die Beispiele in `deploy/SETUP.md`
  (als `n8n.<ip-mit-bindestrichen>.sslip.io` / `evo.<ip-…>.sslip.io`).
- n8n-Containername → `N8N_CONTAINER` in `.env.example`, Caddy-Proxy-Ziel,
  Eingangs-Webhook-URL.
- Variante A/B → ob der `caddy`-Service mitläuft oder dein bestehender Proxy genutzt wird.

---

## B) Nur DU in Firebase / Google (ich habe keinen Zugriff)

- [ ] **Firestore aktivieren** im Projekt `jupidu-36804` (**Native Mode**).
      ⚠️ Bisher nutzt ihr die Realtime DB — Firestore ist evtl. noch nicht angelegt.
- [ ] **`deploy/firestore.rules` veröffentlichen** (Console → Firestore → Regeln).
- [ ] **Service-Account-JSON erzeugen** (`gcloud`-Befehle in `deploy/SETUP.md`,
      Rollen `roles/datastore.user` + `roles/storage.objectAdmin`).
      → Das ist **Übergabe-Wert 3** für den n8n-Chat.

---

## C) Nur DU auf dem VPS (Befehle in `deploy/SETUP.md`)

- [ ] Bestehendes Setup prüfen (`docker ps`, Ports 80/443, n8n-Netz) — Schritt 0
- [ ] Ports **80/443** öffnen (`ufw` + Hostinger-Panel) — Let's Encrypt braucht Port 80
- [ ] Stack starten (`docker compose up -d`) — Schritt 4
- [ ] n8n dem Netz `wa_net` beitreten (`docker network connect`) — Schritt 5
- [ ] n8n auf HTTPS umstellen (`WEBHOOK_URL`/`N8N_HOST`) + neu starten — Schritt 6
- [ ] Zweit-SIM per **QR** verbinden + Instanz-Webhook auf n8n setzen — Aufgabe 2

---

## Erinnerung: 3 Übergabe-Werte für den n8n-Chat

1. **n8n-URL:** `https://n8n.<IP-mit-Bindestrichen>.sslip.io`  (gebe ich final aus, sobald IP da)
2. **Evolution intern:** `http://evolution_api:8080` · API-Key
   `1da5027fb491d99a0e2f95139afe4cc522980fd00316df39` · Instanz `communicator`
3. **Firebase:** Service-Account-JSON (`communicator-n8n.json`), Projekt `jupidu-36804`
