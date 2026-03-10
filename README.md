# ⚡ Solakon ONE — Dynamischer Offset Blueprint

Dieser Blueprint ergänzt den [Solakon ONE Nulleinspeisung Blueprint](https://github.com/D4nte85/Solakon-One-Nulleinspeisung-Blueprint-homeassistant) um einen selbstanpassenden Nullpunkt-Offset. Anstatt einen festen Wert (z. B. 30 W) zu verwenden, misst der Offset-Regler kontinuierlich die Schwankungsbreite des Netzsignals und erhöht den Puffer automatisch — genau so weit wie nötig, um Einspeisung zu verhindern.

---

## 🧠 Funktionsprinzip

Das System besteht aus zwei Teilen, die zusammenarbeiten:

### 1. Spike-Filter (im Blueprint)

Der Roh-Netzwert (z. B. vom Shelly 3EM) kann kurze, sprunghafte Ausreißer enthalten. Der Spike-Filter verhindert, dass diese kurzen Spitzen die Statistik verfälschen:

* Änderungen **kleiner als die Spike-Schwelle** (Standard: 200 W) werden sofort übernommen.
* Änderungen **größer als die Spike-Schwelle** werden erst nach einer **Bestätigungszeit** (Standard: 3 s) übernommen, sofern der Wert stabil bleibt.

### 2. Volatilitäts-Regler (Sensoren + Blueprint)

Ein Statistik-Sensor berechnet die **Standardabweichung (StdDev)** des gefilterten Netzwerts über die letzten 60 Sekunden. Daraus wird der dynamische Offset berechnet:

| Netz-StdDev | Berechneter Offset (Beispiel) | Szenario |
| --- | --- | --- |
| 0–15 W | 30 W | Stabiles Netz, Nacht |
| 50 W | 82 W | Waschmaschine, Heizstab |
| 143+ W | 250 W | Maximale Begrenzung (Cap) |

---

## 🛠️ Installation: Schritt für Schritt

Alle Komponenten werden direkt über die Benutzeroberfläche unter **Einstellungen → Geräte & Dienste → Helfer** angelegt.

### Schritt 1 — Helfer und Sensoren erstellen

#### 1.1 — input_number: Spike-Filter Puffer

1. **Helfer erstellen → Zahl**
2. Name: `Solakon Netz spike-gefiltert`
3. Minimaler Wert: `-10000` / Maximaler Wert: `10000`
4. Einheit: `W` / Anzeigemodus: `Eingabefeld`
5. Entity ID: **`input_number.solakon_grid_stable`**

#### 1.2 — input_number: Dynamischer Offset Zone 1

1. **Helfer erstellen → Zahl**
2. Name: `Solakon Offset Zone 1 dynamisch`
3. Minimaler Wert: `0` / Maximaler Wert: `500`
4. Einheit: `W` / Initialwert: `30`
5. Entity ID: **`input_number.solakon_offset_zone1`**

#### 1.3 — Statistik-Sensor: Standardabweichung

1. **Helfer erstellen → Statistik**
2. Name: `solakon_grid_stddev_60s`
3. Eingabesensor: `input_number.solakon_grid_stable`
4. Charakteristik: `Standardabweichung`
5. Zeitraum: `00:01:00` (60 Sekunden)
6. Maximale Anzahl an Messwerten: `30` (bei 2s Polling)

#### 1.4 — Template-Sensor: Berechneter Offset

1. **Helfer erstellen → Template → Template für einen Sensor erstellen**
2. Name: `Solakon Dynamischer Offset`
3. Zustandstemplate:

```jinja2
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

```

4. Maßeinheit: `W` / Geräteklasse: `Leistung` / Symbol: `mdi:chart-bell-curve`

---

### Schritt 2 — Blueprint importieren

---

### Schritt 3 — Automatisierung erstellen

1. **Einstellungen → Automatisierungen → Automatisierung erstellen → Blueprint**
2. **"Solakon ONE — Dynamischer Offset"** auswählen.
3. Sensoren zuordnen:
* **Netz-Leistungssensor:** Dein Shelly (z. B. `sensor.shelly3em_total_power`)
* **Spike-Filter Puffer:** `input_number.solakon_grid_stable`
* **Berechneter Offset Sensor:** `sensor.solakon_dynamischer_offset`
* **Ziel-Helper Offset Zone 1:** `input_number.solakon_offset_zone1`



---

### Schritt 4 — Nulleinspeisung verknüpfen

Trage im **Solakon ONE Nulleinspeisung Blueprint** unter dem Feld **"Offset Zone 1 (Dynamisch)"** den Helfer `input_number.solakon_offset_zone1` ein.

---

## ⚙️ Parameter-Referenz

| Parameter | Standard | Wirkung |
| --- | --- | --- |
| **Spike-Schwelle** | 200 W | Ab wann ein Sprung als "Spike" geprüft wird. |
| **Bestätigungszeit** | 3 s | Wie lange ein Sprung stabil sein muss. |
| **Mindest-Offset** | 30 W | Der Puffer bei absolut ruhigem Netz. |
| **Noise Floor** | 15 W | Filtert Grundrauschen des Sensors aus der StdDev. |
| **Volatilitäts-Faktor** | 1.5 | Multiplikator für die Schwankungsbreite. |
| **Cap** | 250 W | Maximaler Offset, um Ertragseinbußen zu begrenzen. |

---

## 📋 Changelog

### V1

* Vollständige UI-Konfiguration möglich.
* Spike-Filter mit Bestätigungslogik.
* Dynamische Anpassung basierend auf 60s-Standardabweichung.
* Hysterese-Schutz zur Schonung des Speichers/Logs.
