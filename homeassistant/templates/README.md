# Home Assistant Template-Entities

Eigenständige Template-Sensoren, die nicht direkt in einer Automation hängen
- meist zur Beobachtung/Kalibrierung, bevor ein Wert produktiv in eine
Automation übernommen wird.

| Datei | Rolle |
| --- | --- |
| `verschattung_bewoelkt_kuehl_test.yaml` | Historischer Vergleich der Bewölkt+kühl-Erkennung gegen die Helligkeitssensoren |

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
prüfen - dafür sind bereits zwei richtungsabhängige Helligkeitssensoren
vorhanden (West: `sensor.sonne_westseite_illuminance_mittelwert`, genutzt in
`verschattung_west.yaml`; Süd: analog).

**Auswertung:** *Entwicklerwerkzeuge → Historie*, diesen Sensor zusammen mit
den Helligkeitssensoren auswählen und die An/Aus-Phasen gegen die
Helligkeitskurven abgleichen. Da die Helligkeitssensoren richtungsabhängig
sind (reagieren auf Sonnenstand, nicht nur auf Bewölkung), ist der Abgleich
nur in den Tagesstunden aussagekräftig, in denen bei klarem Himmel
tatsächlich direkte Sonne auf den jeweiligen Sensor treffen würde - z.B.
Süd über den Mittag, West nachmittags/abends.

Threshold (50%) und Entity-ID sind hart auf denselben Stand wie die
Automation kopiert, **keine gemeinsame Quelle** - Home Assistant kennt keine
automationsübergreifenden Variablen (siehe
`../automations/README.md`, "Bekannte Kopplung"). Wird die Schwelle in der
Automation nach der Beobachtungsphase angepasst, muss sie hier von Hand
nachgezogen werden, oder der Sensor wird nach Abschluss der Kalibrierung
wieder entfernt.

**Einbinden:** entweder als eigener Eintrag unter der `template:`-Sektion in
`configuration.yaml` (mit `- ` vor `binary_sensor:` einrücken, falls dort
schon eine Liste existiert) oder über die UI: *Einstellungen → Geräte &
Dienste → Hilfsbereich → Vorlage*.
