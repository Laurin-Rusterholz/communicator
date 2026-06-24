# Communicator

Selbstgehosteter WhatsApp-Eigenbau. Single-File-Chat-Frontend (`index.html`,
Vanilla JS, Firebase Web SDK) im **Schiefer/Leinen-Design**. Liest direkt aus
**Firestore** (Polling) und sendet über einen **n8n-Webhook**. WhatsApp-Anbindung
über die **Evolution API (Baileys)** auf dem VPS.

> ⚠️ Evolution/Baileys verstößt gegen die WhatsApp-AGB (Sperrrisiko). Bewusst
> akzeptiert, private Zweit-SIM, kein Geschäftsbetrieb.

```
[Kunde WhatsApp] ⇄ [Evolution API (Docker)] ⇄ [n8n (Docker)] ⇄ [Firebase: Firestore + Storage] ⇄ [HTML-Frontend (Netlify)]
                     └─────────── selbes Docker-Netz, intern ──────────┘        Frontend LIEST Firestore,
                              nur n8n öffentlich (HTTPS via Caddy)                SENDET an n8n-Webhook
```

## Repo-Inhalt

| Pfad | Zweck |
|------|-------|
| `index.html` | **Das Programm.** Chat-Inbox (Evolution + Firestore). Auf Netlify ablegen. |
| `deploy/` | VPS-Stack: `docker-compose.yml`, `.env.example`, `Caddyfile`, `firestore.rules`, **`SETUP.md`** (Schritt-für-Schritt). |
| `legacy-meta-cloud-rtdb.html` | Vorgängerversion (Meta Cloud API + RTDB + Quantus). Nur Referenz. |

## Frontend (`index.html`)

- **Kontaktliste**: Name/Nummer, letzte Nachricht, Zeit, Ungelesen-Zähler, Avatare.
- **Chatverlauf**: Text, **Bilder**, **PDF-Dokumente**, **Sprachnachrichten** –
  anzeigen *und* senden (📎 für Datei, 🎤 für Sprachaufnahme via MediaRecorder).
- **Zeitstempel** + **Tages-Trenner** an jeder Nachricht.
- **Lesebestätigungen**: Häkchen aus `status` – gesendet `✓`, zugestellt/gelesen
  `✓✓`, gelesen zusätzlich teal eingefärbt.
- **Volltextsuche** über alle Nachrichten (Kontakte + Nachrichtentexte).
- **Lesen**: direkt aus Firestore per Polling (alle 4 s). Beim Öffnen eines Chats
  wird `unread` auf 0 gesetzt (einziger Schreibzugriff des Frontends).
- **Senden**: `POST {n8nPublicUrl}/webhook/wa-send`, Medien als Base64.
- **Schiefer/Leinen**: dunkel per Default, 🌓 schaltet auf Leinen (hell) um
  (in `localStorage` gemerkt). Kein Login.

### Konfiguration eintragen

Oben in `index.html` (Block „KONFIGURATION"):

1. **`firebaseConfig`** – Web-Config aus der Firebase-Console (Projekt
   `jupidu-36804`). Ist bereits mit den echten Werten gefüllt.
2. **`CONFIG.n8nPublicUrl`** – die öffentliche n8n-URL. Platzhalter `1-2-3-4`
   durch deine VPS-IP-mit-Bindestrichen ersetzen **oder** im laufenden Frontend
   über ⚙️ setzen (wird in `localStorage` gespeichert).

## Datenvertrag (verbindlich – identisch im n8n-Chat)

Firebase-Projekt **`jupidu-36804`**, Firestore + Storage.

**Collection `wa_chats`** (Doc-ID = Nummer, nur Ziffern, z. B. `41791234567`):
```js
{ number, name, lastText, lastTs /*ms*/, unread /*Zahl*/, updatedAt /*ms*/,
  // Teil 1 (Quantus/KI) – optional:
  quantusContactId?, quantusOrgId?,                       // Verknüpfung – Frontend schreibt
  aiSummary?: { text, todos:[string], updatedAt /*ms*/ }  // n8n schreibt (action:summarize)
}
```
**Collection `wa_messages`** (Auto-ID):
```js
{ number, dir:"in"|"out", type:"text"|"image"|"document"|"audio",
  text, mediaUrl, mime, fileName, ts /*ms*/, status:"sent"|"delivered"|"read", waMsgId }
```
**Storage**: `wa-media/{number}/{ts}_{fileName}` → `mediaUrl` = Download-URL.

**Senden (Frontend → n8n)**: `POST {n8nPublicUrl}/webhook/wa-send`
```js
// Body
{ to /*Ziffern*/, type, text?, mediaBase64? /*reines Base64, kein data:-Prefix*/, mime?, fileName? }
// Antwort
{ ok:boolean, waMsgId?, error? }
```
**Eingang (Evolution → n8n)**: `POST {intern}/webhook/wa-incoming`

> **CORS-Hinweis für den n8n-Chat:** Da das Frontend (Netlify) per Browser auf den
> n8n-Webhook postet, müssen **alle** Webhooks (`wa-send`, `wa-ai`, `wa-quantus`)
> CORS erlauben (`Access-Control-Allow-Origin: *`) und die `OPTIONS`-Preflight
> beantworten (im n8n-Webhook-Node „Allowed Origins" = `*` setzen).

### KI-Gateway (Frontend → n8n): `POST {n8nPublicUrl}/webhook/wa-ai`

Ein Endpunkt, `action` im Body. **Kein KI-Schlüssel im Frontend** – die Logik liegt in n8n.
```js
// Body: { action, number?, msgId?, text?, targetLang?, style? }
summarize        → schreibt wa_chats.aiSummary { text, todos:[string], updatedAt }; Antwort { ok }
suggest_replies  → { ok, suggestions:[string] }
draft_task       → { ok, title, due?, column? }
translate        → { ok, text }     // Eingabe: msgId ODER text, plus targetLang   (Teil 2)
polish           → { ok, text }     // Eingabe: text, optional style                (Teil 2)
```

### Quantus-Gateway (Frontend → n8n): `POST {n8nPublicUrl}/webhook/wa-quantus`

Quantus-**Lesen und Task-Anlage laufen komplett über n8n** – das Frontend greift nie direkt
auf Quantus-Collections zu; in `wa_chats` liegen nur die Link-IDs. `action` im Body:
```js
search       { q }                          → { ok, results:[{ id, kind:"person"|"org", name, subtitle? }] }
get          { number, contactId?, orgId? } → { ok, contact?{ id, name, role?, orgName? },
                                                     org?{ id, name, type? },
                                                     tasks:[{ title, status, dueDate? }] }   // offene Aufgaben
create_task  { title, description?, status, priority /*1–5*/, dueDate?,
               sourceApp:"communicator", sourceNumber, sourceMsgId?,
               quantusContactId?, quantusOrgId? }            → { ok, taskId? }
```
> **Quantus-Schema (geklärt):** Personen `entities.persons`, Organisationen
> `entities.organizations`, Aufgaben in der Tasks-Collection mit Board-Spalten
> `todo | doing | waiting | review | done` und Priorität `1–5`. n8n mappt
> `status`/`priority` auf die Quantus-Felder und ergänzt `sourceApp/sourceNumber/sourceMsgId`.

## Firestore-Sicherheitsregeln

Siehe `deploy/firestore.rules`: **Lesen frei**. Das Frontend darf auf `wa_chats`
nur eine kleine **Feld-Allowlist** schreiben – `unread` (nur auf `0`) sowie die
Quantus-Verknüpfung `quantusContactId`/`quantusOrgId` (Teil 1). Sonst kein
Schreibzugriff; n8n schreibt über das Admin SDK (Service-Account) und umgeht die
Regeln. **Tradeoff (Single-User, kein Login):** Ohne Auth schützt die Allowlist
nicht vor einem böswilligen Client, verhindert aber zuverlässig, dass das Frontend
n8n-Hoheitsfelder (`lastText`, Nachrichten-Status, `aiSummary` …) versehentlich
überschreibt. In der Firebase-Console unter *Firestore → Regeln* einspielen.

## VPS-Stack aufsetzen

Komplette Anleitung in **`deploy/SETUP.md`** (bestehendes Setup prüfen → Stack
starten → n8n ins Netz holen → n8n auf HTTPS umstellen → Zweit-SIM per QR
verbinden → Service-Account anlegen).

### Übergabe-Werte für den n8n-Chat
1. **n8n-URL:** `https://n8n.<VPS-IP-mit-Bindestrichen>.sslip.io`
2. **Evolution intern:** `http://evolution_api:8080`, API-Key aus `deploy/.env`,
   Instanz `communicator`.
3. **Firebase:** Service-Account-JSON (`deploy/SETUP.md` zeigt, wie du ihn
   erzeugst), Projekt `jupidu-36804`.
