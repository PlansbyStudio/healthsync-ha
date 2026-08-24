# HealthSync für Home Assistant

Begleit-Integration zur **HealthSync**-iPhone-App. Die App liest Apple-Health-
Daten (Schritte, Herzfrequenz, HRV, Schlaf, Kalorien, Workouts, Körpermaße,
Vitalwerte) und schickt sie an Home Assistant; diese Integration nimmt sie auf
einem Webhook entgegen und stellt sie als Sensoren bereit. Reines Local Push —
keine Cloud, kein Polling, keine externen Abhängigkeiten.

> **Herkunft:** Fork von [mannotfood/healthsync](https://github.com/mannotfood/healthsync)
> (MIT, © 2026 Jamie Hill). Angepasst: deutsche Oberfläche, vollständige
> Metrikliste im `get_readings`-Selector, eigener Repo-Pfad. Der ursprüngliche
> Lizenztext liegt unverändert in [`LICENSE`](LICENSE).

## Mehrere Personen / mehrere iPhones

Das ist der Kern des Setups: **Pro Person bzw. pro iPhone wird die Integration
einmal hinzugefügt.** Jeder Eintrag erzeugt eine eigene Webhook-URL, eigene
Geräte und eigene Sensoren. Auf jedem iPhone trägst du in der App die zur
Person gehörende URL ein — damit ist die Zuordnung eindeutig, ohne dass die
App irgendetwas über Home-Assistant-Benutzer wissen muss.

1. Einstellungen → Geräte & Dienste → Integration hinzufügen → **HealthSync**
2. Feld **Name** ausfüllen, z. B. `Tim`. Ergebnis: Geräte heißen
   `HealthSync (Tim)` und `HealthSync (Tim) Workouts`, Entitäten entsprechend
   `sensor.healthsync_tim_schritte_heute` usw.
3. Für die nächste Person wiederholen.

Zusätzlich trägt die App in jedem Payload das Feld `user` mit (App →
Einstellungen → Benutzer). Es landet in den `healthsync_sample`-Events und ist
als Gegenprobe nützlich — die eigentliche Zuordnung macht aber die Webhook-URL.

## Einrichtung

1. Integration wie oben hinzufügen. Optional ein **gemeinsames Geheimnis**
   setzen — empfohlen, sobald HA aus dem Internet erreichbar ist. Payloads mit
   falschem Geheimnis werden mit 401 abgelehnt.
2. Nach der Einrichtung zeigt eine Benachrichtigung die erzeugte Webhook-URL.
   Diese **vollständig** in die App kopieren (Einstellungen → Home Assistant),
   nicht aufteilen. Das Geheimnis dort identisch eintragen.
3. In der App **Verbindung testen** antippen — in HA sollte die Meldung
   „HealthSync-Test erfolgreich“ erscheinen.

## Installation

**HACS:** HACS → Integrationen → ⋮ → Benutzerdefiniertes Repository →
`https://github.com/PlansbyStudio/healthsync-ha`, Kategorie *Integration* →
installieren → HA neu starten.

**Manuell:** `custom_components/healthsync/` nach `config/custom_components/`
kopieren und HA neu starten.

## Entitäten

Es entstehen zwei Geräte: **HealthSync** (Schritte, Herz- und Vitalwerte,
Schlaf, Körpermaße, Aktivitätssummen, Sync-Status) und **HealthSync Workouts**.

| Entität | Bedeutung |
|---|---|
| Schritte heute | Schritte seit lokal Mitternacht |
| Aktive Kalorien heute | Aktivenergie (kcal) seit Mitternacht |
| Herzfrequenz | Letzter Herzfrequenz-Messwert (bpm) |
| Ruheherzfrequenz | Letzter Ruhepuls (bpm) |
| Herzfrequenzvariabilität | Letzter HRV-SDNN-Wert (ms) |
| Blutdruck (systolisch/diastolisch) | Letzter Blutdruckwert (mmHg) — zwei Entitäten, weil HealthKit zwei Größen führt |
| Geh-Herzfrequenz | Letzter Durchschnitt beim Gehen (bpm) |
| Herzfrequenz-Erholung | Erholung 1 Minute nach Belastung (bpm) |
| VHF-Anteil | Vorhofflimmern-Anteil (%) |
| Blutsauerstoff | Letzter SpO₂-Wert (%) |
| Atemfrequenz | Atemzüge/min |
| Körpertemperatur | Letzter Wert (°C) |
| Blutzucker | Letzter Wert (mg/dL) |
| Etagen heute / Trainingsminuten heute / Ruheenergie heute / Geh- + Laufstrecke heute | Tagessummen seit Mitternacht |
| VO2max | Letzter Wert (mL/(kg·min)) |
| Gewicht / BMI / Körperfettanteil / Magermasse / Körpergröße / Taillenumfang | Letzter Wert |
| … -Messwert (Event-Entitäten) | Feuern einmal pro *einzelnem* Messwert, auch wenn mehrere in einem Sync ankommen — exakter Wert plus Apples eigener Zeitstempel |
| Schlaf letzte Nacht | Stunden Schlaf der letzten 24 h; Phasen (`deep_minutes`, `rem_minutes`, `core_minutes`, `awake_minutes`) als Attribute |
| Eingeschlafen / Aufgewacht | Lokale Uhrzeit; volles ISO-Datum im Attribut `timestamp` |
| Letzter Sync | Zeitpunkt des letzten Payloads |

**HealthSync Workouts:** Typ, Dauer, Distanz und Kalorien des letzten Workouts,
zehn Einzel-Entitäten für die zehn jüngsten Workouts sowie die Event-Entität
*Workout abgeschlossen* als sauberer Automations-Trigger.

Jedes Sample feuert zusätzlich ein `healthsync_sample`-Event (Rohdaten, ohne
`secret`). Der Test-Button feuert `healthsync_test`.

### Genaue Historie für Herzfrequenz, HRV, VO2max und Gewicht

Diese vier werden zusätzlich als Langzeitstatistik geführt — jeder Messwert
fließt in ein stündliches Min/Max/Mittel, datiert auf die Stunde, in der er
tatsächlich entstanden ist (nicht auf den Sync-Zeitpunkt).

**Wichtig:** Die Standard-Verlaufsansicht zeichnet rohe Zustandswechsel, nicht
diese Statistik — dort sieht es deshalb oft nach einem flachen Wert aus, der
einmal pro Sync springt. Für die echte Auflösung eine **Statistikdiagramm**-
Karte nutzen:

```yaml
type: statistics-graph
title: Herzfrequenz
entities:
  - sensor.healthsync_herzfrequenz
stat_types: [min, mean, max]
period:
  hour: 1
```

### Vollständige, ungemittelte Historie — jeder Messwert, jede Metrik

Jedes gesendete Sample wird zusätzlich exakt so archiviert, wie Apple Health es
geliefert hat — echter Wert, echter Zeitstempel, ohne Rundung oder Gruppierung.
Die Ablage liegt in einer eigenen kleinen Datenbank neben HAs Speicher und ist
damit in HA-Backups enthalten.

Abfrage über die Aktion **`healthsync.get_readings`**:

```yaml
action: healthsync.get_readings
data:
  device_id: <dein HealthSync-Gerät>
  metric: heartRate
  start: "2026-08-01T00:00:00"
  end: "2026-08-13T00:00:00"
```

`start`/`end` sind optional — ohne beide kommt die komplette Historie. Gilt auch
für `metric: workouts` (jedes je synchronisierte Workout, unbegrenzt, nicht vom
Recorder-Purge betroffen).

Beispiel-Dashboard: [`dashboard-examples/metric-history-graph.yaml`](dashboard-examples/metric-history-graph.yaml)

### Logbuch beruhigen

`Letzter Sync` und die Messwert-Event-Entitäten feuern häufig. Rein kosmetisch —
Daten, Verlauf und Automationen bleiben unberührt:

```yaml
logbook:
  exclude:
    entities:
      - sensor.healthsync_letzter_sync
      - event.healthsync_herzfrequenz_messwert
```

## Hinweise

- Die App sendet einen ganzen Batch erneut, wenn ein Sample nicht ankam. Die
  Integration dedupliziert Wiederholungen (Metrik + Zeitraum + Wert), Tages-
  summen zählen also nicht doppelt.
- Die App braucht eine **https**-URL, außer HA liegt im lokalen Netz — ein
  Nabu-Casa-Cloudhook funktioniert gut.
- Tagessummen setzen zu HAs lokaler Mitternacht zurück und überstehen Neustarts.
- Herzfrequenz-Historie wächst mit einer Apple Watch schnell. Bei knappem
  Speicher den Sensor aus dem `recorder` ausschließen.

## Lizenz

MIT — siehe [`LICENSE`](LICENSE).
