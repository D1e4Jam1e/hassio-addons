# Home Assistant Automationen

Fünf Automationen, die sich `cover.schlafzimmer` und
`switch.153931628878753_power` teilen, plus eine unabhängige
Wohnzimmer-Automation:

| Datei | Rolle |
| --- | --- |
| `rolladen_schlafzimmer_schichterkennung.yaml` | Schichterkennung, Tagschlaf, Sonnenauf-/-untergang |
| `verschattung_nord_ost.yaml` | temperaturbasierter Sonnenschutz, 5 Räume, mit Bewölkt+kühl-Override |
| `klima_schlafzimmer_ein.yaml` | Klimagerät ein |
| `klima_schlafzimmer_aus.yaml` | Klimagerät aus |
| `verschattung_west.yaml` | helligkeitsbasierter Sonnenschutz, Küche + HWR |
| `schrankbeleuchtung.yaml` | Schrankbeleuchtung Wohnzimmer, gekoppelt an Apple TV |

## schrankbeleuchtung.yaml

Schaltet die beiden Schrank-Lichter im Wohnzimmer mit
`media_player.wohnzimmer_apple_tv_wohnzimmer` ein und wieder aus. Beide
Richtungen in einer Automation statt in getrennten Ein-/Aus-Dateien (wie bei
der Klimaanlage), weil sich Media Player und Lichter zwischen beiden Zweigen
nicht unterscheiden - eine zweite Datei hätte nur dieselben Entity-IDs
dupliziert.

Ein `media_player.turned_off`-Trigger allein hätte die vorhandene
`light.turn_on`-Aktion erneut ausgelöst, da beide Trigger dieselbe
Automation auslösen. Die beiden Trigger tragen deshalb IDs (`an`/`aus`), und
ein `choose` wählt anhand der Trigger-ID zwischen `light.turn_on` und
`light.turn_off`.

### An/Aus über `state` statt über purpose-specific Trigger (Fix 31.08.)

Ursprünglich liefen An/Aus über `media_player.turned_on`/`turned_off`.
Traces vom 31.08. zeigten, dass diese Trigger nach dem ersten Auslösen
aufgehört haben zu feuern: auf 15:13:23/15:13:50 (An/Aus, beide korrekt im
Trace sichtbar) folgten laut Aktivitätsprotokoll vier weitere echte
Aus-/Einschaltwechsel des Apple TV (16:51/16:52 und 16:57/16:58 Uhr) - keiner
davon erzeugte einen Trace, die Automation hat also gar nicht reagiert. Der
Apple TV durchläuft beim Reconnect kurz Zwischenzustände (`unknown`/`idle`),
was den purpose-specific Triggern offenbar die Spur verliert, welcher
Zustand vorher als "aus" galt.

Ersetzt durch den klassischen `state`-Trigger (`from: "off"` für An,
`to: "off"` für Aus) - der reagiert stumpf auf den Zustandsübergang selbst,
statt intern eine eigene "ist gerade an/aus"-Logik zu pflegen, und kennt
das Problem deshalb nicht.

(Der einzige weitere Trace aus dieser Zeit hatte `trigger: null` - eine
manuelle Ausführung über die UI, bei der kein `trigger.id` gesetzt wird und
deshalb erwartungsgemäß kein `choose`-Zweig zutrifft. Das ist kein Bug,
sondern der bereits besprochene Unterschied zwischen "Automatisierung
ausführen" und einem echten Trigger-Ereignis.)

### light.kugeln nur bei Dunkelheit

Beim Einschalten bekommt `light.kugeln` zusätzlich den Effekt "TV time" -
aber nur, wenn die Sonne unter dem Horizont steht (`sun.elevation below 0`),
sonst wird es explizit ausgeschaltet.

Damit das auch nachreagiert, wenn der Apple TV schon läuft und sich
währenddessen die Tageszeit ändert, gibt es zwei weitere Trigger
(`dunkel`/`hell`) auf `sun.sun`, Attribut `elevation`, jeweils mit
Schwelle 0. Beide sind zusätzlich an die Bedingung "Apple TV ist gerade an"
(`state` von `media_player.wohnzimmer_apple_tv_wohnzimmer` ist nicht `off`)
gebunden - sonst würde bei jedem Sonnenauf-/-untergang `light.kugeln`
angefasst, auch wenn gar nicht ferngesehen wird.

`numeric_state` statt eines purpose-specific Triggers, weil eine
Elevation-Schwelle von exakt 0° kein benannter Standardfall ist (siehe
Diskussion zu bürgerlicher/nautischer/astronomischer Dämmerung oben) -
dafür gibt's keine fertige Abkürzung, nur die generische Ebene.

## verschattung_west.yaml

### „HWR fährt zu weit zu" — die Korrektur konnte den Fall nicht sehen

Die Drift-Absicherung wartet 20 Sekunden und prüft dann nach. Geprüft wurde
aber `is_state(cover, 'closed')` — also **nur exakt 0%**. Fährt der HWR über
sein Ziel hinaus und landet bei 25%, ist sein State `open`, und die Korrektur
lief nie an. Genau der beobachtete Fall.

Verschärft dadurch, dass der 10-Minuten-Trigger als zweites Netz ausfällt: er
greift nur, wenn die Position noch `> 80` ist. Ein auf 25% gedrifteter Rolladen
ist das nicht — er wird also von keiner der beiden Absicherungen mehr angefasst
und bleibt bis zum nächsten Öffnen dort stehen.

Geprüft wird jetzt die tatsächliche Position gegen die Zielposition
(`current_position < ziel − 5`). Durchgespielt: bei Ziel 78 korrigieren 0, 15,
25, 45 und 50; bei Ziel 50 korrigieren 0, 15 und 25.

Das ist eine Reparatur der Symptombehandlung — die Ursache liegt tiefer.

### Ist die Drift überhaupt noch da?

Derselbe Fehler steckte auch in `verschattung_nord_ost.yaml` (dort in fünf
Zweigen) und ist mitkorrigiert.

Beide Automationen schreiben jetzt einen Logbuch-Eintrag, wenn die Korrektur
anläuft — mit Ziel- und Ist-Position und `entity_id` des Rolladens, also über
die Entität filterbar:

> **Verschattung West** — Positions-Drift erkannt (Ziel 78%, tatsächlich 25%) —
> Zielposition erneut angefahren.

### Die Ursache liegt in den Aktoren, nicht im Controller

Die Absicherung bleibt, unabhängig vom Controller: die **EnOcean-Aktoren selbst
arbeiten mit Laufzeiten**, sie haben keine Positionssensoren. Die gemeldete
Position ist also immer eine Schätzung aus „wie lange bin ich gefahren", nicht
eine Messung. Ein Wechsel des Controllers — homee, wibutler, direkter
Bus-Abgriff — ändert daran nichts.

Zwei Konsequenzen, die man beim Lesen der Logbuch-Einträge kennen muss:

- **Sie zeigen nur, was der Aktor zugibt.** Meint er, er stehe auf 78%, während
  er physisch tiefer steht, sieht Home Assistant davon nichts. Die Einträge sind
  eine Untergrenze, kein vollständiges Bild.
- **Der Fehler summiert sich nicht auf.** Motoren dieser Bauart erkennen die
  Endlagen und setzen ihre Schätzung dort zurück. Weil beide Verschattungen
  praktisch immer aus der oberen Endlage heraus verschatten (der periodische
  Zweig verlangt sogar Position > 80), ist der beobachtete Versatz der Fehler
  *einer einzelnen Fahrt* — also ein systematischer Kalibrierfehler, kein
  wachsender Drift.

Dass der HWR reproduzierbar zu weit fährt, passt genau dazu. Der wirksame
Hebel ist die Laufzeit im Aktor; die Nachkorrektur hier fängt nur ab, was
danach noch danebengeht.

### Nach dem Wechsel auf wibutler prüfen

Die Drift bleibt (siehe oben), aber die Anbindung wechselt — und die gesamte
Positionslogik hängt an zwei Voraussetzungen, die integrationsabhängig sind:

1. **Attribut `current_position`** muss vorhanden sein. Fehlt es, hätte die
   Drift-Prüfung jeden Fahrbefehl als „auf 0% gedriftet" gewertet und das
   Logbuch mit Falschmeldungen geflutet — genau die Messung, um die es hier
   geht. Die Prüfung ist deshalb jetzt None-sicher: fehlt das Attribut, greift
   nur noch der alte `closed`-Fall. Durchgespielt bei Ziel 78: Attribut fehlt →
   keine Korrektur, 0 und 25 → Korrektur, 73 und 78 → keine.
2. **`cover.set_cover_position`** muss unterstützt sein. Bietet die
   Matter-Anbindung nur Auf/Zu/Stopp, funktioniert keine der beiden
   Verschattungen mehr — dann bräuchte es eine andere Lösung als Zielpositionen.

Beides steht in *Entwicklerwerkzeuge → Zustände* beim jeweiligen `cover`:
`current_position` in den Attributen, `supported_features` mit gesetztem
Positions-Bit (4).

Ebenfalls prüfen: ob die Entity-IDs den Integrationswechsel überlebt haben. Alle
fünf Automationen sprechen `cover.schlafzimmer`, `cover.kuche`, `cover.hwr`,
`cover.badezimmer`, `cover.wc`, `cover.wohnzimmer` und `cover.terrasse` direkt
an — legt die neue Anbindung sie als `..._2` an, laufen die Automationen ins
Leere, ohne einen Fehler zu werfen.

### HWR konnte nie wieder geöffnet werden

Der Schließen-Zweig nimmt HWR bewusst von der „überspringe geschlossene
Rolladen"-Regel aus (`respect_closed: false`), weil 0% wegen der
Zwangsbelüftung nie zulässig ist. Der **Öffnen**-Zweig hatte diese Ausnahme
nicht — er übersprang jeden Rolladen mit State `closed`, also auch den HWR.

War der HWR einmal auf 0% gelandet, konnte ihn diese Automation damit nie wieder
öffnen: der Öffnen-Zweig überspringt ihn, und der Schließen-Zweig läuft nur bei
Hitze und Sonne. Bei einem Rolladen, für den 0% laut eigener Beschreibung nie
zulässig ist, ist das der unangenehmere der beiden Fehler. HWR ist jetzt auch im
Öffnen-Zweig ausgenommen.

### Zielposition HWR 78%

Wie gewünscht weiter offen (Wunsch: 75–80%). Wert steht in der Variable
`positionen` zusammen mit der Küche (50%); die Toleranzprüfung „steht schon nah
genug" bedient sich aus derselben Stelle, damit die beiden nicht wieder
auseinanderlaufen.

### `mode: queued` statt `restart`

Wie bei Nord/Ost: bei `restart` bricht jeder Trigger die 20-Sekunden-Korrektur
ab. Hier wiegt das schwerer, weil zwei Rolladen nacheinander abgearbeitet werden
— ein Durchlauf dauert über 40 Sekunden.

## klima_schlafzimmer_ein.yaml / _aus.yaml

### Rolladen-Check robuster, Kriterium unverändert

„Rolladen unten" bleibt das Signal für „es wird geschlafen" — und das bewusst:
es gilt für beide Schlafenden, unabhängig von Schichten. Eine Sperre über
`schichtmodus_schlafzimmer` wurde deshalb wieder verworfen; sie hätte nur die
Nachtschicht-Tagschläferin geschützt und den zweiten Schläfer in der normalen
Nacht gar nicht.

Ergänzt sind nur die zwei Zustände, in denen der State `closed` noch nicht bzw.
nicht mehr anliegt, obwohl der Rolladen faktisch unten ist. Beide sperren
zusätzlich, geben nie etwas frei:

| Zustand | Prüfung |
| --- | --- |
| `closing` | State-Bedingung um `closing` erweitert — während der Fahrt ist der State weder `open` noch `closed`, unabhängig von der Fahrzeit-Kalibrierung |
| Restdrift | `current_position >= 20` — bei 2% wäre der State `open`; fehlt das Attribut, gilt 0 (gesperrt) |

Die Grenze bei 20% schneidet nichts ab: der Rolladen steht praktisch nur auf 0,
78 oder 100. Durchgespielt: 0/2/19 sperren, ab 20 gibt frei.

Relevant ist das vor allem, weil die Automation nach dem Einschalten auch
`set_cover_position: 78` fährt — aus einem Fehlstart würde also Lärm **und**
Licht.

### Mindestlaufzeit gegen Takten

Die Ausschalt-Automation prüft über den 10-Minuten-Trigger nur den
Momentanwert (`numeric_state below 22`, ohne `for`) — ein einzelner Messwert
unter 22°C genügte also. Zusammen mit den 20 Minuten Sperre auf der
Einschaltseite konnte daraus ein Takten im 20-Minuten-Raster werden. Da das
Schalten die Netzspannung kappt, greift der geräteeigene Verdichterschutz nicht.
Ergänzt: das Gerät muss mindestens **15 Minuten** gelaufen sein.

### Bekannte Kopplung

Die Zielposition **78** steht an zwei Stellen: in der Variable `positionen` der
Verschattung und als `set_cover_position` in der Einschalt-Automation.
Automationsübergreifende Variablen gibt es in Home Assistant nicht — wird die
Verschattungsposition des Schlafzimmers geändert, muss sie an beiden Stellen
nachgezogen werden. In beiden Dateien vermerkt.

### Doku-Abweichungen korrigiert

- Einschalten: die Beschreibung nannte `cover.rolladen_schlafzimmer`, geprüft
  wird `cover.schlafzimmer`.
- Ausschalten: die Beschreibung nannte „höchstens 2K", das Template rechnet mit
  `<= 3`. Der zweite Absatz derselben Beschreibung nannte bereits 3K.

## verschattung_nord_ost.yaml

Temperaturbasierter Sonnenschutz für die fünf Nord-/Ost-Räume, seit 22.08.2026
mit Bewölkt+kühl-Override (siehe unten). Hier liegt sie, weil sie sich mit der
Schichterkennung denselben Rolladen teilt.

### Koordination mit der Schichterkennung

Beide Automationen fahren `cover.schlafzimmer`. Im Protokoll sichtbar geworden
am 3. August: um **16:00:00** öffnete das Sicherheitsnetz der Schichterkennung
den Rolladen, um **16:00:01** zog die Verschattung ihn wieder herunter.

Die Verschattung überspringt Rolladen mit State `closed` bereits in allen
Zweigen — der Tagschlaf war also im Normalfall geschützt. Der Schutz hängt
allerdings daran, dass der Rolladen **exakt** auf Position 0 steht. Genau das
ist bei der in der Automation dokumentierten Positions-Drift nicht garantiert:
landet er auf 2%, ist sein State `open` und die Verschattung hätte ihn mitten im
Tagschlaf hochgefahren.

Ergänzt wurde deshalb: `cover.schlafzimmer` wird übersprungen, solange
`input_select.schichtmodus_schlafzimmer` auf `nacht` steht — in allen vier
Zweigen, die diesen Rolladen anfassen.

Bewusst nur `nacht`. `spaet` steht von der Abfahrt am frühen Nachmittag bis zum
nächsten Morgen und würde die Verschattung den halben Tag aussperren, `frueh`
sogar bis 21:30. Beide brauchen den Schutz nicht — zur Schlafenszeit ist der
Rolladen dort ohnehin über den Sonnenuntergang geschlossen.

Die Gegenseite: das 16:00-Netz und die Aufwach-Erkennung setzen den Modus jetzt
auf `keine`, **bevor** sie den Rolladen fahren. Die Verschattung ist damit im
selben Moment wieder zuständig und zieht ihn bei Hitze direkt auf
Verschattungsposition, statt erst beim nächsten 10-Minuten-Zyklus. Das entsperrt
zugleich die Klimaanlage, die bei geschlossenem Rolladen blockiert ist.

### Zielpositionen an einer Stelle

Die Zielposition des Schlafzimmers war auf drei Werte verteilt: gefahren wurde
auf **78**, die „steht schon nah genug"-Prüfung verglich gegen **70**, und der
gemeinsame „alle 5"-Zweig fuhr auf **50**.

Die 70 in der Prüfung war der schädlichste der drei: bei Position 78 ergibt
`|78 − 70| = 8 > 5`, die Zielposition galt also als zu weit weg und wurde bei
jedem Trigger erneut angefahren — die Prüfung sollte genau das verhindern.

Alle Werte stehen jetzt in einer Variable, aus der sich sowohl Fahrbefehl als
auch Prüfung bedienen:

```yaml
positionen:
  cover.schlafzimmer: 78
  cover.badezimmer: 50
  cover.wc: 50
  cover.wohnzimmer: 50
  cover.terrasse: 50
```

Das Schlafzimmer landet damit auch im „alle 5"-Zweig auf 78 statt auf 50.

### `mode: queued` statt `restart`

Die Drift-Korrektur wartet 20 Sekunden und prüft dann nach. Bei `restart` bricht
jeder in dieser Zeit feuernde der zwölf Trigger den Lauf ab — also genau die
Korrektur, die nach dem Overshoot vom 26.07. gebaut wurde. Vertretbar, weil
Temperaturen träge sind: auf der Nordseite kommt die Sonne höchstens abends kurz
vorbei, dicht aufeinanderfolgende Trigger sind nicht zu erwarten. `max: 5`.

### Bewölkt + kühl überstimmt die Innentemp (22.08.2026)

Beobachtet: bei Regen und kühlem Wetter verschattete die Automation trotzdem,
obwohl auf der Nord-/Ostseite gar keine Sonne zum Blocken da ist. Ursache: die
fünf Räume hängen **ausschließlich** an ihrem Innentemp-Sensor (siehe
Beschreibung oben — bewusst kein Helligkeits-, Außentemp- oder
Vorhersage-Faktor). Steigt die Innentemp durch interne Wärmequellen (Kochen,
Elektronik, Personen) über 23°C, schließt die Automation auch dann, wenn
draußen keine nennenswerte Sonne durchkommt — das Verschatten bringt in dem
Moment nichts, es verdunkelt den Raum nur unnötig.

Variable `bewoelkt_kuehl` (finaler Stand nach drei Korrekturen, siehe unten):

```jinja
{{ states('sensor.obersulm_willsbach_bewolkungsgrad') | float(0) > 50
   and states('sensor.wkh_temperature_outside') | float(100) < 23 }}
```

Solange sie zutrifft:

- blockieren alle vier Schließen-Zweige (zusätzliche Bedingung
  `not bewoelkt_kuehl`) — es wird nicht neu verschattet;
- fährt ein neuer, eigener Zweig **aktiv** alle 5 Rolladen wieder hoch, auch
  wenn die jeweilige Innentemp noch über der 21°C-Öffnen-Schwelle liegt. Das
  ist der einzige Zweig der Automation, der Innentemp bewusst überstimmt.
  Respektiert dabei dieselben Ausnahmen wie die bestehenden Öffnen-Zweige:
  Schlafzimmer bleibt bei Schichtmodus `nacht` unangetastet, bereits
  komplett geschlossene Rolladen (state `closed`) werden übersprungen.

Der proaktive „Außentemp > 25°C UND Vorhersage > 25°C"-Zweig braucht keine
eigene Ausnahme dafür: Außentemp > 25°C und < 23°C schließen sich gegenseitig
aus.

#### Korrektur 1: Rolladen fuhren trotz erfüllter Override-Bedingung nicht hoch

Im Test aufgefallen (Trace zeigte die Override-Variable als `true`, aber
keine Aktion): der Öffnen-Zweig hatte anfangs zusätzlich eine
Trigger-Einschränkung (`trigger.id` musste einem bestimmten Trigger
entsprechen). Löste stattdessen einer der Innentemp-Trigger aus — z.B.
`wz_terrasse_close`, weil die Küchentemperatur über 23°C stieg — blockierte
das zwar korrekt den zugehörigen Schließen-Zweig, aber der Öffnen-Zweig
matchte wegen der Trigger-Einschränkung ebenfalls nicht. Kein `choose`-Zweig
traf zu, die Rolladen blieben stehen, bis der nächste `periodic_check` (bis
zu 10 Min. später) kam.

Die Trigger-Einschränkung ist jetzt entfernt — die Override-Variable allein
entscheidet, unabhängig davon, welcher der zwölf Trigger den Lauf ausgelöst
hat. Der Zweig reagiert damit sofort, egal von welchem Trigger die Auswertung
angestoßen wurde.

#### Korrektur 2: `weather.forecast_home` war als Kriterium ungeeignet

Ursprünglich stand hier eine Bedingung auf `weather.forecast_home` (Met.no):
`in [rainy, pouring, lightning-rainy]`, später erweitert um `cloudy`/
`partlycloudy`. Im Betrieb weiter beobachtet: die Nord-/Ost-Räume
verschatteten weiterhin, obwohl es draußen den ganzen Tag nie wärmer als
knapp 23°C wurde. Zwei Traces (15:00 und 15:10 Uhr) zeigten die
Override-Variable als `false` — der Override griff also gar nicht erst.

Ursache, per Verlaufsdaten von `weather.forecast_home` und
`sensor.wkh_temperature_outside` nachvollzogen:

- `weather.forecast_home` ist ein **Momentan-Zustand**, kein Tagesmuster. Am
  fraglichen Tag wechselte er mehrmals stündlich: `rainy` (11:54) →
  `partlycloudy` (12:57) → `sunny` (15:03) → ... Um 15:00/15:10 Uhr stand er
  auf `partlycloudy` — nicht in der ursprünglichen Zustandsliste.
- `sensor.wkh_temperature_outside` lag zur gleichen Zeit bei ca. 21–22°C,
  also unter der Temperaturschwelle. Die reine Temperaturbedingung hätte den
  Fall also korrekt erkannt — die UND-Verknüpfung mit der volatilen
  Wetter-Kategorie hat es verhindert.

Kurz danach zusätzlich beobachtet: `weather.forecast_home` zeigte `sunny`,
während es tatsächlich bewölkt war — der Zustand ist offenbar modellbasiert
(Vorhersage fürs aktuelle Stündchen), keine echte Beobachtung, und kann daran
vorbeiliegen.

#### Korrektur 3: Umstieg auf DWD-Bewölkungsgrad (numerisch statt Kategorie)

`weather.forecast_home` komplett ersetzt durch
`sensor.obersulm_willsbach_bewolkungsgrad` — eine DWD-Stationsmessung
(Deutscher Wetterdienst, HACS-Integration
[FL550/dwd_weather](https://github.com/FL550/dwd_weather)) statt einer
Modell-Zustandskategorie. Ein numerischer Prozentwert lässt sich nicht in
eine falsche Kategorie einsortieren und ist nicht auf eine feste Werteliste
angewiesen — deckt automatisch auch andere Fälle als „Regen" ab, z.B. dichte,
aber trockene Bewölkung.

Schwelle zunächst mit 75% angesetzt, angelehnt an die grobe meteorologische
Einteilung „stark bewölkt bis bedeckt" (6–8 Achtel). Noch am selben Tag per
Live-Abgleich korrigiert: bei einem abgelesenen Wert von 53% wurde der Himmel
bereits als „ziemlich zu" beschrieben — 75% wäre also viel zu träge gewesen
und hätte den ursprünglichen Fall kaum noch abgedeckt. Endgültige Schwelle:
**50%**.

Trigger `bewoelkt_kuehl_open` entsprechend von einem `state`-Trigger auf
`weather.forecast_home` zu einem `numeric_state`-Trigger auf
`sensor.obersulm_willsbach_bewolkungsgrad` (`above: 50`) geändert, damit die
Automation beim Überschreiten der Schwelle sofort neu auswertet statt bis zu
10 Minuten auf den nächsten `periodic_check` zu warten.

Variable und Trigger von `regen_kuehl`/`regen_kuehl_open` in
`bewoelkt_kuehl`/`bewoelkt_kuehl_open` umbenannt — „Regen" war nie das
eigentliche Kriterium, sondern fehlende direkte Sonne.

**Falls die Station wechselt oder die Integration neu eingerichtet wird:**
`sensor.obersulm_willsbach_bewolkungsgrad` ist stations- bzw.
konfigurationsspezifisch benannt (Station „Obersulm-Willsbach", Q242) — bei
einer anderen DWD-Station oder einer neu aufgesetzten Integration muss die
Entity-ID in der Variable `bewoelkt_kuehl` und im Trigger
`bewoelkt_kuehl_open` angepasst werden (*Entwicklerwerkzeuge → Zustände*
prüfen).

**Kalibrierung der 50%-Schwelle:** wie gut die DWD-Stationsmessung die
tatsächlichen Lichtverhältnisse am Haus trifft, ist bisher nur an einem
einzelnen Live-Abgleich (53% ≈ „ziemlich zu") festgemacht. Für einen
belastbareren Abgleich über mehrere Tage gegen die vorhandenen
Helligkeitssensoren (West, Süd) siehe
`../templates/verschattung_bewoelkt_kuehl_test.yaml` — ein eigenständiger
Test-Sensor, bewusst nicht in diese Automation eingebunden.


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
Schichtbeginn** zugeordnet (Toleranz 4 h):

| Ankunft bei der Arbeit | Modus |
| --- | --- |
| 02:00–10:00 | `frueh` (Start 06:00) |
| 10:00–18:00 | `spaet` (Start 14:00) |
| 18:00–02:00 | `nacht` (Start 22:00) |

Damit sind Abweichungen von einer oder zwei Stunden egal, und es gibt keine
Fenstergrenze mehr, an der die Erkennung stillschweigend ausfällt. Die Grenzen
liegen genau in der Mitte zwischen zwei Schichtbeginnen — dort kommt niemand
zur Arbeit.

Zugrunde liegt das klassische 3-Schicht-System 06:00 / 14:00 / 22:00.
Praktisch beginnen die Schichten etwa eine Stunde früher; beobachtet wurde eine
Ankunft um **19:46** bei offiziellem Beginn 22:00. Mit Toleranz 4 h bleiben
dafür noch 1¾ Stunden Reserve bis zur Grenze um 18:00. Passen die Schichtzeiten
grundsätzlich nicht, reicht es, oben in der Automation `schicht_starts` (Beginn
je Schicht, Dezimalstunden — `5.5` = 05:30) und `schicht_toleranz` anzupassen.

Bei gleichmäßigen 8-Stunden-Schichten und Toleranz 4 wird jede Ankunft
zugeordnet. Wer `schicht_starts` auf ungleichmäßige Abstände ändert, kann Lücken
erzeugen — dann bleibt der Modus unverändert und es gibt einen Logbuch-Eintrag.

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

3. **Das Heimkehr-Ereignis kann komplett ausbleiben.** Steht das Auto zuhause,
   während sie mit dem iPhone bei der Arbeit ist, kann der Person-State schon
   auf `home` stehen — bei ihrer echten Ankunft wechselt er dann gar nicht mehr,
   und `arrived_home` feuert nie. Deshalb wird im Heimkehr-Fenster zusätzlich
   alle 30 Minuten geprüft: Nachtmodus aktiv und Rolladen nicht geschlossen →
   zufahren. Normalerweise ist der Rolladen dann ohnehin schon zu
   (Sonnenuntergang) — der Fall greift vor allem, wenn abends die Klimaanlage
   lief und das Schließen ausgesetzt wurde.

Springt der Tracker trotzdem so ungünstig, dass gar keine Schicht erkannt wird,
greift das Heimkehr-Netz: Ankunft zuhause zwischen 04:00 und 09:00 schließt den
Rolladen in jedem Fall.

> **Bewusst in Kauf genommen:** springt der Person-State an einem normalen Tag
> zwischen 04:00 und 09:00 auf `home` (GPS-Zucker), setzt das Heimkehr-Netz
> `nacht` und schließt den Rolladen — es bleibt dann bis 09:30 bzw. 16:00
> dunkel. Ärgerlich, aber nach der Priorisierung der harmlosere Fehler. Fällt
> das im Betrieb öfter auf, wäre der Ansatzpunkt, zusätzlich zu verlangen, dass
> vorher überhaupt eine Abwesenheit von mehreren Stunden lag.

### Klimaanlage: Abluftschlauch im Fenster

Das mobile Klimagerät führt seinen Abluftschlauch durchs Fenster — der Rolladen
kann physisch nicht schließen, solange es läuft. Die ursprüngliche Automation
kannte diese Ausnahme nur beim Schließen zum Sonnenuntergang. Die
Nachtschicht-Zweige schlossen bedingungslos, wären also bei ihrer Ankunft um
06:30 auf den Schlauch gefahren. Dass das bisher nie passiert ist, liegt nur
daran, dass die Nachtschicht-Erkennung nie ausgelöst hat.

Jetzt setzen **alle** schließenden Zweige aus, wenn die Anlage an ist:

| Zweig | Klimaanlage berücksichtigt |
| --- | --- |
| Sonnenuntergang | ja (war schon so) |
| Heimkehr aus der Nachtschicht | ja (neu) |
| Letztes Netz (Heimkehr 04:00–09:00) | ja (neu) |
| Zwangs-Schließen im Heimkehr-Fenster | ja (neu) |
| Klimaanlage ausgeschaltet | entfällt — Anlage ist dann aus |

Der Schichtmodus wird davon nie blockiert: er wird auch dann gesetzt, wenn der
Rolladen gerade nicht zufahren kann. Jeder ausgesetzte Schließvorgang landet im
Logbuch — läuft die Anlage durch, während sie schlafen will, steht dort, warum
es hell bleibt. Wer das aktiv gemeldet haben möchte, kann in diesen Zweigen
neben `logbook.log` eine `notify.*`-Aktion ergänzen.

### Klimaanlage: Nachschließen funktionierte nach Mitternacht nicht

Läuft die Klimaanlage zum Sonnenuntergang, wird das Schließen ausgesetzt und
erst nachgeholt, wenn sie ausgeschaltet wird. Die Bedingung dafür war
`condition: sun / after: sunset` — und die ist **nur zwischen Sonnenuntergang
und Mitternacht** wahr. Nach Mitternacht bezieht sie sich bereits auf den
Sonnenuntergang des neuen Tages und wird falsch. Wurde die Klimaanlage also um
01:00 ausgeschaltet, blieb der Rolladen bis zum Morgen offen.

Jetzt wird `after: sunset` **oder** `before: sunrise` geprüft, also beide
Hälften der Nacht.

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

**Im Echtbetrieb bestätigt** (Nacht 2./3. August): die Automation wurde erst um
22:52 aktiviert, die Ankunft bei der Arbeit war da längst vorbei. Die
Anwesenheitsprüfung um **23:00:00** hat den Modus gesetzt, der Rolladen schloss
um 23:43 beim Ausschalten der Klimaanlage, blieb den ganzen Vormittag zu und
öffnete um 16:00 über das Sicherheitsnetz.

### Logbuch-Einträge finden

Alle `logbook.log`-Aufrufe tragen `entity_id:
input_select.schichtmodus_schlafzimmer`. Im Protokoll (`/logbook`) lässt sich
damit auf diese Entität filtern und man sieht alle Meldungen chronologisch
untereinander, statt im Gesamtstrom zu scrollen. Ohne `entity_id` tauchen
Einträge nur in der ungefilterten Ansicht auf.

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
