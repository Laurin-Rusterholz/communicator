# Communicator – VPS-Setup (Evolution API + n8n + Caddy)

Schritt-für-Schritt für deinen Hostinger-VPS. Reihenfolge einhalten.
Ersetze überall:
- `1-2-3-4` → deine **VPS-IP mit Bindestrichen** (z. B. `203.0.113.47` → `203-0-113-47`)
- `<N8N_CONTAINER>` → Name deines bestehenden n8n-Containers (aus Schritt 0)
- `<EVOLUTION_API_KEY>` → der Wert aus `deploy/.env` (`1da5027f…fd00316df39`)

> **Architektur**
> `WhatsApp ⇄ Evolution API ⇄ n8n ⇄ Firebase (Firestore+Storage) ⇄ HTML-Frontend (Netlify)`
> Evolution & n8n liegen im selben Docker-Netz `wa_net` und sprechen **intern**
> (`http://evolution_api:8080`, `http://<N8N_CONTAINER>:5678`). Öffentlich (HTTPS)
> ist nur Caddy: `n8n.<ip>.sslip.io` (für das Frontend) und `evo.<ip>.sslip.io`
> (für QR-Pairing/Manager). Kein eigener Domainname nötig.

---

## Schritt 0 — Bestehendes Setup prüfen (NICHTS überschreiben)

```bash
ssh root@DEINE_VPS_IP

# Welche Container laufen, welche Ports sind belegt?
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'

# Wo liegt die n8n-compose? (häufig ~ , /opt , /root)
grep -rl --include=docker-compose.y*ml -e 'n8n' / 2>/dev/null

# An welchem Docker-Netz hängt n8n schon?
docker inspect -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}' <N8N_CONTAINER>

# Läuft schon ein Reverse Proxy / ist 80/443 belegt?
docker ps --filter 'publish=80' --filter 'publish=443'
ss -tlnp | grep -E ':80 |:443 ' || echo 'Ports 80/443 frei'
```

Merke dir: **n8n-Containername** und ob **80/443 frei** sind.
- 80/443 **frei** → Variante A (unser Caddy, Standard).
- Es läuft **schon ein Reverse Proxy** (Caddy/Traefik/nginx) → Variante B (unten).

---

## Schritt 1 — Dateien auf den VPS legen

```bash
mkdir -p /opt/communicator-stack && cd /opt/communicator-stack
# docker-compose.yml, Caddyfile und .env.example aus deploy/ hierher kopieren
# (per scp, git clone oder Editor). Dann:
cp .env.example .env
```

## Schritt 2 — `.env` anpassen

```bash
nano .env
```
- `1-2-3-4` in `N8N_HOSTNAME`/`EVO_HOSTNAME` durch deine IP-mit-Bindestrichen ersetzen.
- `N8N_CONTAINER=` auf deinen echten n8n-Containernamen setzen.
- `POSTGRES_PASSWORD` / `EVOLUTION_API_KEY` sind bereits zufällig gesetzt – lassen.

## Schritt 3 — Firewall öffnen (Let's Encrypt braucht Port 80)

```bash
ufw allow 80/tcp && ufw allow 443/tcp   # falls ufw aktiv
# Zusätzlich im Hostinger-Panel (hPanel → Firewall) 80 + 443 freigeben.
```

## Schritt 4 — Stack starten

**Variante A (Standard, eigener Caddy):**
```bash
cd /opt/communicator-stack
docker compose up -d
docker compose logs -f caddy   # auf "certificate obtained" warten, dann Strg+C
```

**Variante B (du hast schon einen Reverse Proxy):**
Den `caddy`-Service NICHT starten und in deinem bestehenden Proxy zwei Hosts ergänzen
(`n8n.<ip>.sslip.io` → `<N8N_CONTAINER>:5678`, `evo.<ip>.sslip.io` → `evolution_api:8080`):
```bash
docker compose up -d postgres redis evolution_api
```

## Schritt 5 — n8n dem Netz `wa_net` beitreten lassen

Damit Caddy → n8n, Evolution → n8n und n8n → Evolution sich **per Containername**
finden:
```bash
docker network connect wa_net <N8N_CONTAINER>
# Test (sollte 'evolution_api' auflösen):
docker exec <N8N_CONTAINER> sh -c "wget -qO- http://evolution_api:8080 >/dev/null && echo OK"
```

## Schritt 6 — n8n auf die öffentliche HTTPS-URL umstellen

In deiner **bestehenden** n8n-Konfiguration (compose/.env) ergänzen/anpassen:
```env
N8N_HOST=n8n.1-2-3-4.sslip.io
N8N_PROTOCOL=https
N8N_PORT=5678
WEBHOOK_URL=https://n8n.1-2-3-4.sslip.io/
N8N_EDITOR_BASE_URL=https://n8n.1-2-3-4.sslip.io/
N8N_PROXY_HOPS=1
```
Neu starten:
```bash
docker restart <N8N_CONTAINER>      # bzw. `docker compose up -d <service>` in deiner n8n-compose
```
Prüfen:
```bash
curl -I https://n8n.1-2-3-4.sslip.io/   # 200/302 + gültiges TLS
curl -I https://evo.1-2-3-4.sslip.io/   # Evolution erreichbar
```

---

## AUFGABE 2 — Zweit-SIM per QR verbinden

### A) Bequem über die Manager-UI
1. `https://evo.1-2-3-4.sslip.io/manager` öffnen.
2. Als **API-Key** den `EVOLUTION_API_KEY` eingeben.
3. **Create Instance** → Name `communicator`, Channel **Baileys** (Standard).
4. QR anzeigen lassen und mit der **Zweit-SIM** scannen
   (WhatsApp → Einstellungen → *Verknüpfte Geräte* → *Gerät verknüpfen*).

### B) Oder per API (Instanz anlegen + QR holen)
```bash
# Instanz anlegen (liefert qrcode.base64 + pairingCode zurück)
curl -sS -X POST "https://evo.1-2-3-4.sslip.io/instance/create" \
  -H "apikey: <EVOLUTION_API_KEY>" -H "Content-Type: application/json" \
  -d '{"instanceName":"communicator","integration":"WHATSAPP-BAILEYS","qrcode":true}'

# QR später erneut holen / neu verbinden:
curl -sS "https://evo.1-2-3-4.sslip.io/instance/connect/communicator" \
  -H "apikey: <EVOLUTION_API_KEY>"
```
Den Wert von `qrcode.base64` (beginnt mit `data:image/png;base64,…`) in die
Browser-Adresszeile einfügen → QR scannen. Status prüfen:
```bash
curl -sS "https://evo.1-2-3-4.sslip.io/instance/connectionState/communicator" \
  -H "apikey: <EVOLUTION_API_KEY>"      # erwartet: "state":"open"
```

### C) Eingangs-Webhook der Instanz auf n8n setzen
Damit eingehende Nachrichten (inkl. Medien als Base64) an den **WA-Inbound**-Workflow
gehen, den der andere Chat baut:
```bash
curl -sS -X POST "https://evo.1-2-3-4.sslip.io/webhook/set/communicator" \
  -H "apikey: <EVOLUTION_API_KEY>" -H "Content-Type: application/json" \
  -d '{
    "webhook": {
      "enabled": true,
      "url": "http://<N8N_CONTAINER>:5678/webhook/wa-incoming",
      "webhookByEvents": false,
      "base64": true,
      "events": ["MESSAGES_UPSERT","MESSAGES_UPDATE","SEND_MESSAGE","CONTACTS_UPSERT"]
    }
  }'
```
> Ältere v2-Versionen erwarten die Felder flach (`{"enabled":true,"url":…,"events":…}`)
> statt unter `"webhook"`. Falls 400: ohne den `webhook`-Wrapper erneut senden.
> `base64:true` ⇒ Evolution schickt Mediendaten direkt mit – n8n lädt sie nach
> Firebase Storage hoch.

---

## Firebase-Service-Account für n8n (musst DU anlegen)

Ich kann in deinem Google-Projekt keinen Account anlegen (kein Zugriff auf deine
GCP-Konsole). Mach es so – dauert 1 Minute:

**Per `gcloud` (lokal oder Cloud Shell):**
```bash
gcloud config set project jupidu-36804
gcloud iam service-accounts create communicator-n8n \
  --display-name="Communicator n8n (Firestore+Storage)"

gcloud projects add-iam-policy-binding jupidu-36804 \
  --member="serviceAccount:communicator-n8n@jupidu-36804.iam.gserviceaccount.com" \
  --role="roles/datastore.user"          # Firestore lesen/schreiben

gcloud projects add-iam-policy-binding jupidu-36804 \
  --member="serviceAccount:communicator-n8n@jupidu-36804.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"     # Medien in Storage hochladen

gcloud iam service-accounts keys create communicator-n8n.json \
  --iam-account=communicator-n8n@jupidu-36804.iam.gserviceaccount.com
```
Die erzeugte **`communicator-n8n.json`** ist die Datei, die du im n8n-Chat als
Google-/Firebase-Credential hinterlegst.

**Oder per Konsole:** Firebase-Console → ⚙️ *Projekteinstellungen* →
*Dienstkonten* → *Neuen privaten Schlüssel generieren* → JSON herunterladen.
(Das Standard-Firebase-Admin-Dienstkonto hat Firestore- **und** Storage-Rechte.)

---

## Firestore-Regeln einspielen

`deploy/firestore.rules` in der Firebase-Console unter
**Firestore → Regeln** einfügen und veröffentlichen (oder
`firebase deploy --only firestore:rules`). Lesen ist frei, das Frontend darf nur
`unread` auf 0 setzen; n8n schreibt via Admin SDK an den Regeln vorbei.

---

## ✅ Übergabe-Werte für den n8n-Chat

1. **Öffentliche n8n-URL:** `https://n8n.1-2-3-4.sslip.io`
   → Frontend ruft `https://n8n.1-2-3-4.sslip.io/webhook/wa-send` auf.
2. **Evolution intern:** Base-URL `http://evolution_api:8080`,
   API-Key `1da5027fb491d99a0e2f95139afe4cc522980fd00316df39`,
   Instanz `communicator`.
3. **Firebase:** Service-Account-JSON `communicator-n8n.json` (oben erzeugt),
   Projekt `jupidu-36804`, Storage-Bucket `jupidu-36804.firebasestorage.app`.
