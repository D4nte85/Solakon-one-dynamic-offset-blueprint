# Solakon ONE — Dynamischer Offset (Spike-Filter & Volatilitäts-Regler)

> 🇩🇪 Deutsche Dokumentation. **[English version at the bottom ↓]**

> Ergänzt die Nulleinspeisung um einen selbstanpassenden Offset. Der Blueprint analysiert die Netzschwankungen der letzten 60 Sekunden und erhöht den Sicherheitspuffer automatisch, wenn das Netz unruhig ist — z.B. durch taktende Verbraucher wie Kompressoren oder Waschmaschinen.

## Installation

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/D4nte85/Solakon-one-dynamic-offset-blueprint/main/solakon_dynamic_offset_blueprint.yaml)

---

## Grundprinzip

Der Blueprint verschiebt das Regelziel des PI-Reglers nach oben:

```
Regelziel = 0 W + Offset
```

Ohne Offset würde der Regler auf 0 W einregeln. Bei fluktuierenden Geräten (Kompressor, Waschmaschine) führt das zu ständigem Ein-/Ausspeisen. Der dynamische Offset hält einen **Netzbezugspuffer**, der groß genug ist, um typische Schwankungen aufzufangen.

Die Standardabweichung der letzten 60 Sekunden dient als Schätzung der nötigen Puffergröße — je unruhiger das Netz, desto größer der Offset. Da die gesamte Reaktionskette (Shelly ~1s → HA → Modbus → Inverter → MPPT-Ramping) mehrere Sekunden beträgt, passt sich der StdDev-Ansatz aus der Vergangenheit an: nach dem ersten Takt-Zyklus eines Geräts ist der Offset bereits kalibriert.

```
Stromzähler (roh)
      │
      ▼
┌─────────────────────┐
│   Spike-Filter      │  Sprünge > Schwelle → erst nach X s bestätigt
│  (input_number)     │
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  Template-Brücke    │  input_number → sensor (HA-Statistik-Pflicht)
│     (sensor)        │
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  StdDev 60s         │  Netz-Volatilität der letzten Minute
│     (sensor)        │
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  Blueprint          │  Berechnet Offset → Zone 1 & Zone 2
│  (Automation)       │
└─────────────────────┘
```

---

## Benötigte Helfer

| # | Typ | ID | Zweck |
|---|-----|----|-------|
| 1 | `input_number` | `solakon_netz_spike_gefiltert` | Spike-gefilterter Zwischenwert |
| 2 | Template-Sensor | `solakon_netz_brucke` | Brücke für Statistik-Sensor |
| 3 | Statistik-Sensor | `solakon_grid_stddev_60s` | Standardabweichung über 60 s |
| 4 | `input_number` | `solakon_offset_zone1` | Offset-Ausgabe Zone 1 |
| 5 | `input_number` | `solakon_offset_zone2` | Offset-Ausgabe Zone 2 |

> **Reihenfolge einhalten!** Jeder Helfer hängt vom vorherigen ab.

---

## Schritt-für-Schritt Einrichtung

### Schritt 1 — Spike-Filter Puffer

*Einstellungen → Geräte & Dienste → Helfer → **Zahl erstellen***

| Feld | Wert |
|------|------|
| Name | `Solakon Netz spike-gefiltert` |
| Objekt-ID | `solakon_netz_spike_gefiltert` |
| Minimalwert | `-10000` |
| Maximalwert | `10000` |
| Einheit | `W` |

### Schritt 2 — Statistik-Brücke

*Helfer → **Template → Sensor erstellen***

> ⚠️ Pflicht: Home Assistant akzeptiert kein `input_number` direkt als Statistik-Eingabe.

| Feld | Wert |
|------|------|
| Name | `Solakon Netz Brücke` |
| Objekt-ID | `solakon_netz_brucke` |
| Einheit | `W` |
| Zustandsklasse | `measurement` |
| Geräteklasse | `power` |

**Zustandstemplate:**
```jinja2
{{ states('input_number.solakon_netz_spike_gefiltert') | float(0) }}
```

### Schritt 3 — Statistik-Sensor (StdDev 60s)

*Helfer → **Statistik erstellen***

| Feld | Wert |
|------|------|
| Name | `Solakon Grid StdDev 60s` |
| Objekt-ID | `solakon_grid_stddev_60s` |
| Eingabesensor | `sensor.solakon_netz_brucke` |
| Charakteristik | Standardabweichung |
| Zeitraum | `00:01:00` |

### Schritt 4 — Ziel-Helfer für Offset-Ausgabe

*Helfer → **Zahl erstellen** — je einmal für Zone 1 und Zone 2*

| Feld | Zone 1 | Zone 2 |
|------|--------|--------|
| Name | `Solakon Offset Zone1` | `Solakon Offset Zone2` |
| Objekt-ID | `solakon_offset_zone1` | `solakon_offset_zone2` |
| Min / Max | `0` / `500` | `0` / `500` |
| Einheit | `W` | `W` |

### Schritt 5 — Blueprint importieren & Automation anlegen

1. Blueprint-Datei nach `config/blueprints/automation/solakon/` kopieren
2. *Einstellungen → Automationen → **Blueprint-Automation erstellen***
3. Blueprint `Solakon ONE — Dynamischer Offset` wählen
4. Pflichtfelder belegen, optionale Parameter anpassen

---

## Blueprint-Parameter

### Pflichtfelder

| Parameter | Beschreibung |
|-----------|-------------|
| 🔌 Netz-Leistungssensor | Roher Zähler-Sensor (z.B. `sensor.shelly3em_power`) |
| 🗃️ Spike-Filter Puffer | `input_number.solakon_netz_spike_gefiltert` |
| 📊 Statistik-Sensor | `sensor.solakon_grid_stddev_60s` |
| 🎯 Ziel-Helper Zone 1/2 | `input_number.solakon_offset_zone1/2` |

### Optionale Parameter

| Parameter | Standard | Beschreibung |
|-----------|----------|--------------|
| ⚡ Spike-Schwelle | `300 W` | Sprünge über diesem Wert → Spike-Verdacht |
| ⏱️ Bestätigungszeit | `3 s` | Wartezeit vor Übernahme eines Spike-Werts |
| 📉 Minimaler Offset | `30 W` | Grundpuffer bei ruhigem Netz |
| 📈 Maximaler Offset | `250 W` | Obergrenze bei unruhigem Netz |
| 🔇 Rausch-Schwelle | `15 W` | StdDev darunter = Grundrauschen |
| 📊 Volatilitäts-Faktor | `1.5` | Verstärkung oberhalb der Rausch-Schwelle |

### Offset-Formel

```
volatility_buffer = max(0, (StdDev - noise_floor) × factor)
offset            = clamp(min_offset + volatility_buffer, min_offset, cap_offset)
```

### Beispielwerte (min_offset=30, noise_floor=15, factor=1.5)

| Netz-Zustand | StdDev | Offset |
|:------------|:------:|:------:|
| Sehr ruhig | 5 W | 30 W *(Minimum)* |
| Normal | 30 W | 53 W |
| Unruhig | 80 W | 128 W |
| Sehr unruhig | 160 W | 228 W |
| Extrem | 250 W+ | 250 W *(Maximum)* |

---

## Fehlerbehebung

| Symptom | Lösung |
|---------|--------|
| Offset bleibt auf Minimum | `sensor.solakon_grid_stddev_60s` prüfen — ggf. einige Minuten nach Ersteinrichtung warten |
| Offset reagiert zu träge | Bestätigungszeit senken oder Spike-Schwelle erhöhen |
| Offset zu aggressiv | Volatilitäts-Faktor oder maximalen Offset reduzieren |
| `input_number` nimmt keinen Wert an | Min/Max-Bereich prüfen (0–500 W empfohlen) |

---

---

# English Documentation

> 🇬🇧 English documentation. **[Deutsche Version oben ↑](#solakon-one--dynamischer-offset-spike-filter--volatilitäts-regler)**

> Adds a self-adjusting offset to zero-feed-in control. The blueprint analyses grid fluctuations over the last 60 seconds and automatically increases the safety buffer when the grid is volatile — e.g. due to cycling loads like compressors or washing machines.

## How It Works

The blueprint shifts the PI controller's target upward:

```
Target = 0 W + Offset
```

Without an offset, the controller regulates to 0 W. With cycling devices this causes constant feed-in/draw oscillation. The dynamic offset maintains a **grid draw buffer** large enough to absorb typical fluctuations.

The standard deviation over 60 seconds estimates the required buffer size. Since the full reaction chain (Shelly ~1s → HA → Modbus → Inverter → MPPT ramp) takes several seconds, the StdDev approach adapts from history — after the first cycle of a device, the offset is already calibrated.

---

## Required Helpers

| # | Type | ID | Purpose |
|---|------|----|---------|
| 1 | `input_number` | `solakon_netz_spike_gefiltert` | Spike-filtered intermediate value |
| 2 | Template sensor | `solakon_netz_brucke` | Bridge for statistics sensor |
| 3 | Statistics sensor | `solakon_grid_stddev_60s` | Standard deviation over 60 s |
| 4 | `input_number` | `solakon_offset_zone1` | Offset output Zone 1 |
| 5 | `input_number` | `solakon_offset_zone2` | Offset output Zone 2 |

> **Order matters!** Each helper depends on the previous one.

---

## Step-by-Step Setup

### Step 1 — Spike-Filter Buffer

*Settings → Devices & Services → Helpers → **Create number***

| Field | Value |
|-------|-------|
| Name | `Solakon Netz spike-gefiltert` |
| Object ID | `solakon_netz_spike_gefiltert` |
| Min / Max | `-10000` / `10000` |
| Unit | `W` |

### Step 2 — Statistics Bridge

*Helpers → **Template → Create sensor***

> ⚠️ Required: Home Assistant does not accept `input_number` directly as a statistics input.

| Field | Value |
|-------|-------|
| Name | `Solakon Netz Brücke` |
| Object ID | `solakon_netz_brucke` |
| Unit | `W` |
| State class | `measurement` |
| Device class | `power` |

**State template:**
```jinja2
{{ states('input_number.solakon_netz_spike_gefiltert') | float(0) }}
```

### Step 3 — Statistics Sensor (StdDev 60s)

*Helpers → **Create statistic***

| Field | Value |
|-------|-------|
| Name | `Solakon Grid StdDev 60s` |
| Object ID | `solakon_grid_stddev_60s` |
| Input sensor | `sensor.solakon_netz_brucke` |
| Characteristic | Standard deviation |
| Time period | `00:01:00` |

### Step 4 — Offset Output Helpers

*Helpers → **Create number** — once for Zone 1, once for Zone 2*

| Field | Zone 1 | Zone 2 |
|-------|--------|--------|
| Name | `Solakon Offset Zone1` | `Solakon Offset Zone2` |
| Object ID | `solakon_offset_zone1` | `solakon_offset_zone2` |
| Min / Max | `0` / `500` | `0` / `500` |
| Unit | `W` | `W` |

### Step 5 — Import Blueprint & Create Automation

1. Copy blueprint file to `config/blueprints/automation/solakon/`
2. *Settings → Automations → **Create blueprint automation***
3. Select `Solakon ONE — Dynamischer Offset`
4. Fill in required fields, adjust optional parameters as needed

---

## Blueprint Parameters

### Required

| Parameter | Description |
|-----------|-------------|
| 🔌 Grid Power Sensor | Raw meter sensor (e.g. `sensor.shelly3em_power`) |
| 🗃️ Spike-Filter Buffer | `input_number.solakon_netz_spike_gefiltert` |
| 📊 Statistics Sensor | `sensor.solakon_grid_stddev_60s` |
| 🎯 Target Helper Zone 1/2 | `input_number.solakon_offset_zone1/2` |

### Optional Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| ⚡ Spike Threshold | `300 W` | Deviations above this value trigger spike detection |
| ⏱️ Confirmation Time | `3 s` | Wait time before accepting a spike value |
| 📉 Minimum Offset | `30 W` | Base buffer during calm grid conditions |
| 📈 Maximum Offset | `250 W` | Upper limit during volatile conditions |
| 🔇 Noise Floor | `15 W` | StdDev below this = measurement noise |
| 📊 Volatility Factor | `1.5` | Amplification above the noise floor |

### Offset Formula

```
volatility_buffer = max(0, (StdDev - noise_floor) × factor)
offset            = clamp(min_offset + volatility_buffer, min_offset, cap_offset)
```

### Example Values (min_offset=30, noise_floor=15, factor=1.5)

| Grid State | StdDev | Offset |
|:-----------|:------:|:------:|
| Very calm | 5 W | 30 W *(minimum)* |
| Normal | 30 W | 53 W |
| Volatile | 80 W | 128 W |
| Very volatile | 160 W | 228 W |
| Extreme | 250 W+ | 250 W *(maximum)* |

---

## Troubleshooting

| Symptom | Solution |
|---------|----------|
| Offset stays at minimum | Check `sensor.solakon_grid_stddev_60s` — may need a few minutes after first setup |
| Offset too slow to react | Lower confirmation time or raise spike threshold |
| Offset too aggressive | Reduce volatility factor or maximum offset |
| `input_number` rejects values | Check min/max range (0–500 W recommended) |
