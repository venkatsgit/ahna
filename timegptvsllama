## 1. Use Case Definition

As a data centre engineer, I want to monitor a **chiller system** equipped with multiple sensors (temperature, pressure, flow rate) to detect **anomalies in real-time** and **predict future failures**. The goal is proactive maintenance by spotting unusual behaviour early.

---

## 2. Current Solution: LSTM + STUMPY Pattern Matching

* **LSTM Model:**
  Trained on 4 months of 1-minute interval sensor data, predicts the next 1440 minutes (24 hours) of sensor readings.

* **STUMPY Matrix Profile:**
  Built on 3 months of historical data, used for pattern matching to identify motifs and discords (anomalies) in sliding 1-hour windows of real-time sensor data.

### Simple Solution Diagram

```
[Sensor Data Stream]
         │
         ▼
  ┌───────────────┐        ┌─────────────┐
  │Real-time Data │───────▶│Incremental  │
  │ (last 1 hour) │        │Matrix Profile│
  └───────────────┘        └─────────────┘
         │                          │
         │                          ▼
         │                  Anomaly Detection
         │
         ▼
┌─────────────────────┐
│     LSTM Model      │
│(predict next 1440m) │
└─────────────────────┘
         │
         ▼
 Forecasted Sensor Values
         │
         ▼
 [Optional] Pattern Matching on Predicted Values (Query-only)
```

---

## 3. Pitfalls of Current Approach

* **Computational Complexity:**
  Incremental matrix profile computation grows with data size, leading to heavier processing over time.

* **Matrix Profile Scaling:**
  Querying and updating large matrix profiles can become a bottleneck.

* **Model Integration:**
  Separate modules for pattern matching and forecasting can be complex to maintain and tune.

* **Catastrophic Forgetting & Drift:**
  LSTM models may forget older patterns; both approaches need updates for sensor behavior changes.

---

## 4. Transformer Models & TimeGPT

* **Transformers excel at:**
  Capturing **long-range dependencies** in time series with self-attention, handling entire sequences in parallel efficiently.

* **Why not generic LLMs like LLaMA?**
  LLaMA is designed for natural language, not time series. It lacks time-aware training and specialized forecasting capabilities.

* **TimeGPT:**
  A transformer model **specifically trained on billions of real-world time series** for forecasting and anomaly detection. It unifies forecasting and anomaly detection with uncertainty estimates and supports multivariate data.

* **Advantages:**

  * Handles longer sequences and cross-sensor dependencies better than LSTMs.
  * Provides probabilistic forecasts with confidence intervals for anomaly detection.
  * More scalable and maintainable as a unified model.

---

## 5. TimeGPT vs LLaMA Quick Comparison

| Feature               | TimeGPT                                     | LLaMA                               |
| --------------------- | ------------------------------------------- | ----------------------------------- |
| Primary Use Case      | Time series forecasting & anomaly detection | General natural language processing |
| Architecture          | Encoder-decoder transformer                 | Decoder-only transformer            |
| Training Data         | 2M+ time series from various domains        | Large text corpora                  |
| Domain Specialization | High (time series)                          | General-purpose (text)              |
| Availability          | Closed model, open SDK via Nixtla API       | Open model (weights/code)           |

---

## 6. TimeGPT Availability

* The **model itself is closed source**, accessed via Nixtla’s API or platforms like Azure Studio.
* SDKs are **open source** (Apache 2.0 licensed) for easy integration into your monitoring system.

---
