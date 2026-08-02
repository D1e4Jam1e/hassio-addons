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

4. **Zeitfenster lagen neben der Ankunft.** Der Trigger feuert beim *Betreten*
   der Zone, also zum Schichtbeginn — bei einer Nachtschicht 22:00–06:00 also
   gegen 21:45. Die Fenster (`23:00-05:00`, `14:00-22:00`, `07:00-12:00`)
   beschrieben aber die Schicht*dauer*, nicht die Ankunft. Selbst mit korrektem
   Zonen-State hätte die Nachtschicht-Erkennung nie gegriffen, weil 21:45 vor
   dem Fensterbeginn 23:00 liegt.

### Neue Erkennungslogik

Statt starrer Fenster wird die Ankunftszeit dem **zeitlich nächsten
Schichtbeginn** zugeordnet (Toleranz 3 h):

| Ankunft bei der Arbeit | Modus |
| --- | --- |
| 03:00–09:00 | `frueh` (Start 06:00) |
| 11:00–17:00 | `spaet` (Start 14:00) |
| 19:00–01:00 | `nacht` (Start 22:00) |
| 01:00–03:00, 09:00–11:00, 17:00–19:00 | unverändert + Logbuch-Eintrag |

Damit sind Abweichungen von einer halben oder ganzen Stunde egal, und es gibt
keine Fenstergrenze mehr, an der die Erkennung stillschweigend ausfällt. Passen
die Schichtzeiten nicht, reicht es, oben in der Automation die Variablen
`schicht_starts` (Beginn je Schicht, Dezimalstunden — `5.5` = 05:30) und
`schicht_toleranz` anzupassen.

Angenommen wurde das klassische 3-Schicht-System 06:00 / 14:00 / 22:00, passend
zur genannten Nachtschicht 22:00–06:00.

### Die kritische Richtung: Rolladen öffnet nach der Nachtschicht zu früh

Zu spät öffnen ist harmlos (Rolladen von Hand hoch), zu früh öffnen weckt sie
mitten im Tagschlaf. Alle Regeln sind deshalb in diese Richtung ausgelegt — ein
zu Unrecht gesetzter Nachtmodus kostet nichts, er wird spätestens um 16:00
zurückgesetzt. Drei Schichten Absicherung:

1. **Ankunft bei der Arbeit** (~21:45) setzt `nacht`.
2. **Anwesenheitsprüfung** alle 30 Min. zwischen 23:00 und 05:00 — fängt es auf,
   wenn das Ankunfts-Ereignis verlorenging.
3. **Letztes Netz: Heimkehr zwischen 04:00 und 08:00** schließt den Rolladen
   auch dann, wenn beide vorherigen Wege versagt haben, und setzt `nacht` nach
   (entspricht der alten Homee-Logik). Damit greifen auch Aufwach-Erkennung und
   das 16:00-Netz. Fenster über `heimkehr_nacht_von` / `heimkehr_nacht_bis`.

Sechs Zweige können den Rolladen öffnen — alle sind bei Modus `nacht`
blockiert; nur die Aufwach-Erkennung und das 16:00-Netz kommen durch.

#### Aufwach-Erkennung erst ab 11:00

`binary_sensor.bewegung_wifa_occupancy` 10 Minuten durchgehend `on` galt bisher
jederzeit als Aufwachen. Das ist genau in der kritischen Richtung gefährlich:
sie kommt um 06:30 nach Hause, räumt auf und geht durch den Flur — nach 10
Minuten wäre der Rolladen wieder hochgefahren, direkt nachdem er geschlossen
hat. Bewegung zählt deshalb erst ab `aufwach_fruehestens` (11:00) als
Aufwachen. Auch ein kurzer Gang zur Toilette und andere Personen im Flur fallen
damit raus.

Preis dafür: steht sie um 10:30 auf, bleibt es dunkel bis 16:00 oder bis sie von
Hand öffnet. Nach deiner Priorisierung der bessere Fehler.

> **Bitte prüfen:** wie lange bleibt `binary_sensor.bewegung_wifa_occupancy`
> nach der letzten Bewegung noch `on` (bei Zigbee-Meldern oft
> `occupancy_timeout`)? Liegt der Wert bei 10 Minuten oder darüber, ist die
> Bedingung „10 Minuten durchgehend" praktisch bedeutungslos — dann reicht eine
> einzelne Bewegung, und die Zeitsperre ab 11:00 ist der einzige Schutz. In dem
> Fall die 10 Minuten im Trigger `wake_motion` deutlich höher setzen.

### Zweiter Erkennungsweg: Anwesenheitsprüfung

Das Betreten der Zone ist ein **einmaliges Ereignis**. Geht es verloren (GPS
ungenau, Handy offline, HA gerade neu gestartet), wäre der Modus für den ganzen
Tag verloren. Deshalb prüft ein `time_pattern`-Trigger alle 30 Minuten, ob sie
sich gerade im **Kern** einer Schicht bei der Arbeit aufhält:

| Kernzeit (Schichtbeginn +1 h bis +7 h) | Modus |
| --- | --- |
| 07:00–13:00 | `frueh` |
| 15:00–21:00 | `spaet` |
| 23:00–05:00 | `nacht` |

Diese Prüfung setzt den Modus **nur, wenn er noch `keine` ist** — eine bereits
erkannte Schicht wird nie überschrieben. Damit setzt eine Überstunde nach der
Frühschicht nicht versehentlich `spaet`. Puffer über `kern_von`/`kern_bis`
einstellbar.

Beide Wege sind idempotent und ergänzen sich: die Ankunft erkennt sofort und
präzise, die Anwesenheitsprüfung fängt auf, was durchgerutscht ist (und schreibt
dann einen Logbuch-Eintrag).

5. **`frueh` blieb ewig stehen.** Der Modus `frueh` wurde nirgends
   zurückgesetzt. Er hat zwar keine Rolladen-Wirkung, blieb aber bis zur
   nächsten erkannten Schicht stehen. Er wird jetzt um **21:30** zurückgesetzt —
   bewusst nicht schon beim morgendlichen Öffnen, denn dann würde die
   Anwesenheitsprüfung bei Frühschicht-Überstunden ab 15:00 auf `keine` treffen
   und fälschlich `spaet` setzen. 21:30 liegt nach dem Ende der
   Spätschicht-Kernzeit und vor der Anfahrt zur Nachtschicht.

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
