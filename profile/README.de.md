# Formtastic

**Die mobile No-Code Formular-App für Unternehmen mit Außendienst.**

🌐 [formtastic.de](https://formtastic.de) · 📱 [App öffnen](https://app.formtastic.de) · 🇬🇧 [English version](./README.md)

---

Mit Formtastic erfassen Servicetechniker, Handwerker und Außendienstmitarbeiter Protokolle, Checklisten und Berichte direkt vor Ort – per Smartphone oder Tablet, inklusive Foto, Unterschrift und GPS. Das fertige PDF landet automatisch beim Empfänger. Kein Abtippen, kein Doppelaufwand: **Der Bericht ist fertig, bevor der Techniker wieder im Auto sitzt.**

**100% offline-fähig · DSGVO-konform · Made in München.**

## Was Formtastic kann

- 🧱 **Drag-&-Drop Formularbaukasten** — Formulare ohne Programmierung erstellen
- 📄 **PDF / Word / Excel Export** im eigenen Corporate Design
- 🌍 **Mehrsprachige Formulare** — eine Vorlage, mehrere Sprachen
- 🔌 **Schnittstellen** — Google Drive, Google Sheets, SFTP/FTP, Webhooks, CSV-Export
- 📶 **100% offline** — funktioniert auch ohne Netz, synchronisiert später
- 🤝 **Persönlicher Einrichtungs-Service** inklusive

## Für wen

Betriebe ab 5 Mitarbeitern im Außendienst: Handwerk, Servicetechnik, Facility Management, Gebäudereinigung, Hausverwaltung, Logistik, Produktion, medizinische Dienstleister.

[→ Branchenlösungen ansehen](https://formtastic.de/)

## App herunterladen

<a href="https://play.google.com/store/apps/details?id=com.infopunks.formastic"><img src="https://formtastic.de/wp-content/uploads/2024/09/Get_it_on_Google_Play.png" alt="Bei Google Play laden" height="40"></a>
<a href="https://apps.apple.com/us/app/formtastic-forms/id1241888072"><img src="https://formtastic.de/wp-content/uploads/2024/09/Download_on_the_App_Store.png" alt="Im App Store laden" height="40"></a>

## Hilfe & Ressourcen

- 📖 **Dokumentation:** [formtastic.de/dokumentation](https://formtastic.de/dokumentation/)
- ❓ **FAQ:** [formtastic.de/faq](https://formtastic.de/faq/)
- 🛟 **Support-Portal:** [support.formtastic.de](https://support.formtastic.de/)
- 🆘 **Fragen zu diesem Repository?** [SUPPORT.de.md](./SUPPORT.de.md)
- ⭐ **Erfolgsgeschichten:** [formtastic.de/erfolgsgeschichten](https://formtastic.de/erfolgsgeschichten/)
- 💶 **Preise:** [formtastic.de/preise](https://formtastic.de/preise/)

---

# Entwickler-Referenz

In diesem Repository veröffentlichen wir technische Referenzen für die Integration mit Formtastic. Der Quellcode der Anwendung ist nicht öffentlich.

## Massen-Export (CSV)

Exportiert Formulareinreichungen gesammelt als CSV-Datei. Die Authentifizierung erfolgt per Query-Parameter.

```
GET https://app.formtastic.de/export-bulk-external.csv?token=<token>&template=<id>&state=<state>&from=<YYYY-MM-DD>&to=<YYYY-MM-DD>
```

**Query-Parameter**

| Parameter | Typ | Pflicht | Beschreibung |
|---|---|---|---|
| `token` | string | ja | Dein API-Authentifizierungs-Token |
| `state` | string | ja | Status der Formulardaten: `sent_and_saved`, `filled`, `draft` oder `rejected` |
| `template` | integer | ja | ID der Formularvorlage |
| `from` | string | ja | Startdatum (`YYYY-MM-DD`) |
| `to` | string | ja | Enddatum (`YYYY-MM-DD`) |
| `transposed` | boolean | nein | Transponiertes Layout verwenden (Standard `false`) |
| `onlyOnlineForms` | boolean | nein | Nur Online-Formulareinreichungen einschließen (Standard `false`) |

**Antwort** — `200 OK` — CSV-Datei zum Download.

**Fehler**

| Code | Bedeutung |
|---|---|
| 400 | Fehlende oder ungültige Parameter |
| 401 | Ungültiges Token |
| 403 | Export-Funktion in deinem Tarif nicht verfügbar |

---

## Webhooks

Mit Webhooks sendet Formtastic Ereignisse zu Formulareinreichungen nahezu in Echtzeit an deine eigenen Systeme. Wenn sich der Status einer Einreichung ändert (z. B. wenn sie gespeichert oder zugewiesen wird), sendet Formtastic einen HTTP `POST`-Request an eine von dir festgelegte URL.

### Überblick

- **Transport**: HTTP `POST` mit `Content-Type: application/json`.
- **Auslöser**: Eine Statusänderung an einer Formulareinreichung einer Vorlage, für die ein Webhook konfiguriert ist.
- **Body**: Ein JSON-Objekt mit Informationen zu Formular, Einreichung und dem auslösenden Benutzer.
- **Authentifizierung**: Jeder Webhook hat ein eigenes Secret-Token, das mit jedem Request mitgesendet wird.
- **Retries**: Fehlgeschlagene Zustellungen werden automatisch wiederholt; dein Endpoint muss daher idempotent sein.

### Payload-Schema

Eine Zustellung ist ein einzelner `POST` an deine URL mit folgendem JSON-Body:

```json
{
  "form": {
    "id": 10631,
    "name": "All fields"
  },
  "run_id": "9543c085-0c19-431f-b654-0b0e970063d1",
  "form_data": {
    "id": 435666,
    "data": {
      "input_text_with_default": "Default",
      "input_integer_1_to_5": 1,
      "check": false,
      "...": "Feldwerte mit dem Platzhalternamen des Feldes als Schlüssel"
    },
    "state": "sent_saved",
    "create_time": "2026-04-27T13:50:49.127075+00:00",
    "update_time": "2026-04-27T13:50:49.382868+00:00"
  },
  "timestamp": "2026-04-27T13:50:49.400034+00:00",
  "triggered_by": {
    "id": 83,
    "email": "user@example.com"
  },
  "lifecycle_state": "sent_saved"
}
```

### Feldreferenz

| Feld | Typ | Beschreibung |
|---|---|---|
| `form.id` | number | Die ID der Formularvorlage. |
| `form.name` | string | Der Name der Vorlage zum Zeitpunkt der Auslösung. |
| `run_id` | string | Eine UUID, die diese Zustellung identifiziert. Wird für **Idempotenz** verwendet. |
| `form_data.id` | number | Die ID der Einreichung. |
| `form_data.data` | object | Die Feldwerte der Einreichung, mit dem Platzhalternamen des Feldes als Schlüssel. Die Struktur hängt vom jeweiligen Formular ab. |
| `form_data.state` | string | Der Lifecycle-Status der Einreichung zum Zeitpunkt der Auslösung. |
| `form_data.create_time`, `form_data.update_time` | string | ISO-8601-Zeitstempel der Einreichung. |
| `timestamp` | string | ISO-8601-Zeitstempel, wann der Webhook ausgelöst wurde. |
| `triggered_by` | object | Der Benutzer, der die Statusänderung ausgelöst hat (id, email). |
| `lifecycle_state` | string | Gleicher Wert wie `form_data.state`; zusätzlich auf oberster Ebene bereitgestellt. |

### Lifecycle-Status

| Status | Bedeutung |
|---|---|
| `inbox` | Neu eingegangen, noch nicht bearbeitet. |
| `in_work` | Wird von einer zugewiesenen Person bearbeitet. |
| `sent_assigned` | An eine bestimmte Person zugewiesen gesendet. |
| `sent_saved` | Gesendet und gespeichert. |
| `removed` | Gelöscht / archiviert. |

### Authentifizierung

Jeder Webhook hat ein eigenes **Secret**, das bei der Erstellung generiert wird. Dieses Secret wird mit jeder Zustellung mitgesendet, damit dein Empfänger die Echtheit prüfen kann.

Dein Empfänger sollte:

1. Das Secret aus dem eingehenden Request auslesen.
2. Es (in konstanter Zeit) mit dem Secret vergleichen, das du bei der Webhook-Konfiguration gespeichert hast.
3. Den Request mit `401` ablehnen, wenn die Werte nicht übereinstimmen.

Wenn das Secret kompromittiert wurde, kann es auf der Detailseite des Webhooks neu generiert werden. Das vorherige Secret wird dabei sofort ungültig.

### Antwort auf eine Zustellung

- **2xx** — Die Zustellung gilt als **erfolgreich**. Es erfolgen keine weiteren Versuche.
- **Alles andere oder ein Netzwerkfehler** — Die Zustellung gilt als **fehlgeschlagen** und Formtastic wiederholt sie.

Antworte so schnell wie möglich. Aufwändige Verarbeitung sollte **nach** der Bestätigung erfolgen (z. B. über einen Background-Job).

### Retries und Idempotenz

Wenn eine Zustellung fehlschlägt, plant Formtastic einen weiteren Versuch. Alle Versuche zu demselben Ereignis teilen sich dieselbe `run_id`, dein Empfänger kann denselben Payload also mehrfach erhalten.

- **Dedupliziere per `run_id`.** Speichere die `run_id` jeder erfolgreich verarbeiteten Zustellung und überspringe Duplikate.
- **Mache deine Verarbeitung idempotent.** Auch ohne explizite Deduplizierung sollten deine nachgelagerten Effekte mehrfaches Ausführen vertragen.

Ein Lauf wird wiederholt, bis ein Versuch erfolgreich ist, das Retry-Budget aufgebraucht ist (der Lauf endet dann im Status `failed`) oder ein Administrator ihn abbricht.

### Best Practices

- **Nutze HTTPS** mit einem gültigen Zertifikat.
- **Prüfe das Secret** bei jedem Request.
- **Antworte schnell und verarbeite im Hintergrund**, damit die Zustellungen innerhalb des Timeouts liegen.
- **Sei idempotent.** Retries sind der Normalfall, nicht die Ausnahme.
- **Logge `run_id` und `timestamp`** jeder akzeptierten Zustellung.
- **Rotiere das Secret**, wenn es möglicherweise kompromittiert wurde, und aktualisiere deinen Empfänger entsprechend.

---

## Kontakt

**Formtastic GmbH**
Amalienstraße 77, 80799 München
📞 +49 89 7168020-90 · ✉️ [info@formtastic.de](mailto:info@formtastic.de)
🔗 [LinkedIn](https://www.linkedin.com/company/formtastic-gmbh/)

[Impressum](https://formtastic.de/impressum/) · [Datenschutz](https://formtastic.de/datenschutzerklaerung/) · [AGB](https://formtastic.de/agb/)
