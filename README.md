# Solakon ONE — Dynamischer Offset (Spike-Filter & Volatilitäts-Regler)

> Ergänzt die Nulleinspeisung um einen selbstanpassenden Offset. Der Blueprint analysiert die Netzschwankungen der letzten 60 Sekunden und erhöht den Sicherheitspuffer automatisch, wenn das Netz unruhig ist — z.B. durch taktende Verbraucher wie Kompressoren oder Waschmaschinen.

## Installation

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/D4nte85/Solakon-one-dynamic-offset-blueprint/main/solakon_dynamic_offset_blueprint.yaml)


---

## Übersicht: Wie es funktioniert

```
Stromzähler (roh)
      │
      ▼
┌─────────────────────┐
│   Spike-Filter      │  Sprünge > Schwelle werden erst nach X Sekunden
│  (input_number)     │  bestätigt → verhindert Fehlreaktionen
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  Template-Brücke    │  Wandelt input_number → sensor (HA-Statistik-Pflicht)
│     (sensor)        │
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  StdDev 60s         │  Misst Netz-Volatilität der letzten Minute
│     (sensor)        │
└─────────────────────┘
      │
      ▼
┌─────────────────────┐
│  Blueprint          │  Berechnet Offset
│  (Automation)       │  und schreibt in Zone 1 & Zone 2
└─────────────────────┘
      │
      ▼
   Zone 1 / Zone 2
  (input_number)
```

---

## Benötigte Helfer

Der Blueprint benötigt **4 externe Helfer**.

| # | Typ | ID | Zweck |
|---|-----|----|-------|
| 1 | `input_number` | `solakon_netz_spike_gefiltert` | Spike-gefilterter Zwischenwert |
| 2 | Template-Sensor | `solakon_netz_brucke` | Brücke für Statistik-Sensor |
| 3 | Statistik-Sensor | `solakon_grid_stddev_60s` | Standardabweichung über 60 s |
| 4 | `input_number` | `solakon_offset_zone1` | Offset-Ausgabe für Zone 1 |
| 5 | `input_number` | `solakon_offset_zone2` | Offset-Ausgabe für Zone 2 |

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
| Anzeigemodus | Eingabefeld |

---

### Schritt 2 — Statistik-Brücke

*Helfer → **Template → Sensor erstellen***

> ⚠️ Zwingend erforderlich: Home Assistant akzeptiert kein `input_number` direkt als Eingabe für den Statistik-Helper.

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

---

### Schritt 3 — Statistik-Sensor (StdDev 60s)

*Helfer → **Statistik erstellen***

| Feld | Wert |
|------|------|
| Name | `Solakon Grid StdDev 60s` |
| Objekt-ID | `solakon_grid_stddev_60s` |
| Eingabesensor | `sensor.solakon_netz_brucke` |
| Charakteristik | Standardabweichung |
| Zeitraum | `00:01:00` |

---

### Schritt 4 — Ziel-Helfer für Offset-Ausgabe

*Helfer → **Zahl erstellen** — je einmal für Zone 1 und Zone 2*

| Feld | Zone 1 | Zone 2 |
|------|--------|--------|
| Name | `Solakon Offset Zone1` | `Solakon Offset Zone2` |
| Objekt-ID | `solakon_offset_zone1` | `solakon_offset_zone2` |
| Minimalwert | `0` | `0` |
| Maximalwert | `500` | `500` |
| Einheit | `W` | `W` |

---

### Schritt 5 — Blueprint importieren & Automation anlegen

1. Blueprint-Datei nach `config/blueprints/automation/solakon/` kopieren
2. *Einstellungen → Automationen → **Blueprint-Automation erstellen***
3. Blueprint `Solakon ONE — Dynamischer Offset` wählen
4. Pflichtfelder belegen (Netz-Sensor, alle Helfer)
5. Optionale Parameter nach Bedarf anpassen (s. u.)

---

## Blueprint-Parameter

### Pflichtfelder

| Parameter | Beschreibung |
|-----------|-------------|
| 🔌 Netz-Leistungssensor | Roher Sensor vom Stromzähler (z.B. `sensor.shelly3em_power`) |
| 🗃️ Spike-Filter Puffer | `input_number.solakon_netz_spike_gefiltert` |
| 📊 Statistik-Sensor | `sensor.solakon_grid_stddev_60s` |
| 🎯 Ziel-Helper Zone 1/2 | `input_number.solakon_offset_zone1/2` |

### Optionale Parameter (mit Standardwerten)

| Parameter | Standard | Beschreibung |
|-----------|----------|--------------|
| ⚡ Spike-Schwelle | `300 W` | Sprünge größer als dieser Wert werden als Spike behandelt |
| ⏱️ Bestätigungszeit | `3 s` | Wartezeit, bevor ein Spike-Wert übernommen wird |
| 📉 Minimaler Offset | `30 W` | Untere Grenze des Offsets bei ruhigem Netz |
| 📈 Maximaler Offset | `250 W` | Obere Grenze des Offsets bei sehr unruhigem Netz |
| 🔇 Rausch-Schwelle | `15 W` | StdDev unterhalb dieser Grenze gilt als Grundrauschen |
| 📊 Volatilitäts-Faktor | `1.5` | Verstärkung der Volatilität auf den Offset |

### Offset-Formel

```
volatility_buffer = max(0, (StdDev - noise_floor) × factor)
offset            = clamp(min_offset + volatility_buffer, min_offset, cap_offset)
```

**Beispiele bei verschiedenen Netzsituationen:**

| Netz-Zustand | StdDev | Berechneter Offset |
|-------------|--------|--------------------|
| Sehr ruhig | 5 W | 30 W *(Minimum)* |
| Normal | 30 W | 53 W |
| Unruhig | 80 W | 128 W |
| Sehr unruhig | 160 W | 228 W |
| Extrem | 250 W+ | 250 W *(Maximum)* |

---

## Fehlerbehebung

**Offset springt nicht / bleibt auf Minimum**
→ Prüfe ob `sensor.solakon_grid_stddev_60s` einen Wert liefert (ggf. mehrere Minuten warten nach Ersteinrichtung).

**Offset reagiert zu träge auf Lastspitzen**
→ Bestätigungszeit verringern oder Spike-Schwelle erhöhen.

**Offset zu aggressiv / Einspeisung zu hoch**
→ Volatilitäts-Faktor reduzieren oder maximalen Offset begrenzen.

**`input_number` akzeptiert keinen Wert vom Blueprint**
→ Sicherstellen, dass Min/Max-Bereich den Offset-Werten entspricht (0–500 W empfohlen).
