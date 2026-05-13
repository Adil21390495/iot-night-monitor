# IoT Night-Time Monitoring System for Residential Care

**BEng (Hons) Electrical and Electronic Engineering — Individual Project**  
Manchester Metropolitan University | 2025–26

---

## Overview

A low-cost, non-wearable IoT system for detecting behavioural and environmental anomalies in residential care environments during night-time hours. The system combines multi-modal sensing, rule-based anomaly classification, and an LLM-generated plain-language reporting layer to provide actionable decision support for care staff during overnight shifts.

**Total hardware cost: ~£45** — over 90% less than commercial alternatives (Tunstall, Careium: £500–£2,000/room).

---

## Screenshots

### Dashboard — live room monitoring, night shift report, and history view
![Night Monitor Dashboard](screenshot-dashboard.png)

### Settings & admin panel — configurable night shift hours, alert tiers, manager view
![Settings and Admin Panel](screenshot-settings.png)

---

## System Architecture

```
Sensors (PIR + DHT22 + Door)
        │
        ▼
Raspberry Pi Pico W  ──(HTTP/Wi-Fi)──▶  ThingSpeak Cloud
        │                                       │
        │                               Time-series logging
        │                               (temp, humidity,
        │                                motion, door events)
        ▼
Rule-based Anomaly Detection
  Normal / Attention / Concern
        │
        ▼
LLM Explanation Layer
  (structured prompt → plain-English Night Shift Report)
        │
        ▼
React + TypeScript Dashboard
  (multi-client, multi-room, real-time status)
```

---

## Hardware Components

| Component | Purpose | Cost |
|---|---|---|
| Raspberry Pi Pico W | Microcontroller + Wi-Fi | ~£6 |
| DHT22 | Temperature + humidity (GPIO 15) | ~£10 |
| PIR Motion Sensor | Movement detection (GPIO 2) | ~£10 |
| Magnetic Door Sensor | Bathroom activity inference (GPIO 10) | ~£10 |
| Wiring + breadboard | Prototyping | ~£5 |
| **Total** | | **~£45** |

**Design rationale:** Raspberry Pi Pico W selected over ESP32 (higher idle current, ~2× cost) and Raspberry Pi 4 (excessive compute, power, and cost for this application). Non-wearable sensor design chosen to eliminate compliance issues common in elderly populations with wearable devices.

---

## Anomaly Detection Logic

Rule-based threshold classification implemented in MicroPython on the Pico W:

```python
def classify(motion, door, temp, humidity):
    if motion > 18 or door > 6 or temp > 24:
        return "Concern"
    elif motion > 10 or door > 3 or temp > 22.5 or humidity > 65:
        return "Attention"
    else:
        return "Normal"
```

**Threshold derivation:**
- Motion thresholds based on published nocturnal activity norms for elderly individuals (<10 significant events under undisturbed sleep)
- Temperature bounds from NHS recommended overnight range for elderly residents: 16–24°C; Attention triggered at 22.5°C
- Humidity Attention threshold: 65% (comfort threshold above typical nocturnal baseline)

**Design rationale — rule-based over ML:** Deep learning models (cf. Cejudo et al., 2025) achieve higher detection ceilings but require computational resources incompatible with embedded deployment and produce non-interpretable outputs. Rule-based classification provides full decision transparency, zero training data requirement, and deterministic auditable behaviour — appropriate priorities for a care environment where caregiver trust and false-alarm cost are critical.

---

## Evaluation Results

Evaluated across eight controlled test scenarios (5 Normal, 2 Attention, 1 Concern):

| Metric | Result |
|---|---|
| Total scenarios | 8 |
| Correctly classified | 7 / 8 |
| Detection accuracy | **88%** |
| False positives | **0** |
| False negatives | 1 (Scenario 4 — compound threshold) |

**Scenario breakdown:**

| # | Description | Motion | Door | Expected | Detected | Result |
|---|---|---|---|---|---|---|
| 1 | Baseline quiet night | 0 | 0 | Normal | Normal | ✓ |
| 2 | Light activity | 2 | 1 | Normal | Normal | ✓ |
| 3 | Cool quiet night | 1 | 0 | Normal | Normal | ✓ |
| 4 | Moderate — 8 motion, 3 door | 8 | 3 | Attention | Normal | ✗ FN |
| 5 | Elevated — 15 motion, 5 door, 68% humidity | 15 | 5 | Attention | Attention | ✓ |
| 6 | Concern — 22 motion, 8 door, 22.8°C | 22 | 8 | Concern | Concern | ✓ |
| 7 | Moderate night — 5 motion, 2 door | 5 | 2 | Normal | Normal | ✓ |
| 8 | Completely quiet | 0 | 0 | Normal | Normal | ✓ |

**False negative analysis (Scenario 4):** The motion threshold of 10 was set conservatively to protect against false positives across Normal scenarios. This rendered the model insufficiently sensitive to moderate compound events. In a clinical context, a false negative carries greater risk than a false positive — an undetected deterioration is more dangerous than an unnecessary welfare check. Threshold recalibration to motion ≥ 7 and door ≥ 2 is the recommended adjustment for future iterations.

**Statistical caveat:** With n=8 scenarios, the 88% figure carries inherent uncertainty — a single additional misclassification would reduce accuracy to 75%. These results are presented as indicative of prototype performance under controlled conditions, not as a statistically validated benchmark. A minimum of 50–100 scenarios across varied resident profiles would be required for generalised accuracy claims.

---

## Data Pipeline

- Sensor data sampled at 15-second intervals, timestamped and structured into records
- Transmitted via HTTP to ThingSpeak across four dedicated channel fields: temperature, humidity, motion events, door events
- Firebase evaluated during design phase; ThingSpeak selected for simpler API and direct Wokwi simulation compatibility
- Software-level PIR debounce filtering applied to eliminate false triggers from thermal fluctuations and air movement
- Door sensor debounce logic prevents rapid state changes being logged as multiple events

---

## LLM Explanation Layer

Detected anomalies are formatted into structured prompts and passed to an LLM API. Predefined prompt templates ensure output consistency. Example output for a Concern scenario:

> *"Night Shift Report — 02:15. Resident activity has been elevated throughout the night. Motion events are significantly above baseline (22 events recorded). Bathroom visits are frequent (8 events). Room temperature is slightly above the recommended comfort range at 22.8°C. Recommend a welfare check at earliest opportunity."*

This directly addresses the interpretability gap identified across reviewed literature: no prior system combined non-wearable sensing with night-time constraint and LLM plain-language output in a single deployable architecture.

---

## Dashboard

Built using React + TypeScript via bolt.new. Features:
- Multi-client selection (monitor multiple residents)
- Multi-room view (Room 1, Room 2, etc.)
- Real-time environmental panels (temperature, humidity)
- Activity metrics display (motion count, door count)
- High-level status indicator: **All Normal / Attention / Concern**
- Night Shift Report panel (LLM output)

Design principle: minimise cognitive load for caregivers operating during night shifts. No technical jargon in user-facing output.

---

## Literature Context

This project addresses three limitations identified across all reviewed studies:

| Gap | Source | How addressed |
|---|---|---|
| Detection without explanation | Shahid et al. (2022), Ali et al. (2025) | LLM Night Shift Report layer |
| High FP rates from daytime noise | Lai et al. (2025) — >30% FP in full-day systems | Night-time constraint |
| Binary classification insufficient | Efendi et al. (2025) — stable/unstable only | Three-tier Normal/Attention/Concern |
| Black-box deep learning | Cejudo et al. (2025) | Rule-based transparent classification |

---

## Limitations

- Evaluation conducted under controlled simulation conditions, not live deployment
- Thresholds manually configured; not learned from real nocturnal behavioural data
- DHT22 not formally calibrated against a reference instrument
- PIR detection range not systematically characterised
- Dashboard not tested with real caregivers — usability assessed through functional testing only
- Real-world deployment would require: ethics approval, DPIA (UK GDPR Article 35), informed consent under Mental Capacity Act 2005, and DTAC assessment

---

## Regulatory Context

- **Care Act 2014** — statutory duty of care during overnight periods
- **NHS Long Term Plan (2019)** — technology adoption in social care as strategic priority
- **UK GDPR Article 9** — behavioural health data classified as special category data
- **Mental Capacity Act 2005** — consent requirements for monitoring vulnerable residents

---

## Tech Stack

- **Firmware:** MicroPython on Raspberry Pi Pico W
- **Simulation:** Wokwi (IoT circuit simulator)
- **Cloud:** ThingSpeak (4-field channel: temp, humidity, motion, door)
- **Dashboard:** React + TypeScript (bolt.new)
- **LLM API:** OpenAI-compatible endpoint for Night Shift Report generation
- **Version control:** Git

---

## References

Full bibliography available in the project dissertation. Key sources:

- Cejudo et al. (2025) — deep learning anomaly detection, *Sensors*
- Chifu et al. (2022) — probabilistic behavioural modelling, *Applied Sciences*
- Lai et al. (2025) — narrative review of dementia monitoring, *Journal of Applied Gerontology*
- Shahid et al. (2022) — non-wearable IoT framework, *IEEE Access*
- Xiong et al. (2024) — LLM integration in care settings, *arXiv*

---

*Ethics approval granted by Dr Umar Raza, 04/01/2026.*
