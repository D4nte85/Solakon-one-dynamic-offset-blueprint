# Solakon ONE — Dynamischer Offset (Spike-Filter & Volatilität)

Dieses Paket ergänzt die Nulleinspeisung um einen selbstanpassenden Offset. Er analysiert die Netzschwankungen der letzten 60 Sekunden und erhöht den Sicherheits-Puffer automatisch, wenn das Netz unruhig ist (z.B. durch taktend arbeitende Geräte).

## 🛠️ Vorbereitung (WICHTIG)
Aufgrund technischer Einschränkungen in Home Assistant müssen die Helfer in dieser Reihenfolge angelegt werden:

### 1. Spike-Filter Puffer (Helfer -> Zahl)
*   **Name:** `Solakon Netz spike-gefiltert`
*   **ID:** `input_number.solakon_netz_spike_gefiltert`
*   **Bereich:** -10000 bis 10000 / Einheit: `W` / Modus: Eingabefeld

### 2. Statistik-Brücke (Helfer -> Template -> Sensor)
*Zwingend erforderlich für die Statistik-Funktion.*
*   **Name:** `Solakon Netz Brücke`
*   **ID:** `sensor.solakon_netz_brucke`
*   **Zustandstemplate:** 
    ```jinja
    {{ states('input_number.solakon_netz_spike_gefiltert') | float(0) }}
    ```
*   **Einheit:** `W` / **Zustandsklasse:** `measurement` / **Geräteklasse:** `power`

### 3. Statistik-Sensor (Helfer -> Statistik)
*   **Name:** `Solakon Grid StdDev 60s`
*   **ID:** `sensor.solakon_grid_stddev_60s`
*   **Eingabesensor:** `sensor.solakon_netz_brucke`
*   **Charakteristik:** Standardabweichung / **Zeitraum:** `00:01:00`

### 4. Berechneter Offset (Helfer -> Template -> Sensor)
*   **Name:** `Solakon Dynamischer Offset`
*   **ID:** `sensor.solakon_dynamischer_offset`
*   **Zustandstemplate:**
    ```jinja
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
*   **Einheit:** `W` / **Geräteklasse:** `power` / **Symbol:** `mdi:chart-bell-curve`

### 5. Ziel-Helper (Helfer -> Zahl)
Erstelle `input_number.solakon_offset_zone1` und `input_number.solakon_offset_zone2` (0-500 W).
