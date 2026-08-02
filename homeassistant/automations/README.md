# Home Assistant Automationen

## rolladen_schlafzimmer_schichterkennung.yaml

Rolladensteuerung Schlafzimmer mit automatischer Schichterkennung über den
Standort von `person.nancy_hiller`.

Einfügen entweder als eigener Eintrag in `automations.yaml` (dann mit `- ` vor
`alias:` einrücken) oder über die UI: *Einstellungen → Automatisierungen →
Automatisierung → ⋮ → In YAML bearbeiten* und den Inhalt der Datei einfügen.

### Warum der Schichtmodus vorher nicht gesetzt wurde

1. **Falscher Zonen-State (Hauptursache).** Der Trigger lautete
   `entity_id: person.nancy_hiller` / `to: arbeit`. Der State einer Person ist
   in einer Zone aber der **Anzeigename** der Zone (also z.B. `Arbeit`), nicht
   die Entity-ID `arbeit`. Der Trigger hat deshalb nie ausgelöst, und die drei
   Schichterkennungs-Zweige liefen nie an.
   Fix: echter `zone`-Trigger (`event: enter`) plus State-Trigger mit
   case-insensitivem Vergleich gegen den Anzeigenamen von `zone.arbeit` — so
   funktioniert es unabhängig davon, wie die Zone benannt ist.

2. **`mode: restart`.** Bei elf Triggern in einer Automation bricht jeder neue
   Trigger den gerade laufenden Ablauf ab. Fix: `mode: queued` mit `max: 10`.

3. **Neustart-Fehlauslösung.** Nach einem HA-Neustart wechselt die Person von
   `unknown` auf ihren Zonen-State. Ohne Filter hätte das nachts fälschlich
   "Nachtschicht" gesetzt. Fix: `not_from: [unknown, unavailable]` und Prüfung,
   dass der vorherige State *nicht* schon "Arbeit" war (echtes Betreten statt
   Attribut-Update).

4. **Zeitlücke 05:00–07:00.** Nachtschicht-Fenster endete um 05:00, das
   Frühschicht-Fenster begann erst um 07:00 — eine Ankunft bei der Arbeit
   dazwischen (typischer Frühschichtbeginn) setzte gar keinen Modus. Das
   Frühschicht-Fenster beginnt jetzt um 05:00.

### Voraussetzungen

- `input_select.schichtmodus_schlafzimmer` mit den Optionen
  `keine, frueh, spaet, nacht` (Startwert `keine`)
- `input_boolean.urlaubsmodus_schlafzimmer`
- `zone.arbeit` muss existieren
- `person.nancy_hiller` braucht einen **GPS-fähigen** device_tracker
  (Companion App). Router-/Bluetooth-Tracker liefern nur `home`/`not_home` und
  können Zonen grundsätzlich nicht erkennen.

### Test ohne Wartezeit

*Entwicklerwerkzeuge → Template* zum Prüfen des Zonennamens:

```jinja
{{ states('person.nancy_hiller') }}
{{ state_attr('zone.arbeit', 'friendly_name') }}
```

Beides sollte beim Aufenthalt bei der Arbeit (bis auf Groß-/Kleinschreibung)
übereinstimmen. Danach in *Entwicklerwerkzeuge → Zustände* den State von
`person.nancy_hiller` testweise auf den Zonennamen setzen und im Trace der
Automation prüfen, welcher Zweig greift.
