# WatDet - Smart Water Contamination Detection

> Smart Monitoring. Safer Tomorrow.
> K! Hacks 4.0 | Track: To Assure Clean and Safe Drinking Water | Team Rapid Prototype | Bannari Amman Institute of Technology

## 1. Overview
WatDet is a low-cost IoT device plus mobile app that checks tap or borewell water at the source, detects abnormal readings with AI/ML, alerts users instantly, filters the water, and helps citizens report suspected contamination to the responsible authorities.

## 2. Problem
- Water quality is rarely monitored, so contamination goes unnoticed and affects public health.
- Laboratory testing is slow, costly and not available everywhere.
- Citizens have no quick, trusted channel to report suspected contamination.
- Authorities lack recorded, location-tagged data for decisions.

## 3. Key Features
- Live readings of pH, TDS, turbidity, temperature and chlorine from a flowing tap or borewell.
- AI/ML anomaly detection that flags abnormal readings and the likely contamination type.
- Three alert levels: SAFE, WARNING, DANGER.
- Smart alerts: in-app push, SMS, and an AI-generated voice call for extreme conditions.
- Auto-report and complaint system that sends reports to the respective department officials.
- Safe-usage tips tailored to the detected condition (for example: boil, filter, do not drink).
- Multi-stage filtration unit triggered automatically on abnormal readings.
- History dashboard of recorded data for trends and decisions.
- Source-to-tap pinpointing using device location tags.
- Crowd + sensor verification: citizen complaints are cross-checked with hardware data.

## 4. System Architecture
```
Water Source -> Sensor Layer -> ESP32 -> Wi-Fi -> Firebase + AI/ML -> Mobile App
 (tap/borewell)  (5 sensors)   (process) (HTTPS)  (store, detect)    (alerts, complaints)
                                  |                                       |
                                  +-> Relay -> Filtration unit            +-> Authority report (SMS / email / call)
```

## 5. Prototype Model

### 5.1 Physical flow
```
 [Water Inlet]
      |
 [Sensor Chamber: pH | TDS | Turbidity | Temp | Chlorine]
      |                         |
      |                  analog/digital signals
      |                         v
      |                   [ESP32 Controller] --- Wi-Fi ---> [Firebase] ---> [WatDet App]
      |                         |
      |                  relay + buzzer + LED
      v                         |
 [Valve / Pump Relay] <---------+
      |
 [Sediment] -> [Activated Carbon] -> [Ion Exchange] -> [RO Membrane]
      |
 [Pure and Safe Water Outlet]
```

### 5.2 Components (bill of materials)
| Part | Purpose |
|---|---|
| ESP32 DevKit | Controller with built-in Wi-Fi |
| Analog pH sensor | Acidity / alkalinity |
| SEN0189 | Turbidity (cloudiness) |
| Gravity analog TDS sensor | Total dissolved solids |
| Chlorine / ORP sensor | Chlorine residual (or oxidation proxy) |
| DS18B20 | Water temperature |
| 5V relay + small pump or solenoid valve | Diverts water to the filtration unit |
| Buzzer + 3 LEDs | Local SAFE / WARN / DANGER indication |
| 4.7 kOhm resistor, voltage divider resistors | DS18B20 pull-up, turbidity level shifting |
| 4-stage filter set | Sediment, carbon, ion exchange, RO |

### 5.3 Suggested pin map (ESP32)
| Signal | Pin |
|---|---|
| pH (analog) | GPIO34 |
| TDS (analog) | GPIO35 |
| Turbidity (analog) | GPIO32 |
| Chlorine / ORP (analog) | GPIO33 |
| DS18B20 (1-Wire) | GPIO4 |
| Relay | GPIO26 |
| Buzzer | GPIO27 |
| LEDs | GPIO25, GPIO14, GPIO12 |

Hardware notes:
- ESP32 ADC pins accept 3.3 V max. SEN0189 can output up to 4.5 V, so use a voltage divider.
- Use ADC1 pins (GPIO32-39); ADC2 does not work while Wi-Fi is on.
- Add a 4.7 kOhm pull-up between DS18B20 data and 3.3 V.
- Calibrate pH with buffer solutions (4.0, 7.0) and TDS with a known-ppm solution before use.
- Keep electronics in a sealed enclosure, away from water splash.
- Use a flow chamber so sensors always touch fresh, steady flow.

### 5.4 Alert logic (starter thresholds)
| Parameter | SAFE | WARNING | DANGER |
|---|---|---|---|
| pH | 6.5 - 8.5 | 6.0-6.5 or 8.5-9.0 | outside 6.0 - 9.0 |
| TDS (ppm) | below 500 | 500 - 1000 | above 1000 |
| Turbidity (NTU) | below 1 | 1 - 5 | above 5 |
| Chlorine (mg/L) | 0.2 - 1.0 | 0.1-0.2 or 1.0-4.0 | below 0.1 or above 4.0 |

These are starting values taken from common drinking-water guidance (BIS IS 10500 / WHO). Verify against the latest standard before real use.

- WARNING: app push + safe-usage tip + log event.
- DANGER: push + SMS + AI voice call + relay ON (route through filtration) + authority report.

### 5.5 Firmware skeleton (Arduino IDE)
```cpp
#include <WiFi.h>
#include <OneWire.h>
#include <DallasTemperature.h>

OneWire ow(4); DallasTemperature temp(&ow);
const int PH=34, TDS=35, TURB=32, CL=33, RELAY=26;

float readAnalog(int pin){            // average 20 samples
  long s=0; for(int i=0;i<20;i++){ s+=analogRead(pin); delay(5);} 
  return (s/20.0)*3.3/4095.0;         // volts
}

void setup(){
  Serial.begin(115200); temp.begin();
  pinMode(RELAY, OUTPUT);
  // WiFi.begin(SSID, PASS); Firebase init here
}

void loop(){
  temp.requestTemperatures();
  float t = temp.getTempCByIndex(0);
  float ph = readAnalog(PH);   // convert with calibration curve
  float tds = readAnalog(TDS); // convert with calibration curve
  float tb = readAnalog(TURB);
  float cl = readAnalog(CL);
  // 1) classify SAFE/WARN/DANGER  2) set relay  3) push JSON to Firebase
  delay(5000);
}
```

## 6. Software Stack
- Arduino IDE: ESP32 firmware.
- Python: AI/ML anomaly detection (scikit-learn: Isolation Forest, Random Forest).
- Firebase: stores sensor data, alerts, reports and complaints.
- VS Code: app backend and web services.
- Figma: UI/UX design.
- AI API: generates alert text and voice-call scripts.
- SMS / voice provider (for example Twilio) for text and calls.

## 7. AI/ML Plan
- Stage 1: rule-based thresholds (works from day one).
- Stage 2: anomaly detection on rolling windows of readings to catch sudden drift.
- Stage 3: classifier that maps reading patterns to a likely cause (high turbidity + high TDS = possible sewage or runoff).
- Use device calibration offsets and smoothing to reduce false alarms.
- Sensors flag abnormal parameters only; they cannot identify exact pathogens. Suspected cases need lab confirmation.

## 8. App Screens
- Home: live readings with SAFE / WARN / DANGER badge.
- Details: per-parameter chart and history.
- Alerts: list of events with safe-usage tips.
- Report: complaint form with photo, location and auto-attached sensor data.
- Authorities: department contacts and escalation status.

## 9. Suggested Repository Structure
```
watdet/
|-- firmware/        ESP32 code (Arduino)
|-- ml/              Python models, training notebooks, sample data
|-- backend/         API, Firebase rules, alert service
|-- app/             Mobile app source
|-- hardware/        circuit diagram, pin map, 3D model files
|-- docs/            slides, reports, images
|-- README.md
```

## 10. Getting Started
1. Clone the repository and open `firmware/` in Arduino IDE.
2. Install ESP32 board package, OneWire and DallasTemperature libraries.
3. Wire the sensors using the pin map in section 5.3.
4. Create a Firebase project and add its keys to a local config file (never commit keys).
5. Calibrate each sensor and flash the ESP32.
6. Run the Python model in `ml/` and the backend in `backend/`.
7. Open the app and confirm live readings appear.

## 11. Roadmap
- [ ] Phase 1: sensor prototype + calibration
- [ ] Phase 2: Firebase pipeline + rule-based alerts
- [ ] Phase 3: AI/ML anomaly detection
- [ ] Phase 4: mobile app + complaint system
- [ ] Phase 5: filtration automation + SMS and voice calls
- [ ] Phase 6: field pilot and map view of contamination points

## 12. Feasibility
- Technical: ESP32, common sensors, Firebase, Python.
- Economic: affordable parts and open-source software.
- Operational: simple app with real-time readings and automated alerts.
- Schedule: AI model, app development, testing and verification in phases.
- Limits: sensor drift, calibration need, water splash on electronics.

## 13. Unique Selling Points
- Source-to-tap pinpointing of contamination.
- Actionable safe-usage guidance.
- Crowd + sensor dual verification.
- Faster automated escalation to authorities.
- Low-cost, scalable, with SMS and voice-call alerts.

## 14. Social Impact
- Health: less exposure to unsafe water.
- Environment: awareness of water pollution.
- Technology: smarter water management.
- Community: citizens can report problems.
- Detection: contamination found early.

## 15. Market and Business
- Target: municipalities, government bodies, industries, disaster agencies, households, campuses.
- Market size from the submission: India Rs 5,000+ Cr, global $20+ Billion (add a source).
- Model: B2B / B2G device sales + subscription; free citizen reporting app (freemium); analytics services and grants.

## 16. References
1. Forhad, H. M., et al. (2024). IoT Based Real-Time Water Quality Monitoring System in Water Treatment Plants (WTPs). Heliyon, 10(23). https://doi.org/10.1016/j.heliyon.2024.e40746
2. IoT-Based Water Monitoring Systems: A Systematic Review. (2022). Water, 14(22), 3621. https://doi.org/10.3390/w14223621
3. Advances in Machine Learning and IoT for Water Quality Monitoring: A Comprehensive Review. (2024). Heliyon, 10(6). https://doi.org/10.1016/j.heliyon.2024.e27920
4. Bandara, R. M. P. N. S., Jayasinghe, A. B., & Retscher, G. (2025). The Integration of IoT Sensors and Location-Based Services for Water Quality Monitoring. Sensors, 25(6), 1918. https://doi.org/10.3390/s25061918

## 17. Team
| Role | Name | Dept | Year | College |
|---|---|---|---|---|
| | | | | Bannari Amman Institute of Technology |
