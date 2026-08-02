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
3. **Letztes Netz: Heimkehr zwischen 04:00 und 09:00** schließt den Rolladen
   auch dann, wenn beide vorherigen Wege versagt haben, und setzt `nacht` nach
   (entspricht der alten Homee-Logik). Damit greifen auch Aufwach-Erkennung und
   das 16:00-Netz. Fenster über `heimkehr_nacht_von` / `heimkehr_nacht_bis`.

Sechs Zweige können den Rolladen öffnen — alle sind bei Modus `nacht`
blockiert; nur die Aufwach-Erkennung und das 16:00-Netz kommen durch.

#### Sperrfrist für die Aufwach-Erkennung

`binary_sensor.bewegung_wifa_occupancy` 10 Minuten durchgehend `on` galt bisher
jederzeit als Aufwachen. Der Melder sitzt unten im Flur und ist als
Aufwach-Signal gut gewählt — ein Toilettengang kommt da nicht vorbei, dafür
müsste sie erst die Treppe runter. Kritisch ist nur die **erste Stunde nach der
Heimkehr**: um 06:30 kommt sie an, räumt auf, macht sich fertig — und genau
dieser Weg führt durch den Flur. Nach 10 Minuten wäre der Rolladen wieder
hochgefahren, direkt nachdem er geschlossen hat.

Bewegung zählt deshalb erst als Aufwachen, wenn beides gilt:

| Sperre | Standard | Zweck |
| --- | --- | --- |
| `aufwach_mindestschlaf` | 3 h zuhause | deckt die Ankunfts-/Aufräumphase ab |
| `aufwach_fruehestens` | 09:00 | Plausibilität, falls sie deutlich früher heimkommt |

Bei Heimkehr um 06:30 ist die Erkennung damit ab **09:30** scharf. Die Sperre
hängt bewusst an der Heimkehr statt an einer festen Uhrzeit — so blockiert sie
nicht unnötig bis in den Nachmittag, wenn sie mal früher aufsteht. Gemessen wird
am letzten State-Wechsel von `person.nancy_hiller`; nach einem HA-Neustart
verschiebt sich der Bezugspunkt nach hinten, also in die sichere Richtung.

> **Bitte nach dem Test prüfen:** wie lange bleibt der Melder nach der letzten
> Bewegung noch `on` (bei Zigbee-Meldern oft `occupancy_timeout`)? Liegt der
> Wert bei 10 Minuten oder darüber, ist „10 Minuten durchgehend" praktisch
> bedeutungslos — dann reicht eine einzelne Bewegung, und die Sperrfrist ist der
> einzige Schutz. In dem Fall die 10 Minuten im Trigger `wake_motion` erhöhen.

> **Bekannte Lücke (bewusst offen):** der Trigger feuert nur in dem Moment, in
> dem die Bewegung 10 Minuten erreicht. Steht sie um 09:20 auf — also vor Ablauf
> der Sperrfrist — feuert er einmal, wird abgelehnt und kommt nicht wieder; dann
> öffnet erst das 16:00-Netz. Eine regelmäßige Nachprüfung würde das schließen,
> aber auch bedeuten, dass ein hängender Melder den Rolladen öffnet. Nach der
> Priorisierung (zu früh öffnen ist fatal, zu spät nicht) bleibt es beim
> Trigger.

### Mehrere Tracker (iPhone + Tesla)

`person.nancy_hiller` wird aus mehreren Trackern gebildet. Home Assistant setzt
den Person-State auf den **zuletzt aktualisierten** GPS-Tracker. Bei dieser
Konstellation heißt das:

- das iPhone hat im Gebäude kein Signal — seine letzte Meldung wird alt oder
  springt beim Wiederverbinden;
- das Auto steht nicht immer dabei.

Zwei Konsequenzen sind eingebaut:

1. **Anwesenheitsprüfung schaut auf alle Tracker**, nicht auf den Person-State.
   Meldet *irgendein* Tracker die Arbeitszone, gilt sie als anwesend — der
   Person-State kann auf `not_home` stehen, während der Tesla noch korrekt
   `Arbeit` meldet. Ausgelesen über das Attribut `device_trackers` der Person,
   funktioniert also ohne fest verdrahtete Entity-IDs.
2. **Während des Nachtmodus wird eine gemeldete Ankunft bei der Arbeit
   ignoriert.** Sonst der fatale Fall: sie verlässt um 06:15 das Gebäude, das
   iPhone bekommt wieder Signal und meldet die Arbeitszone nach — das wäre eine
   „Ankunft" um 06:15 und würde den Modus von `nacht` auf `frueh` setzen. Der
   Rolladen wäre um 07:00 hochgefahren. Echte Schichtwechsel verlieren dadurch
   nichts, weil `nacht` spätestens um 16:00 zurückgesetzt wird — lange vor der
   nächsten Anfahrt.

Springt der Tracker trotzdem so ungünstig, dass gar keine Schicht erkannt wird,
greift das Heimkehr-Netz: Ankunft zuhause zwischen 04:00 und 09:00 schließt den
Rolladen in jedem Fall.

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
