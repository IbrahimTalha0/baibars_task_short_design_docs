# Baibars Task — Agricultural Drone Predictive Maintenance System

> ⚠️ **Note:** These repositories was developed in 3 day as part of an interview task. / _Bu repolar, bir mülakat görevi (interview task) kapsamında 3 gün içerisinde geliştirilmiştir._

![Version](https://img.shields.io/badge/version-1.0-blue.svg)
![Python](https://img.shields.io/badge/Python-Data_Generation-yellow.svg)
![Dart](https://img.shields.io/badge/Dart-Diagnostic_Engine-0175C2.svg)
![Flutter](https://img.shields.io/badge/Flutter-UI-02569B.svg)

An end-to-end predictive maintenance and diagnostic system designed for agricultural drones. This project provides fast, explainable, and offline-first drone diagnostics for field technicians with limited technical expertise.

## 📖 Overview

The system is built on a **local-first AI proxy architecture** and consists of three main pipelines:

1. **Synthetic Telemetry Generation (Python):** Simulates realistic drone telemetry, including motor wear, environmental factors (wind, temperature), and battery degradation over time.
2. **Three-Tier Rule-Based Diagnostic Engine (Dart):** A fast, deterministic first-pass engine that evaluates telemetry data, calculates confidence scores, and decides whether an escalation to a Large Language Model (LLM) is necessary.
3. **Structured Result Interface (Flutter):** A user-friendly, light-themed mobile interface designed for outdoor visibility, presenting diagnostic findings, health scores, and risk levels.

## ✨ Key Features

- **Multi-Level Diagnostics:**
  - _Level 1 (Sub-Test):_ Analyzes individual flight modes (Unloaded, Loaded, Autonomous).
  - _Level 2 (Cross-Test):_ Detects conflicting signals and patterns across different flight modes.
  - _Level 3 (Drone-Level):_ Identifies chronic patterns, battery trends, and overall health scores across the drone's entire lifecycle.
- **Token Optimization:** Acts as a local "first responder" to filter and summarize data, reducing LLM token costs by 5x-10x by only escalating complex or low-confidence cases.
- **Contextual Rule Relaxation:** Dynamically adjusts thresholds based on environmental conditions (e.g., relaxing GPS stability thresholds in high winds).
- **Explainable AI:** Every finding is linked to a specific physical component and actionable maintenance step.

## 🏗 Architecture

### 1. Data Generation Layer (Python)

Simulates realistic physical behaviors, including:

- Uneven motor wear and thermal memory (Newton's law of cooling).
- Environmental impacts (sinusoidal temperature changes, stochastic wind).
- Battery discharge rates scaling with payload and flight modes.

### 2. Diagnostic Engine (Dart)

Evaluates 37 different diagnostic rules across 3 service levels. It calculates a weighted **Health Score** based on:

- Motor (40%)
- Battery (25%)
- Autonomous Systems (20%)
- Connectors (15%)

### 3. UI Layer (Flutter)

Built with **Riverpod** for reactive state management. The UI includes:

- Serial-based drone tracking.
- Health score cards and risk badges.
- Scrollable lists of chronic and acute diagnostic findings.

## 📚 Documentation

Detailed documentation can be found in the following files:

- 🇹🇷 [Kısa Tasarım Dokümanı (Short Design Document)](./SHORT_DESIGN_DOCUMENT_TR.md)
- ⚙️ **Core Framework:** `docs_of_core/drone_diagnostic_framework.md`
- 🎛 **Thresholds:** `docs_of_core/thresholds.md`
- 🐍 **Python Simulator Docs:** `baibars_python_script/en_docs/drone_simulator_technical_review.md`

## 🚀 Future Improvements

- **Dynamic Baselines:** Transitioning from model-wide fixed thresholds to drone-specific baselines.
- **Bayesian Confidence Updates:** Systematically increasing failure probability based on recurring warnings.
- **Fleet-Level Analysis:** Comparing drones across the fleet to detect systemic batch issues.
- **Enhanced UI:** Adding 3D drone models with component overlays for immediate visual feedback.

## 👨‍💻 Author

**İbrahim Talha Demir**  
_Date: 2026-05-02_
