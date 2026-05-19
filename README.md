# ICS Malware Detection — HAI Security Dataset

[![Python](https://img.shields.io/badge/Python-3.10+-blue)](https://python.org)
[![TensorFlow](https://img.shields.io/badge/TensorFlow-2.x-orange)](https://tensorflow.org)
[![Dataset](https://img.shields.io/badge/Dataset-HAI%20Security-red)](https://github.com/icsdataset/hai)
[![MITRE](https://img.shields.io/badge/MITRE-ATT%26CK%20for%20ICS-black)](https://attack.mitre.org/matrices/ics/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

Machine learning system for detecting attacks on Industrial Control Systems (ICS/OT)
using the HAI benchmark dataset. Covers the full pipeline from raw sensor ingestion
through feature engineering, model training, ensemble detection, and automated
MITRE ATT&CK for ICS technique attribution.

> **Background**: Built by a field service engineer working with Ignition SCADA and
> VFD systems — combining hands-on OT knowledge with self-taught ML/DL.

---

## Why ICS Security

Industrial Control Systems run power plants, water treatment, manufacturing lines,
and pipelines. Unlike enterprise IT attacks, ICS attacks can cause physical damage —
a manipulated setpoint in a boiler control loop can cause unsafe pressure buildup.
Traditional signature-based detection fails here because ICS protocols and normal
behaviour vary per installation. ML on sensor time-series is the right approach.

---

## Dataset — HAI Security Dataset (KAIST)

4 versions of the Hardware-in-the-Loop Augmented ICS dataset:

| Version | Year | Notes |
|---------|------|-------|
| hai-20.07 | 2020 | Baseline |
| hai-21.03 | 2021 | Extended attack scenarios |
| hai-22.04 | 2022 | New attack types |
| haiend-23.05 | 2023 | Latest version |

**84 sensor columns at 1 Hz** covering physical subsystems:

| Prefix | Physical Subsystem |
|--------|--------------------|
| P1 | Boiler — Steam Generation |
| P2 | Turbine — Power Generation |
| P3 | Condenser — Heat Exchange |
| C | Control Valves |
| FI | Flow Indicators |
| LI | Level Indicators |

Labels: `attack` column (0 = normal, 1 = attack)

---

## Feature Engineering

Time-series features extracted per sensor (30 sensors used):

| Feature Type | Windows / Details |
|-------------|-------------------|
| Rolling mean | 5s, 30s, 60s |
| Rolling std | 5s, 30s, 60s |
| Rolling min/max | 5s, 30s, 60s |
| Rate-of-change (velocity) | diff(1) |
| Acceleration | diff(2) |
| Local z-score deviation | (value - rolling mean) / rolling std |

Total engineered features: **589** for RF and XGBoost.

Statistical validation: KS-test per sensor to identify
most discriminative columns between normal and attack periods.

---

## Models

### Classical / Ensemble

| Model | Input | Notes |
|-------|-------|-------|
| IsolationForest | 30 raw sensors, StandardScaler | Unsupervised baseline |
| Random Forest | 589 engineered features | Supervised, class_weight=balanced |
| XGBoost | 589 engineered features | Supervised, scale_pos_weight tuned |

### Deep Learning — Sequential

| Model | Architecture | Input |
|-------|-------------|-------|
| LSTM Autoencoder | Encoder 128→64→32, Decoder 32→64→128→TimeDistributed | Sequences of 30 sensors |
| TCN-BiLSTM + Attention | TCN(64,d=1)→TCN(64,d=2)→TCN(128,d=4)→TCN(128,d=8) + BiLSTM(64) + MultiHeadAttention | Same sequences |

### TCN-BiLSTM Architecture Detail
Input (seq_len, 30 sensors)
├── TCN branch
│   ├── TCN block (64 filters, dilation=1)  causal padding
│   ├── TCN block (64 filters, dilation=2)
│   ├── TCN block (128 filters, dilation=4)
│   ├── TCN block (128 filters, dilation=8)
│   └── GlobalAveragePooling1D
│
└── LSTM branch
├── BiLSTM (64 units)
├── MultiHeadAttention
└── GlobalAveragePooling1D
│
Concatenate([TCN output, LSTM output])
Dense(64) → Dropout(0.3) → Dense(1, sigmoid)

Each TCN block: Conv1D(causal) → BN → Dropout → Conv1D(causal) → BN → residual Add

### LSTM Autoencoder — Anomaly Detection

Trained only on normal sequences (unsupervised).
Detection threshold = mean + 3σ of training reconstruction errors (MAE).
ormal  → low reconstruction error  → below threshold → NORMAL
Attack  → high reconstruction error → above threshold → ANOMALY

## Ensemble

All four model scores combined on 13,235 test sequences:
RF score (resampled to sequence length)

XGBoost score (resampled)
LSTM Autoencoder reconstruction error (normalised)
TCN-BiLSTM probability
→ Weighted average → final ensemble score
→ Optimal threshold from F1 / Fbeta search

Cross-version generalization tested across all 4 HAI versions.

---

## Analysis Pipeline

| Step | What it does |
|------|-------------|
| Data quality report | Missing values, dtype check, NaN audit |
| KS-test validation | Statistical significance per sensor |
| Sensor EDA | Normal vs attack period time-series overlay |
| Correlation delta | Heatmap of correlation change during attacks |
| Threshold search | F1 / Fbeta / balanced accuracy optimization |
| Detection delay | Seconds from attack start to first detection per window |
| False alarm analysis | FP rate with sensor context |
| Attack window detail | Pre / during / post context zoom per attack |
| Cross-version test | Generalization to held-out HAI versions |
| Subsystem attribution | SHAP top features → physical subsystem |
| MITRE mapping | Anomaly pattern → ATT&CK technique |

---

## MITRE ATT&CK for ICS Mapping

Automated attribution from sensor anomaly pattern → MITRE technique:

| Anomaly Pattern | ATT&CK Technique | Tactic |
|----------------|-----------------|--------|
| High deviation from setpoint (rolling mean) | T0836 — Modify Parameter | Impair Process Control |
| Sensor flatline (rolling std ≈ 0 for >30s) | T0838 — Modify Alarm Settings | Inhibit Response Function |
| Cross-sensor decoupling (correlation breaks) | T0855 — Unauthorized Command Message | Impair Process Control |

**Attribution flow:**
SHAP top features
→ prefix matching (P1, P2, P3, C, FI, LI)
→ physical subsystem (Boiler, Turbine, Condenser)
→ anomaly pattern type
→ MITRE ATT&CK technique
→ alert: "Boiler attack — T0836 Impair Process Control"

---

## Results

| Model | AUC | F1 | Notes |
|-------|-----|----|-------|
| IsolationForest | — | — | Unsupervised, no label needed |
| Random Forest | high | high | 589 features |
| XGBoost | high | high | 589 features |
| LSTM Autoencoder | — | — | Reconstruction threshold |
| TCN-BiLSTM | best | best | Cross-version generalizes |
| Ensemble | best | best | All four combined |

> Exact numbers vary by HAI version and threshold setting.
> TCN-BiLSTM consistently outperforms individual models
> across all 4 dataset versions.

---

## Field Context

This notebook was built by someone who services VFDs and
Ignition SCADA systems professionally. That context matters:

- P1/P2/P3 sensor naming makes physical sense — not just column labels
- Setpoint deviation means something real — not just a statistical artifact  
- Detection delay in seconds matters — 30 seconds is very different
  from 300 seconds in a real plant emergency
- Cross-version generalization reflects real OT environments
  where sensor calibration shifts over time

---

## Tech Stack

| Category | Libraries |
|----------|-----------|
| ML | scikit-learn, XGBoost |
| Deep Learning | TensorFlow/Keras, LSTM, Conv1D, MultiHeadAttention |
| Feature Engineering | pandas, numpy, rolling/diff/z-score |
| Explainability | SHAP TreeExplainer |
| Visualization | matplotlib, seaborn |
| Evaluation | sklearn.metrics, custom threshold search |
