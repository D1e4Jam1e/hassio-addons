# Home Assistant Template-Entities

Eigenständige Template-Sensoren, die nicht direkt in einer Automation hängen
- meist zur Beobachtung/Kalibrierung, bevor ein Wert produktiv in eine
Automation übernommen wird.

| Datei | Rolle |
| --- | --- |
| `verschattung_bewoelkt_kuehl_test.yaml` | Historischer Vergleich der Bewölkt+kühl-Erkennung gegen die Helligkeitssensoren, plus Süd/West-Kombisensor |

## verschattung_bewoelkt_kuehl_test.yaml

Bildet die Bedingung `bewoelkt_kuehl` aus
`../automations/verschattung_nord_ost.yaml` (Bewölkungsgrad > 50% UND
Außentemp < 23°C) als eigenständigen `binary_sensor` ab -
**bewusst nicht in die Automation eingebunden**, nur zur Beobachtung.

Hintergrund: die Automation nutzt seit dem 22.08.2026
`sensor.obersulm_willsbach_bewolkungsgrad` (DWD-Stationsmessung, HACS
`FL550/dwd_weather`) statt der vorher genutzten `weather.forecast_home`-
Zustandskategorie (siehe `../automations/README.md`,
"Bewölkt + kühl überstimmt die Innentemp"). Wie gut diese Stationsmessung
die tatsächlichen Lichtverhältnisse am Haus trifft, lässt sich nur empirisch
prüfen - dafür sind bereits zwei richtungsabhängige Helligkeits-Rohwertsensoren
vorhanden (West: `sensor.sonne_westseite_illuminance`, genutzt gemittelt in
`verschattung_west.yaml`; Süd: `sensor.sonne_sudseite_illuminance`).

**Zweiter Sensor `Sonne Süd/West - Maximum (Test)`:** Süd und West stehen
laut Beobachtung 90° zueinander - `max(Süd, West)` deckt dadurch mehr vom
Tag ab als jeder Sensor einzeln, da bei Bewölkung beide gleichzeitig
einbrechen, bei Klarheit aber nur der gerade "angestrahlte" hoch ausschlägt.
Bewusst `max()` statt Mittelwert, damit der jeweilige Peak nicht künstlich
halbiert wird - ein Mittelwert würde den Vergleich mit dem Bewölkungsgrad
schwerer lesbar machen. Deckt den Vormittag (vor dem Süd-Peak) weiterhin
nicht ab, falls das für die Ost-Räume relevant werden sollte. Nutzt bewusst
die Rohwert-Sensoren statt der geglätteten `*_mittelwert`-Varianten, da es
hier nur um einen visuellen Abgleich geht, nicht um eine steuernde
Automation.

**Auswertung:** *Entwicklerwerkzeuge → Historie*, den `binary_sensor`
zusammen mit dem Kombi-`sensor` (oder wahlweise den beiden Einzelsensoren)
auswählen und die An/Aus-Phasen gegen die Helligkeitskurve abgleichen. Da
die Helligkeitssensoren richtungsabhängig sind (reagieren auf Sonnenstand,
nicht nur auf Bewölkung), ist der Abgleich nur in den Tagesstunden
aussagekräftig, in denen bei klarem Himmel tatsächlich direkte Sonne auf
Süd oder West treffen würde - grob Vormittag bis Abend.

Threshold (50%) und Entity-IDs sind hart auf denselben Stand wie die
Automation kopiert, **keine gemeinsame Quelle** - Home Assistant kennt keine
automationsübergreifenden Variablen (siehe
`../automations/README.md`, "Bekannte Kopplung"). Wird die Schwelle in der
Automation nach der Beobachtungsphase angepasst, muss sie hier von Hand
nachgezogen werden, oder die Sensoren werden nach Abschluss der Kalibrierung
wieder entfernt.

**Einbinden:** entweder als eigene Einträge unter der `template:`-Sektion in
`configuration.yaml` (mit `- ` vor `binary_sensor:`/`sensor:` einrücken,
falls dort schon eine Liste existiert) oder über die UI: *Einstellungen →
Geräte & Dienste → Hilfsbereich → Vorlage*.
