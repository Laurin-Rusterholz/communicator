# Communicator Pro

Single-File-Web-App (`index.html`, Vanilla JS, Firebase Web SDK) als Posteingang für
WhatsApp-Nachrichten, die ein n8n-Workflow über die Meta Cloud API empfängt und in die
Firebase Realtime Database schreibt. Antworten und „Sprachmemos" werden hier bearbeitet;
markierte Nachrichten landen als Aufgabe in **Quantus**.

```
WhatsApp ──► Meta Cloud API ──► n8n ──► Firebase RTDB  ──►  Communicator Pro (diese App)
                                         communicator_inbox          │
                                                                     ├─ Antwort:  status=antwort_bereit + antwortText  ──► n8n ──► WhatsApp
                                                                     └─ Aufgabe:  push quantus_task_inbox  ──► Quantus (ai-sync) importiert als Task
```

## Setup

1. **Firebase-Config eintragen.** Oben in `index.html` die Konstante `firebaseConfig`
   mit den echten Werten füllen (gleiches Projekt wie Quantus, `jupidu-36804`).
   `databaseURL` ist für die Realtime Database **Pflicht**, `storageBucket` für die
   Sprachmemo-Wiedergabe.
2. **Hosten.** Reine statische Datei – z.B. auf Netlify ablegen oder lokal öffnen.
   Kein Build-Schritt.

## Datenmodell `communicator_inbox`

Jeder Kind-Knoten (Key = Nachrichten-ID) hat die von n8n geschriebenen Felder:

| Feld | Bedeutung |
|------|-----------|
| `from` | Absender-Telefonnummer |
| `name` | Absender-Name |
| `text` | Nachrichtentext |
| `type` | z.B. `text`, `sprachmemo` |
| `status` | `persoenlich_offen` \| `sprachmemo_offen` \| `automatisch_beantwortet` |
| `audioId` | ID/URL der Sprachnachricht (für Wiedergabe) |
| `timestamp` | Zeitpunkt (ms, Sekunden oder ISO) |

Beim **Antworten** schreibt die App in denselben Knoten zurück:
`status = "antwort_bereit"`, `antwortText`, `antwortTimestamp`, `antwortBy`.
→ Der n8n-Workflow holt Knoten mit `status = "antwort_bereit"` ab, sendet sie via
WhatsApp und setzt den Status anschließend (z.B. auf `beantwortet`).

## Quantus-Anbindung `quantus_task_inbox`

„✅ Als Aufgabe an Quantus" schreibt einen kompakten Task in `quantus_task_inbox`:

```json
{
  "title": "…", "description": "…", "status": "todo", "priority": 3,
  "dueDate": null, "tags": ["WhatsApp"], "source": "communicator",
  "from": "…", "name": "…", "text": "…", "origMessageId": "…", "createdAt": 0
}
```

Quantus (Repo `ai-sync`, `public/index.html`) liest diese Queue live, legt pro Eintrag
über sein bestehendes `createEntity("task", …)` einen echten Task in `entities.tasks`
an und entfernt den Queue-Eintrag wieder. So erscheint die Aufgabe direkt in Quantus,
ohne dass die App die Quantus-State-Datei manipulieren muss.

## Einstellungen / Auto-Antwort-Steuerung (`/settings`, für n8n)

Über das Zahnrad (⚙️) in der App werden Einstellungen in RTDB unter `/settings`
abgelegt. **n8n liest diese Werte** und entscheidet im Sende-Flow über die
automatischen Antworten (5-Minuten-Verzögerung, „nur eine Antwort bei mehreren
Nachrichten" und Gruppen-Erkennung macht n8n selbst).

| Pfad | Typ | Inhalt |
|------|-----|--------|
| `/settings/availability` | Objekt | `{ mode: "verfuegbar"\|"beschaeftigt", busyUntil: <ISO\|null>, updatedAt: <ISO> }` |
| `/settings/awayMessage` | String | Eigene Abwesenheitsmeldung (leer ⇒ generischer Text mit Termin) |
| `/settings/nextReplyAt` | String\|null | Nächster Antworttermin (ISO 8601) |
| `/settings/groupAwayMessage` | String | Gruppen-Auto-Antwort, Platzhalter `{nextReplyAt}` wird durch den Termin ersetzt |

**Auto-Antwort-Logik in n8n (Vorschlag):**
- Nur senden, wenn `availability.mode === "beschaeftigt"` (und falls `busyUntil`
  gesetzt: nur solange `now < busyUntil`). Bei `verfuegbar` → keine Auto-Antwort.
- Einzelchat: `awayMessage`, falls leer → generischer Text der `nextReplyAt` nennt
  (z.B. „Ich melde mich wieder am &lt;nextReplyAt&gt;.").
- Gruppe: `groupAwayMessage` mit `{nextReplyAt}` ersetzt; nur **einmalig** pro Gruppe.

Der OpenAI-API-Key für die „Zusammenfassen"-Funktion wird **ausschließlich im
Browser (localStorage)** gespeichert – niemals im Code oder in der RTDB.

## Firebase Realtime Database – Regeln

Die App (und n8n) brauchen Lese-/Schreibrechte auf die Pfade. Beispiel
(für den produktiven Einsatz mit Auth absichern):

```json
{
  "rules": {
    "communicator_inbox": { ".read": true, ".write": true },
    "quantus_task_inbox":  { ".read": true, ".write": true },
    "settings":            { ".read": true, ".write": true }
  }
}
```

## Sprachmemos

Zur Wiedergabe wird `audioId` aufgelöst:
- ist `audioId` bereits eine `http(s)`-URL → direkt abgespielt;
- sonst über Firebase Storage unter `CONFIG.audioPathTemplate`
  (Standard `communicator_audio/{audioId}` – bei Bedarf an das n8n-Setup anpassen).
