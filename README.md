# ⚡ Solakon ONE — Dynamischer Offset Blueprint

Dieser Blueprint ergänzt den [Solakon ONE Nulleinspeisung Blueprint](https://github.com/D4nte85/Solakon-One-Nulleinspeisung-Blueprint-homeassistant) um einen selbstanpassenden Nullpunkt-Offset. Anstatt einen festen Wert (z.B. 30 W) zu verwenden, misst der Offset-Regler kontinuierlich die Schwankungsbreite des Netzsignals und erhöht den Puffer automatisch — genau so weit wie nötig, um Einspeisung zu verhindern.

---

## 🧠 Funktionsprinzip

Das System besteht aus zwei Teilen, die zusammenarbeiten:

### 1. Spike-Filter (im Blueprint)

Der Roh-Netzwert (z.B. vom Shelly 3EM) kann kurze, sprunghafte Ausreißer enthalten — ein Gerät schaltet kurz ein und sofort wieder aus, ein MPPT-Regler takt kurz, oder der Sensor liefert einen Messfehler. Diese Spitzen würden ohne Filter die Statistik verfälschen und den Offset unnötig in die Höhe treiben.

Der Spike-Filter funktioniert so:

- Jede Netzwertänderung wird sofort ausgewertet.
- Ist der Sprung **kleiner als die Spike-Schwelle** (Standard: 200 W) → Wert wird sofort in den Puffer `input_number.solakon_grid_stable` geschrieben.
- Ist der Sprung **größer als die Spike-Schwelle** → der Blueprint wartet die konfigurierte Bestätigungszeit (Standard: 3 s). Fällt der Wert vorher zurück, passiert nichts — der Spike wurde verworfen. Hält der Sprung an, handelt es sich um einen echten Lastsprung und er wird übernommen.

### 2. Volatilitäts-Regler (Sensor + Blueprint)

Der `statistics`-Sensor berechnet die **Standardabweichung (StdDev)** des gefilterten Netzwerts über die letzten 60 Sekunden. Diese Zahl beschreibt, wie stark das Netz in der letzten Minute geschwankt hat.

Aus der StdDev berechnet der Blueprint dann den Offset nach dieser Formel:

```
Offset = max(min_offset,  min_offset + max(0, StdDev − noise_floor) × factor)
         → begrenzt auf cap_offset
```

| Parameter | Standard | Bedeutung |
|:----------|:---------|:----------|
| `min_offset` | 30 W | Untergrenze — der Offset fällt nie darunter |
| `noise_floor` | 15 W | StdDev darunter gilt als normales Rauschen, kein Aufschlag |
| `factor` | 1.5 | Wie stark jedes Watt StdDev den Offset erhöht |
| `cap_offset` | 250 W | Absolute Obergrenze |

**Beispielwerte:**

| Netz-StdDev | Berechneter Offset | Typisches Szenario |
|:------------|:-------------------|:-------------------|
| 0–15 W | 30 W | Stabiles Netz, Nacht |
| 20 W | 37 W | Kühlschrankkompressor |
| 50 W | 82 W | Waschmaschine, Heizstab |
| 100 W | 157 W | Starke unregelmäßige Last |
| 143+ W | 250 W | Cap erreicht |

---

## 🛠️ Installation: Schritt für Schritt

### Schritt 1 — Helper und Sensoren über die UI erstellen

Diese Komponenten müssen **vor** dem Blueprint-Import vorhanden sein. Alle werden über **Einstellungen → Geräte & Dienste → Helfer** angelegt.

---

#### 1.1 — input_number: Spike-Filter Puffer

Speichert den spike-gefilterten Netzwert. Wird vom Blueprint sekündlich beschrieben.

1. **Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Zahl**
2. Felder ausfüllen:

| Feld | Wert |
|:-----|:-----|
| Name | `Solakon Netz spike-gefiltert` |
| Minimaler Wert | `-10000` |
| Maximaler Wert | `10000` |
| Schrittweite | `1` |
| Einheit | `W` |
| Anzeigemodus | Eingabefeld |

3. Speichern → Entity ID: **`input_number.solakon_grid_stable`**

---

#### 1.2 — input_number: Dynamischer Offset Zone 1

Wird vom Blueprint beschrieben und im Nulleinspeisung-Blueprint als *Offset Zone 1 (Dynamisch)* eingetragen.

1. **Helfer erstellen → Zahl**
2. Felder ausfüllen:

| Feld | Wert |
|:-----|:-----|
| Name | `Solakon Offset Zone 1 dynamisch` |
| Minimaler Wert | `0` |
| Maximaler Wert | `500` |
| Schrittweite | `1` |
| Einheit | `W` |
| Initialwert | `30` |

3. Speichern → Entity ID: **`input_number.solakon_offset_zone1`**

---

#### 1.3 — input_number: Dynamischer Offset Zone 2

Identisch zu Zone 1, separat damit Zone 1 und Zone 2 später unabhängig angepasst werden können.

1. **Helfer erstellen → Zahl**
2. Felder ausfüllen:

| Feld | Wert |
|:-----|:-----|
| Name | `Solakon Offset Zone 2 dynamisch` |
| Minimaler Wert | `0` |
| Maximaler Wert | `500` |
| Schrittweite | `1` |
| Einheit | `W` |
| Initialwert | `30` |

3. Speichern → Entity ID: **`input_number.solakon_offset_zone2`**

---

#### 1.4 — Statistik-Sensor: Standardabweichung (60 s)

Dieser Sensor berechnet die Standardabweichung des gefilterten Netzwerts der letzten 60 Sekunden. Er wird **nicht** über die Helfer-UI, sondern über die `configuration.yaml` angelegt, da der `statistics`-Plattformtyp im UI nicht verfügbar ist.

Füge folgendes in deine `configuration.yaml` ein (oder in ein eingebundenes Package):

```yaml
sensor:
  - platform: statistics
    name: "solakon_grid_stddev_60s"
    unique_id: solakon_grid_stddev_60s
    entity_id: input_number.solakon_grid_stable
    state_characteristic: standard_deviation
    max_age:
      seconds: 60
    sampling_size: 30
    precision: 1
```

Nach einem **HA-Neustart** ist der Sensor unter `sensor.solakon_grid_stddev_60s` verfügbar.

> ℹ️ `sampling_size: 30` bei `max_age: 60s` entspricht einem Sample alle ~2 Sekunden — passend für typisches Shelly-Polling. Bei schnellerem Polling (< 1 s) kann `sampling_size` auf 60 erhöht werden.

---

#### 1.5 — Template-Sensor: Berechneter Offset

Dieser Sensor berechnet den fertigen Offset-Wert aus der StdDev. Er dient als lesbare Zustandsanzeige und als Trigger für den Offset-Schreiber im Blueprint.

Ebenfalls in `configuration.yaml` einfügen:

```yaml
template:
  - sensor:
      - name: "Solakon Dynamischer Offset"
        unique_id: solakon_dynamic_offset_v1
        unit_of_measurement: W
        device_class: power
        icon: mdi:chart-bell-curve
        state: >
          {% set min_offset  = 30 %}
          {% set cap_offset  = 250 %}
          {% set noise_floor = 15 %}
          {% set factor      = 1.5 %}

          {% set std_dev = states('sensor.solakon_grid_stddev_60s') | float(-1) %}

          {% if std_dev < 0 %}
            {{ min_offset }}
          {% else %}
            {% set volatility_buffer = [0, (std_dev - noise_floor) * factor] | max %}
            {% set result = (min_offset + volatility_buffer) | round(0) | int %}
            {{ [[min_offset, result] | max, cap_offset] | min }}
          {% endif %}

        attributes:
          std_dev_60s:     "{{ states('sensor.solakon_grid_stddev_60s') | float(0) | round(1) }}"
          grid_stable_now: "{{ states('input_number.solakon_grid_stable') | float(0) | round(0) }}"
          last_updated:    "{{ now().strftime('%H:%M:%S') }}"
```

> ℹ️ Die Formel-Parameter (`min_offset`, `cap_offset`, `noise_floor`, `factor`) im Template-Sensor sind **dekorative Standardwerte** für die Sensor-Anzeige. Die tatsächlich aktiven Werte werden direkt im Blueprint konfiguriert und dort bei jeder Berechnung verwendet — der Template-Sensor muss nach Parameteränderungen nicht angefasst werden.

Nach einem weiteren **HA-Neustart** steht `sensor.solakon_dynamischer_offset` zur Verfügung.

---

### Schritt 2 — Blueprint importieren

[![Open your Home Assistant instance and show the blueprint import dialog.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-one-dynamic-offset-blueprint%2Fblob%2Fmain%2Fsolakon_dynamic_offset_blueprint.yaml)

Oder manuell: Die Datei `solakon_dynamic_offset_blueprint.yaml` in das Verzeichnis `config/blueprints/automation/` kopieren und Home Assistant neu laden (**Entwicklerwerkzeuge → YAML → Automatisierungen neu laden**).

---

### Schritt 3 — Automatisierung aus Blueprint erstellen

1. **Einstellungen → Automatisierungen → Automatisierung erstellen → Blueprint**
2. Blueprint **"Solakon ONE — Dynamischer Offset"** auswählen
3. Konfiguration ausfüllen (Pflichtfelder sind markiert):

**Sensoren:**

| Feld | Wert |
|:-----|:-----|
| 🔌 Netz-Leistungssensor | Dein Shelly / Netz-Sensor (z.B. `sensor.shelly3em_channel_a_power`) |
| 🗃️ Spike-Filter Puffer | `input_number.solakon_grid_stable` |
| 📐 Berechneter Offset Sensor | `sensor.solakon_dynamischer_offset` |
| 🎯 Ziel-Helper Offset Zone 1 | `input_number.solakon_offset_zone1` |
| 🎯 Ziel-Helper Offset Zone 2 | `input_number.solakon_offset_zone2` |

Alle anderen Felder können auf den Standardwerten belassen werden.

4. Automatisierung speichern.

---

### Schritt 4 — Nulleinspeisung Blueprint verknüpfen

Im bereits konfigurierten **Solakon ONE Nulleinspeisung Blueprint** die dynamischen Offset-Entities eintragen:

| Blueprint-Feld | Wert |
|:---------------|:-----|
| 🎯 Offset Zone 1 (Dynamisch) | `input_number.solakon_offset_zone1` |
| 🎯 Offset Zone 2 (Dynamisch) | `input_number.solakon_offset_zone2` |

Die statischen Offset-Werte im Nulleinspeisung-Blueprint bleiben als Fallback erhalten, werden aber automatisch überschrieben sobald der dynamische Offset aktiv ist.

---

## ⚙️ Parameter-Referenz

### 🔍 Spike-Filter

| Parameter | Standard | Empfehlung |
|:----------|:---------|:-----------|
| **Spike-Schwelle** | 200 W | Nicht zu niedrig — sonst werden echte Lastsprünge verzögert |
| **Bestätigungszeit** | 3 s | Entspricht der Anforderung: Spitzen < 3 s werden verworfen |
| **Bestätigungstoleranz** | 100 W | ~50 % der Spike-Schwelle |

### 📐 Offset-Formel

| Parameter | Standard | Wirkung |
|:----------|:---------|:--------|
| **Mindest-Offset** | 30 W | Untergrenze — sollte dem statischen Offset im Nulleinspeisung-Blueprint entsprechen |
| **noise_floor** | 15 W | Erhöhen wenn Offset zu früh steigt (rauschender Sensor) |
| **Volatilitäts-Faktor** | 1.5 | Erhöhen für aggressivere Anpassung, senken für konservativere |
| **Cap** | 250 W | Anpassen je nach Haushalt — bei großen Verbrauchern ggf. höher |

### ✍️ Offset-Schreiber

| Parameter | Standard | Wirkung |
|:----------|:---------|:--------|
| **Schreib-Hysterese** | 3 W | Verhindert Flackern bei minimalen StdDev-Änderungen |
| **Logbuch-Schwelle** | 10 W | Auf 0 setzen für vollständiges Logging, erhöhen für ruhigeres Logbuch |
| **Fallback-Intervall** | 60 s | Regelmäßiger Schreib-Trigger falls StdDev-Sensor sich nicht ändert |

---

## 🔄 Zusammenspiel der Komponenten

```
Shelly 3EM (Roh)
      │
      ▼
[Blueprint: Spike-Filter]
  Sprung > 200 W? → 3 s warten → noch da? → übernehmen
  Sprung ≤ 200 W? → sofort übernehmen
      │
      ▼
input_number.solakon_grid_stable  (gefilterter Puffer)
      │
      ▼
sensor.solakon_grid_stddev_60s    (Statistik: StdDev 60 s)
      │
      ▼
sensor.solakon_dynamischer_offset (Template: Formel → Offset-Wert)
      │
      ▼
[Blueprint: Offset-Schreiber]
  Hysterese überschritten? → schreiben
      │
      ├─▶ input_number.solakon_offset_zone1
      └─▶ input_number.solakon_offset_zone2
                │
                ▼
    [Nulleinspeisung Blueprint]
    liest diese Werte als dynamischen Offset
```

---

## ❓ Häufige Fragen

**Der Offset bleibt dauerhaft bei 30 W, obwohl das Netz schwankt.**
→ Prüfe ob `sensor.solakon_grid_stddev_60s` gültige Werte liefert. Der Sensor braucht nach einem Neustart ~60 s um erste Werte zu berechnen. Prüfe außerdem ob der Spike-Filter den gefilterten Wert korrekt in `input_number.solakon_grid_stable` schreibt (Entwicklerwerkzeuge → Zustände).

**Der Offset steigt auch bei ruhigem Netz auf 50–60 W.**
→ Der `noise_floor` ist zu niedrig für deinen Sensor. Erhöhe ihn auf 20–25 W.

**Sprünge durch den Backofen / die Waschmaschine werden nicht übernommen.**
→ Die `spike_confirm_tolerance` ist zu eng. Erhöhe sie auf 150 W, damit echte Lastsprünge auch bei leichtem Schwingen der Netzleistung bestätigt werden.

**Ich möchte Zone 1 und Zone 2 unterschiedliche Offsets geben.**
→ Derzeit schreibt der Blueprint beide Helper auf denselben Wert. Für unterschiedliche Offsets kann die Automatisierung dupliziert und jeweils ein anderer `volatility_factor` oder `min_offset` konfiguriert werden.

---

## 📋 Changelog

### V1
- Spike-Filter mit konfigurierbarer Schwelle, Bestätigungszeit und -toleranz
- Volatilitäts-Regler basierend auf 60-s-Standardabweichung
- Schreib-Hysterese und Logbuch-Schwelle konfigurierbar
- Fallback-Schreibintervall für stabile Netzphasen
- `mode: parallel` — Spike-Filter und Schreiber blockieren sich nicht gegenseitig
