# Changelog

Alle nennenswerten Änderungen am Solakon-Dynamic-Offset-Blueprint.
Das Blueprint hat kein Versionsschema — Einträge sind nach Datum gruppiert.

## 2026-08-06
- Voll-Gate für den Entladeschutz ergänzt: bei vollem Akku (SoC ≥ Voll-Schwelle, Standard 98 %) wird der Schutz abgeschaltet, damit überschüssige PV eingespeist statt abgeregelt wird
- Behebt eine bistabile Verriegelung bei vollem Akku: da der Wechselrichter die PV dann auf den Eigenbedarf drosselt, unterschätzt die gemessene PV-Leistung ihr Potenzial, und der aktive Schutz konnte sich in einen unnötigen dauerhaften Netzbezug einsperren (statt den Überschuss zu exportieren)
- Neuer optionaler Batterie-Sensor (`soc_sensor`) und Zone-1-Parameter „Voll-Schwelle“; ohne Batterie-Sensor unverändertes Verhalten. Der Batterie-Sensor dient nur der Voll-Erkennung, nicht der Zonenwahl (die bleibt beim Haupt-Blueprint)

## 2026-08-02
- Entladeschutz-Vorzeichenflattern behoben: Live-Test zeigte einen sich selbst erregenden Regelkreis (Offset alle 30 s zwischen −250 W und +250 W), weil der Vergleich PV vs. Ausgangsleistung die Ausgangsleistung des PI-Reglers gegen sich selbst verglich
- Vergleichsgröße auf Hauslast (Ausgangsleistung + rohe Netzleistung) umgestellt — physikalische Invariante, unabhängig von der internen PI-Aufteilung zwischen Ausgang und Netz
- Neue Sensoren: rohe Netzleistung (`grid_power_sensor`) sowie `input_number`-Helfer zur Speicherung der EMA-geglätteten Hauslast zwischen den 30s-Zyklen
- Neuer Parameter „Lastglättung α“ (Standard 0.2, ≈ 90 s Zeitkonstante) filtert kurze Lastspitzen (z. B. Wasserkocher)
- Entlade-Hysterese-Standard auf 100 W angehoben (zuvor 50 W)

## 2026-07-31
- Entladeschutz für Zone 1 ergänzt: negativer Offset (Einspeisung) wird zu positivem Bezugspuffer, wenn der Akku entlädt (Ausgangsleistung ≥ PV) — verhindert Ausspeisen von Batterieenergie ins Netz. Kein zusätzliches SoC-Gate, da die SoC-Zonenlogik bereits vom Haupt-Blueprint (Nulleinspeisung) übernommen wird
- Nur Zone 1: Zone 2 und AC erzwingen im Haupt-Blueprint 0 A Entladestrom und deckeln den Ausgang auf PV — dort ist ein Entladeschutz wirkungslos
- Neue optionale Sensor-Sektion (PV-Leistung, Ausgangsleistung) mit Defaults der Solakon-ONE-Entitäten
- Zone 1: Schalter „Entladeschutz“ und „Entlade-Hysterese“ (Standard 50 W, verhindert Vorzeichen-Flattern am Überschuss-Übergang)
- Vollständig abwärtskompatibel: bei ausgeschaltetem Entladeschutz unverändertes Verhalten; Sensor-Ausfall greift fail-safe (Schutz)

## 2026-05-22
- DB-Wachstum behoben: State-Trigger durch `time_pattern` (alle 30 s) ersetzt
- Choose-Bedingungen verfeinert: redundanter Guard entfernt, Prüfung auf geänderten Wert ergänzt

## 2026-03-10 – 2026-03-23
- Erstveröffentlichung und iterative Verbesserungen an Blueprint und README
