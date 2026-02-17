# PV-Forecast basiertes Energiemanagement für OpenHAB

## Überblick

Intelligentes Energiemanagement-System für OpenHAB 5 auf Raspberry Pi 4B mit PV-Forecast basierter Heizungssteuerung und Verbraucheroptimierung.

**Erwartete Einsparung:** 800–1500 €/Jahr durch Eigenverbrauchsoptimierung

---

## Systemstatus

Dein OpenHAB 5 System ist zu **70% bereit** für intelligentes PV-gesteuertes Energiemanagement.

### ✅ Vorhandene Stärken

- Wärmepumpe mit Solar-Integration (`Waermepumpe_Solar_Power_Setpoint`)
- 6-Zonen Fußbodenheizung mit Ventilsteuerung
- 11 Raumtemperatursensoren (EG + OG komplett)
- PV-Wechselrichter mit Batterie, vollständiges Energy-Monitoring
- Wallbox (GoeCharger) mit Ladesteuerung
- Haushaltsgeräte mit Energy-Tracking (Spülmaschine, Waschmaschine, Trockner)
- 20+ Fenster-/Türkontakte, Bewegungsmelder, Luftfeuchte-Sensoren
- Lüftungsanlage mit Temperatur/Flowrate-Monitoring

### ❌ Kritische Lücken

- Kein Astro-Binding (Sonnenhöhe/-azimut fehlt)
- Kein SolarForecast-Binding (PV-Prognose fehlt)
- Kein Wetter-Binding (Bewölkung/Wind fehlt)
- ⚠️ Keine Bodentemperatur-Sensoren (FBH-Trägheit unsichtbar)
- ⚠️ Kein Backofen/Herd-Monitoring (große Wärmequelle fehlt)

---

## Lösungs-Architektur

Ganzheitliches PV-Forecast System mit 3 Säulen:

### 1. Heizungsregelung
Vorlauf-Reduktion bei Sonnenprognose, FBH-Vorsteuerung

### 2. Verbraucher-Priorisierung
Batterie → Warmwasser → Wallbox → Haushaltsgeräte → Einspeisung

### 3. KI-basierte Parameter-Optimierung
Datenloggen → CSV-Export → ChatGPT/lokales ML → Schwellwerte lernen

---

## Implementierungs-Plan

### Phase 1: Basis-Infrastruktur (Woche 1-2)

#### 1.1 SolarForecast-Binding installieren
**Priorität:** KRITISCH | **Aufwand:** 30 Min

- Bundle in OH UI installieren (Settings → Add-ons → SolarForecast)
- Forecast.Solar Account erstellen (kostenlos)
- Thing konfigurieren mit Standort Gaildorf (48.95, 9.99)
- Items anlegen: `PV_Power_Estimate`, `PV_Energy_Remain`, `PV_Energy_Tomorrow`, `PV_Energy_Actual`
- Testen: Log prüfen, Items in UI sichtbar

**Dokumentation:** [SolarForecast Binding](https://www.openhab.org/addons/bindings/solarforecast/)

#### 1.2 Astro-Binding installieren
**Priorität:** KRITISCH | **Aufwand:** 20 Min

- Bundle installieren
- Thing erstellen mit Geolocation Gaildorf
- Items anlegen: `Astro_Sun_Elevation`, `Astro_Sun_Azimut`, `Astro_Sunrise`, `Astro_Sunset`
- Testen: Bei Tag Elevation >0°, nachts <0°

#### 1.3 OpenWeatherMap-Binding installieren
**Priorität:** HOCH | **Aufwand:** 30 Min

- API-Key holen: [openweathermap.org/api](https://openweathermap.org/api) (kostenlos, 1000 Calls/Tag)
- Bridge konfigurieren mit API-Key
- Items anlegen: `Wetter_Cloudiness`, `Wetter_Wind`, `Wetter_Temp_Forecast`

#### 1.4 Persistence erweitern (rrd4j)
**Priorität:** HOCH | **Aufwand:** 15 Min

- Datei bearbeiten: `/etc/openhab/persistence/rrd4j.persist`
- Strategien hinzufügen: `everyMinute` + `forecast`
- Items-Gruppen ergänzen: PV, Temps, Heizung, Fenster, Verbraucher
- OpenHAB neu starten
- Prüfen: MainUI → Analyzer → Items mit Charts sichtbar

---

### Phase 2: Basis-Automatisierung (Woche 3-4)

#### 2.1 Heizung: Einfache Solar-Reduktion
**Priorität:** HOCH | **Aufwand:** 1h

- Rule erstellen (`rules/heizung_solar.rules`)
- Startwerte: `pvPower > 1000W`, `elevation > 20°`, Vorlauf -5°C
- 1 Woche testen, Log analysieren
- Parameter anpassen basierend auf Raumtemp-Reaktion

**Logik:**
```javascript
wenn (pvPower > 1000 && elevation > 20) {
  Vorlauf = 35°C
} sonst {
  Vorlauf = 40°C
}
```

#### 2.2 Wallbox: PV-gesteuertes Laden
**Priorität:** HOCH | **Aufwand:** 45 Min

- Rule erstellen (`rules/wallbox_pv.rules`)
- Schwellwerte: Batterie >80%, PV-Überschuss >3kW, `PV_Energy_Remain > 5 kWh`
- Testen mit Auto angesteckt
- Optional: Dynamische Stromstärke 6-16A implementieren

#### 2.3 Spülmaschine: Timing-Empfehlung
**Priorität:** MITTEL | **Aufwand:** 30 Min

- Item anlegen: `String Spuelmaschine_Empfehlung`
- Notification-Rule (3x täglich Check)
- Alexa-Integration testen
- Optional: Smart Plug für Auto-Start

---

### Phase 3: Datensammlung & KI-Analyse (Woche 5-6)

#### 3.1 Logging aktivieren (2 Wochen Datensammlung)
**Priorität:** MITTEL | **Aufwand:** 1h Setup

- Persistence prüfen (Phase 1.4 abgeschlossen)
- Zusätzliche Items ergänzen:
  - Lag-Items: `Raum_Temp_1h_ago`
  - Fenster-Status-Aggregate: `Group:Contact Fenster_EG_Sued`
  - Heizpuffer-Differenz: `WP_Puffer_Delta`
- 14 Tage laufen lassen ohne Änderungen

#### 3.2 CSV-Export einrichten
**Priorität:** MITTEL | **Aufwand:** 1h

- Python-Script erstellen: `/home/openhab/scripts/export_csv.py`
- Abhängigkeiten installieren: `sudo apt install python3-pandas python3-requests`
- Test-Run durchführen
- Rule für wöchentlichen Export (cron sonntags)

**Script exportiert:**
- `PV_Power_Estimate`
- `Wechselrichter_Solarleistung`
- `Astro_Sun_Elevation`
- Raumtemperaturen
- Außentemperatur
- Bewölkung

#### 3.3 KI-Analyse durchführen
**Priorität:** MITTEL | **Aufwand:** 2h

- CSV hochladen zu ChatGPT Plus / Claude / Perplexity
- Korrelationen analysieren: PV vs. Raumtemp-Delta
- Feature Importance ermitteln
- Schwellwerte vorschlagen: PV > ?W & Elevation > ?° → Heizung -?°C (R² >0.7)
- Plots erstellen: Temp vs. PV über Zeit
- OpenHAB Rule-Code generieren mit optimierten Parametern
- Ergebnisse dokumentieren

---

### Phase 4: Fortgeschrittene Automation (Woche 7-8)

#### 4.1 Zentrale Prioritäts-Logik
**Priorität:** HOCH | **Aufwand:** 2h

- Master-Items anlegen: `PV_Manager_Status`, `PV_Manager_Priority`
- Priorisierungs-Rule implementieren
- Testen: Verschiedene Szenarien (Batterie voll/leer, Auto da/weg)

**Priorisierung:**
1. Batterie laden (bis 80%)
2. Warmwasser-Boost (bis 60°C)
3. Wallbox laden
4. Haushaltsgeräte (Spülmaschine, Waschmaschine, Trockner)
5. Einspeisung

#### 4.2 Waschmaschine/Trockner Sequenzierung
**Priorität:** MITTEL | **Aufwand:** 1h

- Power-Monitoring-Rule (Waschmaschine fertig → Notification)
- Trockner-Timing basierend auf `PV_Energy_Remain`
- Morgen-Forecast für optimales Timing

#### 4.3 Warmwasser-Boost
**Priorität:** HOCH | **Aufwand:** 30 Min

- Rule: Überschuss >2kW + Batterie >95% → `Waermepumpe_Solar_Power_Setpoint`
- Schwellwert: Warmwasser <50°C
- Hysterese einbauen (Schaltverzögerung 10 Min)

---

### Phase 5: UI & Monitoring (Woche 9)

#### 5.1 MainUI Dashboard erstellen
**Priorität:** MITTEL | **Aufwand:** 2h

- Seite "PV Energiemanagement" anlegen
- Widgets:
  - PV-Prognose vs. Ist (Chart)
  - `PV_Manager_Status` Card
  - Verbraucher-Übersicht (Wallbox, Haushaltsgeräte)
  - Heizung-Status (Vorlauf, Sollwert, Solar-Modus)
  - Trend-Lines für `PV_Energy_Remain`, `Batterie_SOC`

#### 5.2 Notifications einrichten
**Priorität:** NIEDRIG | **Aufwand:** 30 Min

- Alexa-Ankündigungen: Spülmaschine/Trockner optimal
- Optional: Telegram/Mail-Binding für wichtige Ereignisse
- Batterie-Warnung <10%, PV-Prognose-Fehler

---

### Phase 6: Optimierung & Hardware (Woche 10+)

#### 6.1 Fehlende Hardware nachrüsten
**Priorität:** NIEDRIG | **Aufwand:** variabel

**Bodentemperatur-Sensoren (DS18B20 via GPIO):**
- EG Wohnzimmer, Büro (Süd-Seiten prioritär)
- Kosten: ~10€/Sensor, Aufwand 2h/Sensor

**Backofen-Monitoring (Shelly 1PM hinter Herd):**
- Kosten: ~20€, Aufwand 1h Installation

**Rollläden EG Büro Süd (falls nicht vorhanden):**
- Solar-Gain-Kontrolle, Kosten ~150€

#### 6.2 Lokales ML-Modell (fortgeschritten)
**Priorität:** NIEDRIG | **Aufwand:** 4h

- scikit-learn auf Pi installieren
- Training-Script schreiben (RandomForest Regression)
- Wöchentliches Auto-Training via Cron
- Predict-Werte als Items zurückschreiben

#### 6.3 Feintuning nach 4 Wochen Betrieb
**Priorität:** HOCH | **Aufwand:** 2h

- Logs analysieren: Welche Rules triggern zu oft/selten?
- Schwellwerte anpassen (±10% iterativ)
- R²-Wert prüfen: <0.7 → mehr Features loggen
- Einsparung berechnen: Vergleich Netzstrom vorher/nachher

---

## Quick-Start (Minimal-Setup für sofortigen Nutzen)

**Wenn nur 1 Wochenende Zeit:**

1. SolarForecast + Astro installieren (1.1 + 1.2) → 1h
2. Persistence aktivieren (1.4) → 15 Min
3. Heizung Solar-Reduktion (2.1) → 1h
4. Wallbox PV-Steuerung (2.2) → 45 Min

**→ Sofort 300-500€/Jahr Einsparung aktiv**

**Für volles System:**

→ Phasen 1-4 durcharbeiten (ca. 8 Wochen mit Datensammlung)

---

## Konfigurationsbeispiele

### SolarForecast Thing

```
Bridge solarforecast:fs-site:gaildorf [location="48.95,9.99"] {
    Thing fs-plane:sued [azimuth=180, declination=35, kwp=7.0, refreshInterval=30]
}
```

### Items

```
Number:Power PV_Power_Estimate "PV Prognose [%.0f W]"
Number:Energy PV_Energy_Remain "PV Rest heute [%.1f kWh]"
Number:Energy PV_Energy_Tomorrow "PV morgen [%.1f kWh]"
Number:Energy PV_Energy_Actual "PV Ist heute [%.1f kWh]"

Number:Angle Astro_Sun_Elevation "Sonnenhöhe [%.1f °]"
Number:Angle Astro_Sun_Azimut "Sonnenrichtung [%.0f °]"
DateTime Astro_Sunrise "Sonnenaufgang"
DateTime Astro_Sunset "Sonnenuntergang"

Number:Dimensionless Wetter_Cloudiness "Bewölkung [%d %%]"
Number:Speed Wetter_Wind "Wind [%.1f m/s]"
Number:Temperature Wetter_Temp_Forecast "Außentemp Prognose"
```

### Heizungs-Rule (Basis)

```javascript
rule "Heizung Solar-Reduktion"
when
    Item PV_Power_Estimate changed or
    Item Astro_Sun_Elevation changed or
    Time cron "0 */30 * * * ?"
then
    val pvPower = PV_Power_Estimate.state as Number
    val elevation = Astro_Sun_Elevation.state as Number
    
    if(pvPower > 1000 && elevation > 20) {
        WaermepumpeHeizkreisFussbodenVorlauftemperaturSollwert.sendCommand(35)
        logInfo("Heizung", "Vorlauf reduziert: PV {}W, Sonne {}°", pvPower, elevation)
    } else {
        WaermepumpeHeizkreisFussbodenVorlauftemperaturSollwert.sendCommand(40)
    }
end
```

---

## Python CSV-Export Script

```python
import requests, json, pandas as pd, sys
from datetime import datetime, timedelta

url = "http://localhost:8080/rest/persistence/items"
items = ["PV_Power_Estimate", "Wechselrichter_Solarleistung", "Astro_Sun_Elevation", 
         "EGWohnzimmerRaumtemp_Temperature", "WaermepumpeAussentemperatur", "Wetter_Cloudiness"]

data = []
for item in items:
    resp = requests.get(f"{url}/{item}?serviceId=rrd4j&starttime={(datetime.now()-timedelta(days=14)).isoformat()}")
    data.append(resp.json())

df = pd.json_normalize(data)
df.to_csv("/tmp/openhab_pv_data.csv", index=False)
print("Export complete: /tmp/openhab_pv_data.csv")
```

---

## KI-Analyse Prompt

```
Analysiere diese OpenHAB-Daten (2 Wochen):
- Korrelationen: PV vs. Raumtemp-Delta, kontrolliert für Außentemp
- Feature Importance: Welche Variable wichtigste für Temp-Anstieg?
- Schwellwerte vorschlagen: PV > ?W & Elevation > ?° → Heizung -?°C (R² >0.7)
- Plots: Temp vs. PV über Zeit, Residual-Analyse
- OpenHAB Rule-Code generieren mit optimierten Parametern
```

---

## Erwartete Ergebnisse

### Einsparungen

- **Jahr 1:** 800–1500 € durch Eigenverbrauchsoptimierung
- **Heizung:** 15-20% Reduktion Netzstrom
- **Wallbox:** 90% PV-Strom bei guter Planung
- **Haushaltsgeräte:** 60-70% PV-Strom durch Timing

### Komfort

- Automatische Optimierung ohne manuelle Eingriffe
- Vorhersehbare Raumtemperaturen durch FBH-Vorsteuerung
- Intelligente Geräte-Empfehlungen via Alexa/Notifications

### System-Zuverlässigkeit

- Failsafes: Mindest-Vorlauftemperatur 35°C
- Wetterabhängige Rückfallebenen
- Logging aller Entscheidungen für Debugging

---

## Ressourcen

- [OpenHAB SolarForecast Binding](https://www.openhab.org/addons/bindings/solarforecast/)
- [OpenHAB Astro Binding](https://www.openhab.org/addons/bindings/astro/)
- [OpenWeatherMap API](https://openweathermap.org/api)
- [Forecast.Solar](https://forecast.solar/)
- [OpenHAB Persistence rrd4j](https://www.openhab.org/addons/persistence/rrd4j/)

---

## Status

**Erstellt:** 17.02.2026  
**Version:** 1.0  
**Standort:** Gaildorf, Baden-Württemberg (48.95°N, 9.99°E)  
**System:** OpenHAB 5 auf Raspberry Pi 4B  

---

## Lizenz

Dieses Projekt ist für private Nutzung konzipiert.
