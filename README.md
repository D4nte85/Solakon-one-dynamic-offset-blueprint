# Solakon ONE — Dynamischer Offset (Volatilitäts-Regler)

> 🇩🇪 Deutsche Dokumentation. **[English version at the bottom ↓](#english-documentation)**

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

Die Standardabweichung der letzten 60 Sekunden dient als Schätzung der nötigen Puffergröße — je unruhiger das Netz, desto größer der Offset. **Wichtig:** Spikes zählen dabei absichtlich in die StdDev hinein — schnelle Schwankungen in beide Richtungen sind genau das, was der Offset abfedern soll. Ein neues stabiles Lastniveau normalisiert die StdDev von selbst innerhalb der nächsten 60 Sekunden.

```
Stromzähler (roh)
      │
      ▼
┌─────────────────────┐
│  StdDev 60s         │  Netz-Volatilität der letzten Minute (Spikes inklusive)
│     (sensor)        │
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  Blueprint          │  Berechnet Offset → Zone 1, Zone 2 & Zone AC
│  (Automation)       │
└─────────────────────┘
```

---

## Benötigte Helfer

| # | Typ | ID | Zweck |
|---|-----|----|-------|
| 1 | Statistik-Sensor | `solakon_grid_stddev_60s` | Standardabweichung über 60 s |
| 2 | `input_number` | `solakon_offset_zone1` | Offset-Ausgabe Zone 1 *(optional)* |
| 3 | `input_number` | `solakon_offset_zone2` | Offset-Ausgabe Zone 2 *(optional)* |
| 4 | `input_number` | `solakon_offset_zone_ac` | Offset-Ausgabe Zone AC *(optional)* |

> **Reihenfolge einhalten!** Der Statistik-Sensor muss vor dem Blueprint angelegt werden.
> **Mindestens einer** der Ziel-Helfer (Zone 1, Zone 2, Zone AC) muss konfiguriert sein.

---

## Schritt-für-Schritt Einrichtung

### Schritt 1 — Statistik-Sensor (StdDev 60s)

*Helfer → **Statistik erstellen***

> Der Statistik-Sensor liest direkt vom Roh-Sensor (z.B. `sensor.shelly3em_power`). Kein Zwischenschritt nötig.

| Feld | Wert |
|------|------|
| Name | `Solakon Grid StdDev 60s` |
| Objekt-ID | `solakon_grid_stddev_60s` |
| Eingabesensor | `sensor.shelly3em_power` *(oder eigener Zähler-Sensor)* |
| Charakteristik | Standardabweichung |
| Zeitraum | `00:01:00` |

### Schritt 2 — Ziel-Helfer für Offset-Ausgabe

*Helfer → **Zahl erstellen** — je nach Bedarf für Zone 1, Zone 2 und/oder Zone AC*

| Feld | Zone 1 | Zone 2 | Zone AC |
|------|--------|--------|---------|
| Name | `Solakon Offset Zone1` | `Solakon Offset Zone2` | `Solakon Offset Zone AC` |
| Objekt-ID | `solakon_offset_zone1` | `solakon_offset_zone2` | `solakon_offset_zone_ac` |
| Min / Max | `0` / `500` | `0` / `500` | `0` / `500` |
| Einheit | `W` | `W` | `W` |

### Schritt 3 — Blueprint importieren & Automation anlegen

1. Blueprint-Datei nach `config/blueprints/automation/solakon/` kopieren
2. *Einstellungen → Automationen → **Blueprint-Automation erstellen***
3. Blueprint `Solakon ONE — Dynamischer Offset` wählen
4. Pflichtfelder belegen, optionale Parameter anpassen

---

## Blueprint-Parameter

### Pflichtfelder

| Parameter | Beschreibung |
|-----------|-------------|
| 📊 Statistik-Sensor | `sensor.solakon_grid_stddev_60s` |

### Ziel-Helfer (mind. einer erforderlich)

| Parameter | Beschreibung |
|-----------|-------------|
| 🎯 Ziel-Helper Zone 1 | `input_number.solakon_offset_zone1` |
| 🎯 Ziel-Helper Zone 2 | `input_number.solakon_offset_zone2` |
| 🎯 Ziel-Helper Zone AC | `input_number.solakon_offset_zone_ac` |

### Optionale Parameter

| Parameter | Standard | Beschreibung |
|-----------|----------|--------------|
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
| Offset reagiert zu träge | Volatilitäts-Faktor erhöhen oder Rausch-Schwelle senken |
| Offset zu aggressiv | Volatilitäts-Faktor oder maximalen Offset reduzieren |
| `input_number` nimmt keinen Wert an | Min/Max-Bereich prüfen (0–500 W empfohlen) |
| Automation läuft nicht | Prüfen ob mindestens ein Ziel-Helfer (Zone 1/2/AC) konfiguriert ist |

---

---

# English Documentation

> 🇬🇧 English documentation. **[Deutsche Version oben ↑](#solakon-one--dynamischer-offset-volatilitäts-regler)**

> Adds a self-adjusting offset to zero-feed-in control. The blueprint analyses grid fluctuations over the last 60 seconds and automatically increases the safety buffer when the grid is volatile — e.g. due to cycling loads like compressors or washing machines.

## How It Works

The blueprint shifts the PI controller's target upward:

```
Target = 0 W + Offset
```

Without an offset, the controller regulates to 0 W. With cycling devices this causes constant feed-in/draw oscillation. The dynamic offset maintains a **grid draw buffer** large enough to absorb typical fluctuations.

The standard deviation over 60 seconds estimates the required buffer size. **Importantly:** spikes intentionally count toward the StdDev — fast swings in both directions are exactly what the offset is meant to absorb. A new stable load level normalises the StdDev on its own within the next 60 seconds.

---

## Required Helpers

| # | Type | ID | Purpose |
|---|------|----|---------|
| 1 | Statistics sensor | `solakon_grid_stddev_60s` | Standard deviation over 60 s |
| 2 | `input_number` | `solakon_offset_zone1` | Offset output Zone 1 *(optional)* |
| 3 | `input_number` | `solakon_offset_zone2` | Offset output Zone 2 *(optional)* |
| 4 | `input_number` | `solakon_offset_zone_ac` | Offset output Zone AC *(optional)* |

> **Order matters!** The statistics sensor must be created before the blueprint.
> **At least one** of the target helpers (Zone 1, Zone 2, Zone AC) must be configured.

---

## Step-by-Step Setup

### Step 1 — Statistics Sensor (StdDev 60s)

*Helpers → **Create statistic***

> The statistics sensor reads directly from the raw sensor (e.g. `sensor.shelly3em_power`). No intermediate step required.

| Field | Value |
|-------|-------|
| Name | `Solakon Grid StdDev 60s` |
| Object ID | `solakon_grid_stddev_60s` |
| Input sensor | `sensor.shelly3em_power` *(or your own meter sensor)* |
| Characteristic | Standard deviation |
| Time period | `00:01:00` |

### Step 2 — Offset Output Helpers

*Helpers → **Create number** — as needed for Zone 1, Zone 2, and/or Zone AC*

| Field | Zone 1 | Zone 2 | Zone AC |
|-------|--------|--------|---------|
| Name | `Solakon Offset Zone1` | `Solakon Offset Zone2` | `Solakon Offset Zone AC` |
| Object ID | `solakon_offset_zone1` | `solakon_offset_zone2` | `solakon_offset_zone_ac` |
| Min / Max | `0` / `500` | `0` / `500` | `0` / `500` |
| Unit | `W` | `W` | `W` |

### Step 3 — Import Blueprint & Create Automation

1. Copy blueprint file to `config/blueprints/automation/solakon/`
2. *Settings → Automations → **Create blueprint automation***
3. Select `Solakon ONE — Dynamischer Offset`
4. Fill in required fields, adjust optional parameters as needed

---

## Blueprint Parameters

### Required

| Parameter | Description |
|-----------|-------------|
| 📊 Statistics Sensor | `sensor.solakon_grid_stddev_60s` |

### Target Helpers (at least one required)

| Parameter | Description |
|-----------|-------------|
| 🎯 Target Helper Zone 1 | `input_number.solakon_offset_zone1` |
| 🎯 Target Helper Zone 2 | `input_number.solakon_offset_zone2` |
| 🎯 Target Helper Zone AC | `input_number.solakon_offset_zone_ac` |

### Optional Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
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
| Offset too slow to react | Increase volatility factor or lower noise floor |
| Offset too aggressive | Reduce volatility factor or maximum offset |
| `input_number` rejects values | Check min/max range (0–500 W recommended) |
| Automation does not run | Verify at least one target helper (Zone 1/2/AC) is configured |
