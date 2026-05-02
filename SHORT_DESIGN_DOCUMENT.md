# Short Design Document — Drone Diagnostic Assistant

**Project:** Baibars Task — Agricultural Drone Predictive Maintenance System  
**Author:** İbrahim Talha Demir  
**Date:** 2026-05-02  
**Version:** 1.0  
**Stack:** Python (data generation) · Dart / Flutter (diagnostic engine + UI)  
**Related Repositories:**  
- [baibars_interview_task_python_script](https://github.com/IbrahimTalha0/baibars_interview_task_python_script)  
- [baibars_task_mobile_app](https://github.com/IbrahimTalha0/baibars_task_mobile_app)

---

## Table of Contents

1. [Problem Modeling](#1-problem-modeling)
   - [1.1 Data Structure and Generation Layer (Python)](#11-data-structure-and-generation-layer-python)
   - [1.2 Diagnostic Engine Architecture (Dart)](#12-diagnostic-engine-architecture-dart)
   - [1.3 Interface Layer (Flutter)](#13-interface-layer-flutter)
2. [Core Assumptions](#2-core-assumptions)
   - [2.1 Thermal Thresholds (per model)](#21-thermal-thresholds-per-model)
   - [2.2 Electrical Thresholds](#22-electrical-thresholds)
   - [2.3 Autonomous Thresholds](#23-autonomous-thresholds)
   - [2.4 Contextual Relaxation Rules](#24-contextual-relaxation-rules)
   - [2.5 Chronic Detection Thresholds](#25-chronic-detection-thresholds)
3. [Diagnostic Model Explanation](#3-diagnostic-model-explanation)
   - [3.1 Approach](#31-approach)
   - [3.2 Token Optimization](#32-token-optimization)
   - [3.3 What Can Break or Produce False Results](#33-what-can-break-or-produce-false-results)
   - [3.4 How the Engine Improves with Real Sensor Data](#34-how-the-engine-improves-with-real-sensor-data)
4. [System Design Decisions](#4-system-design-decisions)
   - [4.1 Missing Sensor Data (NO_DATA)](#41-missing-sensor-data-no_data)
   - [4.2 Conflicting Signals Between Test Modes](#42-conflicting-signals-between-test-modes)
   - [4.3 What Happens When There is a Low Confidence Score](#43-what-happens-when-there-is-a-low-confidence-score)
5. [What Would Be Improved With More Time](#5-what-would-be-improved-with-more-time)
   - [5.1 Real Sensor Data Pipeline](#51-real-sensor-data-pipeline)
   - [5.2 Uncertainty Management](#52-uncertainty-management)
   - [5.3 Maturing the AI Escalation Architecture](#53-maturing-the-ai-escalation-architecture)
   - [5.4 Interface Improvements](#54-interface-improvements)
   - [5.5 Fleet-Level Multi-Drone Chronic Analysis](#55-fleet-level-multi-drone-chronic-analysis)
6. [Lessons Learned](#6-lessons-learned)

---

## Overview

This system provides fast and explainable drone diagnostics for field technicians with low technical expertise. It covers three pipeline stages: **synthetic telemetry generation** (Python), **three-tier rule-based diagnostic engine** (Dart), and **structured result interface** (Flutter). The diagnostic architecture is intentionally designed as a _local-first AI proxy_: it performs a fully deterministic first pass and calculates its own confidence score before deciding whether to escalate to a large language model. This approach keeps the token cost per diagnosis near zero in the majority of cases.

> Reference: Full rule catalog → `docs_of_core/drone_diagnostic_framework.md` in the [`baibars_task_mobile_app`](https://github.com/IbrahimTalha0/baibars_task_mobile_app) repository  
> Threshold values → `docs_of_core/thresholds.md` in the [`baibars_task_mobile_app`](https://github.com/IbrahimTalha0/baibars_task_mobile_app) repository  
> Data generation details → `en_docs/drone_simulator_technical_review.md` or `drone_simulator_teknik_inceleme.md` in the [`baibars_interview_task_python_script`](https://github.com/IbrahimTalha0/baibars_interview_task_python_script) repository

---

## 1. Problem Modeling

### 1.1 Data Structure and Generation Layer (Python)

The generation script is built considering physical realities such as motors not wearing out equally, and a wearing motor drawing more current and heating up more. While each motor wears at different rates due to chronic trends and random manufacturing deviations, this wear is directly reflected in current consumption and temperature calculations. Specifically, the scenario of a single motor being chronically weaker in some models has also been modeled, thus simulating systematic failure behaviors realistically. Imbalance between motors creates a chain effect triggering increased vibration and decreased stability, and these relationships are calculated with interconnected formulas. Furthermore, isolated problems like bearing wear are handled in a way that affects the entire system behavior. The non-linear growth of stress on the motor with increased payload has been included in the model in accordance with real payload profiles in agricultural drone usage.

The script realistically reflects not only motor behaviors but also environmental and operational conditions. Seasonality is modeled with sinusoidal temperature changes, while wind effect is generated with stochastic distributions and included in calculations to affect all components of the system. Battery consumption, efficiency loss, and autonomous flight metrics are dynamically determined by the combined effects of payload, wind, and motor wear. Additionally, thermal memory between tests is preserved according to Newton's law of cooling, and waiting times between flights are simulated in accordance with real operations. While wear accumulated over the life cycle causes all performance metrics to deteriorate over time, edge-case scenarios such as sudden failure and missing sensor data are also included in the system. Model-based threshold values are defined specifically for different drone types to reflect operational realities; operational details such as pilot, customer, usage history, and flight planning are kept constant and consistent to preserve data integrity.

---

### 1.2 Diagnostic Engine Architecture (Dart)

The diagnostic engine is structured to be used in 3 different scenarios. This section contains information about the scenarios and the developed diagnostic engine, while the actual working method and logic of this architecture are explained in the 3. [Diagnostic Model Explanation](#3-diagnostic-model-explanation) section.

```
Level 1 — SubTestDiagnosticService
This is the service to be used to analyze only one type of test (a single loaded/autonomous/unloaded test).
  Input:  single flight mode of a single record (UL / LD / AU)
  Output: single mode findings (motor, electrical, stability, autonomous, drop)
  Rules:  ~15 independent + 3 chain rules (Chain 3, 7, 8)

Level 2 — TestDiagnosticService
This is the service to be used to analyze only one test (all loaded/autonomous/unloaded tests).
  Input:  full test record (all three modes)
  Output: inter-mode pattern findings
  Rules:  Chain 1, 4, 5, 6 + 4 independent multi-mode patterns

Level 3 — DroneDiagnosticService
This is the service to be used to analyze only one drone's tests (all or selected tests).
  Input:  all records belonging to a drone (sorted by date)
  Output: chronic patterns, trends, overall health score, risk level
  Rules:  Chain 2, 9, 10 + battery trend, flight hour threshold,
          false recovery, intermittent drop, M2–M1 residual, wear slope
```

Total: **37 different diagnostic rules** across 3 service levels, covering the 28 uniquely named scenarios in the framework document. All rules produce a `DiagnosticFinding` with `findingId`, `severity`, `category`, `description`, `confidence` (0.0–1.0), `affectedComponents`, `isChainRule`, and `isChronic` fields — these are the fields the interface directly binds to.

The health score formula weights subsystems by criticality:

| Subsystem    | Weight |
| ------------ | ------ |
| Motor        | 40%    |
| Battery      | 25%    |
| Autonomous   | 20%    |
| Connectors   | 15%    |

Each status field adds `0` (OK), `1` (WARNING), `2` (NO_DATA), or `3` (CRITICAL) penalty points.  
`Health Score = 100 − (weighted_penalty_sum / max_possible × 100)`, compressed to the [0, 100] range.

| Score  | Risk Level  | Action             |
| ------ | ----------- | ------------------ |
| 90–100 | Operational | Continue           |
| 70–89  | Low Risk    | Monitor closely    |
| 50–69  | High Risk   | Schedule maint.    |
| < 50   | Critical    | Ground immediately |

> Reference: Health score formula → `drone_diagnostic_framework.md` §3  
> Chain rule details → `drone_diagnostic_framework.md` §4

---

### 1.3 Interface Layer (Flutter)

The Flutter interface has been developed with a light theme so that it can be easily used by everyone and seen more comfortably when used outdoors under the sun.
It uses **Riverpod** for reactive state management. The diagnostic pipeline is called synchronously in the widget tree (`DroneDiagnosticService().diagnose(tests)`) — there is no async boundary in the current implementation because the rule engine is pure Dart computation without I/O. Findings are separated into `chronicFindings` and `nonChronicFindings`, sorted by severity (CRITICAL → WARNING → OK → NO_DATA), and displayed as `DiagnosticFindingCard` components. The `DroneAnalysisPage` presents a pinned health score card and risk badge at the top, followed by a summary ribbon (chronic count, other count, total findings, record count), and then a scrollable list of findings.

Navigation flow: **Home** (model list) → **Model Detail** (serial list) → **Serial Drone** (test records) → **Test Analysis** (single test, three modes) → **Drone Analysis** (drone-level chronic/trend findings).

> Reference: UI component styles → `docs_of_core/ui-styles.md`, `docs_of_core/ui-rules.md`

---

## 2. Core Assumptions

The following thresholds and rules are considered the ground truth of domain knowledge. They encode the line between "normal operational stress" and "failure requiring action".

### 2.1 Thermal Thresholds (per model)

| Model    | Motor Temp UL OK / WARN (°C) | Motor Temp LD OK / WARN (°C) | Imbalance Trigger |
| -------- | ---------------------------- | ---------------------------- | ----------------- |
| CT50     | 63 / 78                      | 76 / 90                      | spread > 12°C     |
| CT50-Pro | 63 / 79                      | 76 / 91                      | spread > 12°C     |
| CT75     | 66 / 82                      | 80 / 95                      | spread > 12°C     |
| CT100    | 68 / 84                      | 82 / 98                      | spread > 12°C     |

The 12°C imbalance spread is a strict threshold: even if each of the four motors is individually in the OK range, if the spread is > 12°C, `motor_imbalance_flag = WARNING`. This catches the onset of single motor bearing failure before the average temperature limit is exceeded.

### 2.2 Electrical Thresholds

- Battery discharge rate: CT50 → 2.5 / 4.0 %/min · CT100 → 3.2 / 5.0 %/min (scales with platform weight)
- Molex connector: 46 / 60°C (CT50) — poses a persistent CRITICAL fire risk
- Pin connector: 50 / 65°C (CT50)
- Loaded efficiency drop: 22 / 38% (CT50) — high drop is an indicator of hidden internal resistance

### 2.3 Autonomous Thresholds

- GPS deviation: 1.2 / 3.0 m — common across all models (GPS accuracy is platform-independent)
- Route completion: 96% / 88% (inverse interpretation; < 88% = CRITICAL) — common
- Sensor anomaly rate: 2.0% / 7.0% — common

### 2.4 Contextual Relaxation Rules

- **Wind > 5.0 m/s:** GPS and stability thresholds are relaxed by 15% (`ok_gps *= 1.15`; `ok_stability *= 0.85`). It is a binary gate, not a continuous function.
- **Near maximum payload (> 90% of max_payload_kg):** motor current WARNING in loaded mode is classified as an _expected warning_, not a failure signal.

### 2.5 Chronic Detection Thresholds

- **Persistence (Chain 2):** A motor is considered "chronic hottest" if it is the hottest in > 60% of all sub-test checks; the imbalance rate must exceed 40% of the records.
- **Battery trend:** Monotonically increasing discharge rate in at least 2 of 3 modes over 4+ records.
- **Wear slope:** ≥ 1.2°C per 100 flight hours in max UL motor temp (OLS linear regression; min 5 data points).
- **False recovery:** Successive ≥ 2 CRITICAL records followed by full-OK, without a recorded maintenance event.

> Reference: Full threshold tables → `thresholds.md`

---

## 3. Diagnostic Model Explanation

### 3.1 Approach

I designed the diagnostic system in Dart to detect various scenarios that might be encountered in the field at an early stage. At the core of this approach is the idea of positioning the system as a "first response layer" running in a local environment (on-device or on Baibars servers). Accordingly, due to time constraints, 28 different singular and chained scenarios were defined and simulated.

In short, the currently written diagnostic engine has been put in place to simulate the ML/AI mentioned below in certain situations. In production, this engine should give way to an AI/ML.

The target diagnostic architecture treats this engine like an _ambulance_. It is the fast, low-cost first responder that initially handles cases, evaluates them, calculates a confidence score, summarizes findings, and escalates to a large cloud model only when truly necessary. It runs offline on the device or Baibars servers, with zero or low cost per call and low latency. Then, when necessary, it hands over to a more capable model or models.

After this initial evaluation, the system summarizes the obtained data and transmits it to higher-capacity closed models (e.g., Opus 4.7) only when necessary. This ensures both that the outputs are in a specific format and compatible with the user interface, and that unnecessary token consumption for each operation is prevented. Since the locally running AI layer simplifies and optimizes the data before transmitting it, it also provides an additional efficiency gain. Through this architecture, it is aimed to achieve an optimization of approximately 5 to 10 times in total token usage.

### 3.2 Token Optimization

The most critical cost reduction principle is positioning two different AIs and sending only the necessary information to the costly one.

| Technique                                                      | Impact                                          |
| -------------------------------------------------------------- | ----------------------------------------------- |
| Send only status fields instead of raw numerical values        | Token cost reduction                            |
| Structured diagnostic summary (counts per subsystem/status)    | Prevents sending the full record history        |
| Confidence gate (< 0.60) before each LLM call                  | Eliminates unnecessary API calls                |
| Template-based report for routine output                       | LLM is only needed for complex narratives       |

> Reference: Token optimization details → `drone_diagnostic_framework.md` §6

### 3.3 What Can Break or Produce False Results

Main false diagnosis scenarios the current rule engine might produce:

**Model-wide fixed thresholds hide individual deviations.** The same threshold is applied to all CT50s; however, the "normal" range of a 450-hour CT50 and a 50-hour CT50 are practically different. A high-hour drone might constantly hover just below the threshold, and while the diagnostic considers it healthy, it might actually be approaching a critical wear threshold.

**Confidence scores are manually assigned, not empirical.** The values `confidence=0.85` for Chain 2 and `confidence=0.90` for Chain 9 are based on expert estimates, not real validation data. Considering that the confidence gate (< 0.60) triggers LLM escalation, a miscalibrated score leads to either an unnecessary LLM call (cost increase) or the suppression of a truly uncertain finding (missed diagnosis).

**Contextual relaxation is a binary gate, not a continuous function.** The threshold does not relax at 4.9 m/s wind; it suddenly relaxes by 15% at 5.1 m/s. Since wind effect is a continuous function in a real environment, this hard transition can produce false alarms or alarm suppressions at boundary values.

**Chain rules are sensitive to missing the first condition.** Chain 7 (temperature-current mismatch) depends on the `motor_current_status = CRITICAL` condition. If the current sensor also experiences a dropout, the chain is never triggered, and the hidden failure with fire risk remains invisible.

**Chronic detection requires a minimum number of records.** 4–5 records are sufficient for Chain 2 and battery trend; however, for a new drone with few flights, chronic motor deviation might not be detected at all. Even if the same failure behavior has been observed in other drones in the fleet, this information is not used in the current architecture.

**False recovery detection depends on maintenance record logging.** If a maintenance event is not logged in the database (`successive ≥ 2 CRITICAL → full-OK` pattern), the false recovery looks like a real recovery, and a critical intermittent failure is missed.

**NO_DATA propagation suppresses chain rules.** When one of the motor temperature fields is `null`, `motor_imbalance_flag = NO_DATA`, and all chain rules depending on imbalance (Chain 2, 9) are suppressed. Since imbalance, which is the earliest signal of bearing wear, is intertwined with the component most prone to sensor failure, if a real failure overlaps with a dropout, it can be silently missed.

### 3.4 How the Engine Improves with Real Sensor Data

When real telemetry data is integrated into the diagnostic engine, the following improvements become possible:

**Drone-based dynamic baselines.** The OK flights in each drone's own history create the expectation envelope (mean ± σ) specific to that unit. The threshold is applied to the individual drone, not the entire model. A reading that is normal at 50 hours might be considered an anomaly at 400 hours; while the current wear slope rule partially catches this with a linear slope, the individual baseline measures it directly.

**Empirical calibration of confidence scores.** When real maintenance results (failure confirmation, spare part replacement records) are matched with diagnostic findings, the true accuracy rate of each rule can be measured. Thus, estimated values like `confidence=0.85` are replaced with real probability scores based on empirical precision and recall rates.

**Bayesian confidence update.** Repeated WARNING findings on the same component systematically increase the probability of failure as the number of observations increases. While the current system catches this with a "chronic counter" logic, Bayesian updating proportionally adds each new observation to the previous confidence — giving an earlier warning with fewer records.

**Making the wind relaxation function continuous.** With real flight data, the wind speed–GPS deviation and wind speed–stability relationships can be modeled with regression. The binary gate (> 5.0 m/s → relax 15%) is replaced by a correction function continuously sampled to wind speed; the false alarm rate at boundary values drops.

**Time series consistency for intermittent failures.** If the same field repeatedly becomes `NO_DATA` in real sensor records, this pattern indicates a cable/connector problem. A time series consistency score wraps which component experiences dropouts how often; when this score is high, chain rules can produce a low-confidence finding instead of being suppressed by triggering with low confidence themselves.

---

## 4. System Design Decisions

### 4.1 Missing Sensor Data (NO_DATA)

If any raw value is `null`, the `ThresholdCalculator` returns `HealthStatus.noData`. The spread in the motor:

| Condition                                                            | Behavior                                                                                                                                                                          |
| -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Single field NO_DATA                                                 | `sensor_dropout` finding is produced (severity: `noData`, confidence: 0.95). Health score is penalized with weight 2 — worse than WARNING, better than confirmed CRITICAL.        |
| ≥ 3 fields NO_DATA in the same record                                | `multi_field_dropout` finding (CRITICAL, confidence: 0.90). Root cause shifts from "single sensor failure" to "bus or logger failure".                                            |
| < 2 valid motor temperatures                                         | Imbalance flag = `NO_DATA`; all chain rules depending on imbalance are suppressed.                                                                                                |
| Same field toggles OK ↔ NO_DATA across records (≥ 2 transitions)     | Level 3 `intermittent_dropout` finding. Implies a loose connector or cable discontinuity — requires a different action than a permanent sensor failure.                           |

### 4.2 Conflicting Signals Between Test Modes

Level 2 (`TestDiagnosticService`) rules exist to resolve inter-mode conflicts that Level 1 alone would misinterpret:

| Conflict Pattern                                       | Correct Interpretation                                                     | Rule                       |
| ------------------------------------------------------ | -------------------------------------------------------------------------- | -------------------------- |
| UL motor temp OK → LD motor temp CRITICAL              | Not a general motor failure, but a payload mechanism failure               | Chain 1                    |
| UL discharge OK → LD discharge CRITICAL                | Battery internal resistance only appears under high load                   | Battery Load Inconsistency |
| UL+LD stability OK; AU GPS CRITICAL + anomaly CRITICAL | Drone is mechanically sound; navigation module failure only in auto mode   | Autonomous Nav Regression  |
| AU route CRITICAL, GPS OK, anomaly OK                  | Battery depleted early — mission too long, not a navigation failure        | Chain 6                    |
| Motor temp OK, motor current CRITICAL                  | Temperature sensor failure — most dangerous hidden failure, fire risk      | Chain 7                    |

Without cross-mode correlation, Level 1 alone would produce a `motor_overheating_ld` CRITICAL finding, leading to an incorrect motor replacement when the real failure is in the propeller pitch system.

### 4.3 What Happens When There is a Low Confidence Score

Each `DiagnosticFinding` carries a `confidence` in the range 0.0–1.0. In the current implementation, this value is shown as an informational field on the `DiagnosticFindingCard` but does not gate UI actions.

**Targeted production behavior** (AI escalation architecture in §3.1):

```
confidence ≥ 0.75    → finding is shown directly to the technician; no LLM call
confidence 0.60–0.74 → finding is shown with an "uncertain" label;
                       explanation is optionally enriched with LLM

confidence < 0.60    → raw finding is not shown to a low-expertise technician;
                       escalated to a closed model (e.g., Opus 4.7);
                       only the structured explanation generated by the LLM is shown
```

This gate prevents findings like Chain 10 pilot anomaly (0.65) or Chain 4 ambiguous vibration (0.55) from misleading the technician with uncertain attributions.

---

## 5. What Would Be Improved With More Time

### 5.1 Real Sensor Data Pipeline

The first change with real telemetry would be **abandoning model-wide fixed thresholds and moving to drone-based baselines**. The historical flights in the OK range of each drone create its own mean ± σ envelope. A reading that is normal for a CT50 at 50 flight hours could be an anomaly at 400 hours — the current system only partially catches this with the wear slope rule.

The Python simulator would be replaced with a **real-time ETL pipeline** that receives telemetry from the on-board logger (via Bluetooth or cellular upload) at the end of the mission. Each record would be validated for schema compliance, temporal ordering, and physical consistency across fields before entering the diagnostic engine.

### 5.2 Uncertainty Management

The current confidence scores are manually assigned constants. A proper uncertainty system would:

1. **Propagate NO_DATA uncertainty:** The confidence of a finding derived from a metric neighboring NO_DATA automatically drops.
2. **Time-decayed confidence:** A finding from a record 90 days ago carries less diagnostic weight than yesterday's finding.
3. **Cross-record Bayesian update:** Repeated WARNING findings on the same component systematically increase the failure probability; it doesn't just trigger a separate "chronic" rule.

### 5.3 Maturing the AI Escalation Architecture

The current system is a two-layer architecture without a fully functioning **Layer 2 ML module** inserted between Layer 1 (rule engine) and Layer 3 (LLM). Developing this intermediate layer both increases diagnostic quality and deepens cost optimization by reducing the number of LLM calls.

**Time series regression for battery degradation trend.** A simple comparison is currently used to detect a monotonic increase in drain rate across records. Linear or polynomial regression produces both slope statistics and a confidence interval — enabling a proactive maintenance prediction like "if it decreases at this rate, it will reach the critical threshold in X flights".

**Clustering for pilot-based anomaly detection.** Chain 10 is currently limited to counting whether pilot-based deviation exists. K-means or DBSCAN can cluster operational metrics like drain_rate and stability_score per pilot to determine which pilot is causing the deviation and from what type of maneuver (hard landing, overload, sudden turn) this deviation originates.

**Environmental condition–anomaly correlation.** Modeling how the anomaly rate changes according to air temperature and wind speed with regression reveals component-specific environmental sensitivity like "this drone breaks down more in hot and windy conditions than others". This information transforms the current binary contextual relaxation rule into a continuous function.

**Sequence pattern matching for false recovery and intermittent failure.** Catching the `[CRITICAL × N → OK]` sequence without a maintenance record is fragile with the current threshold checking approach. An LSTM or HMM-based sequence model can learn the state transition pattern of each drone and flag abnormal state changes like "this transition is unusual for this drone — probably a false recovery".

**Intra-fleet comparative chronic detection.** Chronic detection is currently limited to single drone history. Comparing drones in the same model and usage condition (e.g., Hotelling T² or isolation forest) measures how much an individual drone deviates from the fleet median. A drone whose deviation is constantly growing can signal chronic degradation even before it crosses the threshold. This approach is especially critical for early detection of systemic batch problems across the fleet (see 5.5).

### 5.4 Interface Improvements

- **3D drone model with component overlay:** When M2 is flagged, it should be directly highlighted on the 3D mesh. The field technician instantly sees which physical part to check.
- **Time series charts per metric:** Motor temperature, battery discharge rate, and vibration should be plotted over the flight history. A visual trend is more actionable than numeric slope text.
- **Repair recommendation flow:** A checklist guiding inspection steps, estimated repair time, and required equipment based on specific findings.
- **Push notification on LLM escalation:** When the Dart engine escalates to Layer 3, the structured LLM response should be delivered to the technician via asynchronous push.
- **Fleet-level dashboard:** All drones' health scores should be ranked by risk level; proactive maintenance should be planned before failure occurs.

### 5.5 Fleet-Level Multi-Drone Chronic Analysis

Current chronic detection works on single drone history. With a fleet view, **cross-drone comparison** becomes possible: if 8 out of 12 CT50s operating at the same customer have the same M2 bias, this is not an individual drone problem, but a fleet-level signal (likely a supply batch problem). This changes both the diagnosis and the action — "do an M2 quality inspection on the whole batch" instead of "replace M2 on this drone".

---

## 6. Lessons Learned

**1. Synthetic data must reflect not only the distribution but also the causal structure.**  
The most correct decision in the simulator was to use the UL → LD → AU chain and the Newton cooling model. Generating rows independently and randomly would lead to physically impossible situations, such as a motor that is cold at the end of UL appearing hot at the beginning of LD. Causal consistency is the fundamental element that makes cross-mode correlation rules meaningful. Without this consistency, the Level 2 service would catch artificial patterns originating from the simulator, not real failures.

**2. Confidence scores should not be assigned, but validated based on data.**  
Giving `confidence=0.85` for Chain 2 and `confidence=0.90` for Chain 9 is an expert estimate rather than a measurement. In a production environment, these values need to be empirically calibrated with labeled historical results. Up to this stage, confidence scores should be evaluated as a relative confidence ranking among findings, not as absolute probabilities.

**3. In this domain, a three-tier diagnostic architecture provides the right separation.**  
The separation into single record, inter-mode, and multi-record analysis prevented scope creep in each layer. The single record layer catches acute events (sudden motor failure, connector heat rise), the cross-mode layer catches structural mismatches between test conditions (payload mechanism failure, navigation regression), and the multi-record layer catches gradual degradations not visible on a per-flight basis (chronic bearing wear, battery cell capacity loss). Gathering these structures into a single monolithic analyzer makes both interpreting the rules difficult and reduces independent testability.

**4. The "first responder local AI" approach is a strong solution in terms of cost and latency.**  
In the field, a 3-second LLM latency is not practical for a technician working next to the drone. The Dart engine works network-independently and can produce results in < 1 ms. Escalation to models like Opus 4.7 is done only in truly uncertain cases. Furthermore, since the Dart layer summarizes the findings beforehand, a narrowed and structured input is sent to the LLM instead of raw telemetry. This approach both increases speed and can reduce token usage by approximately 5×–10× compared to the "send everything to LLM" model.

**5. Explainability is not an add-on feature in this system, but a core requirement.**  
Each diagnostic finding includes a Turkish explanation linked to a specific physical component and an actionable maintenance action. This requirement directly shaped the architecture; it is the main reason why ML-based anomaly detection was not preferred despite its potential sensitivity to new failures. For example, an anomaly score of 0.87 alone is not meaningful to the technician; in contrast, a statement like "M2 motor temperature is systematically higher than other motors — bearing wear is likely" clearly shows exactly where to check.

---

_What the document covers: `lib/services/drone_diagnostic_service.dart` · `lib/services/sub_test_diagnostic_service.dart` · `lib/services/test_diagnostic_service.dart` · `lib/services/threshold_calculator.dart` · `lib/features/drone_analysis/drone_analysis_page.dart` · `baibars_python_script/generate_data_v2.py` · `docs_of_core/drone_diagnostic_framework.md` · `docs_of_core/thresholds.md` · `baibars_python_script/generate_data_v2_inceleme.md`_