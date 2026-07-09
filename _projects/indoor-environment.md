---
title: Indoor Environment AI
eyebrow: IoT · Time-Series ML · Full-Stack
subtitle: >-
  A self-built indoor-climate automation system — ESP32/Pico W sensors and
  Shelly smart plugs feed InfluxDB, and an hourly scheduler uses an ML model
  to decide whether a dehumidifier should run.
summary: >-
  Pimoroni Enviro + Shelly sensors stream into InfluxDB via Telegraf; a
  FastAPI backend runs an hourly scheduler that pulls readings, runs a
  scikit-learn humidity regressor (with SHAP explanations) and threshold-
  decides whether to switch a dehumidifier on or off — all managed from a
  React dashboard.
order: 2
cover: /assets/projects/indoor-environment/humidity-chart.jpg
tags: [IoT, InfluxDB, Telegraf, FastAPI, scikit-learn, SHAP, React, Time-Series]
stack: [MicroPython, Pimoroni Enviro, Shelly, Telegraf, InfluxDB, FastAPI, scikit-learn, SHAP, React]
---

> Sensors in the rooms, a time-series database on the shelf, and a model that
> decides when the dehumidifier should run. **Indoor Environment AI** is a
> complete closed loop: edge devices collect indoor climate data, InfluxDB
> stores it, and an hourly scheduler runs an ML regressor whose output is
> compared against a threshold to switch a Shelly-controlled dehumidifier on
> or off — no cloud, no manual intervention.

## The idea

Stop guessing when to run the dehumidifier.

Indoor humidity swings with the weather, the ventilation (FTX) system and
whether a dehumidifier is running. The goal is to **predict** indoor humidity
from sensor history and outdoor weather, then let a scheduler act on that
prediction every hour — turning the dehumidifier on only when the model says
humidity is heading above a threshold, and off otherwise.

The work splits into three layers:

1. **Collect** — battery-powered edge sensors and smart-plug meters push
   readings into InfluxDB, either directly or through Telegraf.
2. **Decide** — a FastAPI backend runs an hourly scheduler that loads a
   trained scikit-learn model, builds lag/periodic features from recent
   sensor history and compares the model's output to a per-hour threshold.
3. **Act** — the chosen `on`/`off` action is sent to the Shelly smart plug
   that powers the dehumidifier, and the state is recorded.

## Architecture

The diagram below shows the full data path: how IoT devices collect readings,
how they land in InfluxDB, and how the backend's scheduler, device manager and
ML model service cooperate to control a real appliance — with SQLite holding
the configuration and a React dashboard driving it all.

<div class="diagram">
<div class="mermaid">
flowchart TD
  subgraph edge["IoT edge devices"]
    ENV["Pimoroni Enviro<br/>Pico W · MicroPython<br/>temp · humidity · gas resistance"]
    SHELLY["Shelly Plug S<br/>relay + power meter<br/>powers the dehumidifier"]
  end

  INFLUX[("InfluxDB<br/>bucket 'home'<br/>time-series store")]
  TG["Telegraf<br/>inputs.http · json_v2"]

  ENV -->|"direct write · every 10 min"| INFLUX
  SHELLY -->|"/status/0 JSON · 30 min"| TG
  TG --> INFLUX

  subgraph be["FastAPI backend"]
    direction TB
    DM["DeviceManager<br/>SwitchDevice · SensorDevice"]
    SCH["DeviceScheduler<br/>hourly asyncio loop"]
    MLS["MLModelService<br/>pickle / ONNX · cached"]
    IP["InputProcessor<br/>lags + sin/cos features"]
  end

  DM -->|"Flux queries"| INFLUX
  SCH --> DM
  SCH --> MLS
  MLS -->|"build features"| IP
  DM -->|"device readings"| IP

  SCH -->|"threshold → on / off"| SHELLY

  SQLITE[("SQLite<br/>devices · schedules<br/>ml_models · state")]
  be <--> SQLITE

  subgraph train["Offline training"]
    SMHI["SMHI API<br/>outdoor weather"]
    NB["Jupyter notebook<br/>HistGradientBoosting + SHAP"]
  end
  INFLUX -->|"CSV backups"| NB
  SMHI --> NB
  NB -->|"joblib dump"| PKL["models/*.pkl"]
  PKL --> MLS

  subgraph fe["React dashboard"]
    UI["Devices · Schedules<br/>ML Models · Status"]
  end
  UI <--> be
</div>
</div>

## The time series this runs on

The chart below is real indoor humidity collected by the Enviro board over
months — the exact signal the model learns to predict. The dehumidifier's
on/off state (the Shelly `ison` reading) is what the scheduler is trying to
optimize.

<figure class="cover-figure">
  <img src="{{ '/assets/projects/indoor-environment/humidity-chart.jpg' | relative_url }}" alt="Indoor humidity time series collected from the Enviro sensor, showing seasonal swings the model predicts" loading="lazy" />
  <figcaption>Indoor humidity over time — the prediction target, measured by the Pimoroni Enviro board and stored in InfluxDB.</figcaption>
</figure>

## Architecture by component

### 1. Edge devices (hardware)

Two families of microcontrollers live in the rooms, plus a smart plug that
both meters and switches the dehumidifier.

- **Pimoroni Enviro (Indoor)** — a Raspberry Pi Pico W running MicroPython.
  Wakes every 10 minutes, takes temperature, humidity and gas-resistance
  readings, and writes them straight to InfluxDB via the firmware's
  `influxdb` upload destination. It has its own captive-portal provisioning
  flow, NTP clock sync, cached-upload retry on WiFi failure and a
  USB-power temperature offset compensation.
- **ESP32 humidity gauge** — a TFT display that renders an analog %RH dial
  by querying the humidity reading over HTTP, with deep-sleep wake on a
  button press. An early "always-visible" view of the same data.
- **Shelly Plug S** — a WiFi smart relay that powers the dehumidifier. It
  exposes a tiny HTTP API (`/relay/0?turn=on|off`, `/status/0`) that both
  Telegraf (for metering) and the backend (for control) call.

### 2. Collection & storage (env + data)

- **Telegraf** polls the Shelly's `/status/0` JSON endpoint every 30 minutes
  with an `inputs.http` + `json_v2` parser, extracting power, relay `ison`,
  temperature, WiFi RSSI and firmware tags, then writes them to InfluxDB v2
  (`outputs.influxdb_v2`, bucket `home`).
- **InfluxDB** is the single time-series store. The Enviro writes its
  readings directly; Telegraf writes the Shelly metrics. Everything is
  queryable with Flux — `range`, `aggregateWindow(every: 1h, fn: mean)` and
  `pivot` to turn fields into columns.
- **Backups** — `influx_backup.sh` exports the bucket to CSV snapshots
  (`data/influxdb_backup-*`), which double as the offline training set for
  the notebooks.
- **SMHI** — Sweden's open weather API (`notebooks/SMHI.py`) pulls outdoor
  temperature and humidity for the nearest station, joined into the training
  data so the model can reason about outdoor conditions.

### 3. Backend (control/backend)

A FastAPI service that owns the device model, the scheduler and the ML
runtime. Auth is an OAuth2 password flow with JWT; all `/api/v1/*` routes are
protected.

- **DeviceManager** — an abstract `Device` base with two implementations:
  `SwitchDevice` (the Shelly plug — `on`/`off` actions via httpx, plus power,
  temperature and `ison` queries) and `SensorDevice` (the Enviro — read-only,
  `noop` action, exposes temperature/humidity/gas-resistance queries). Each
  measurement method delegates to a Flux query in `database/influxdb.py`, so
  the device classes are the single seam between the API and the time-series
  store.
- **DeviceScheduler** — an `asyncio` loop that fires at the top of every
  hour. For each schedule it reads the 24-element `hourly_schedule`, takes
  the current hour's action, and — if ML threshold mode is enabled for that
  hour — runs the model, compares its output to `ml_threshold_value`, and
  picks the `ml_threshold_above_action` or `ml_threshold_below_action`. The
  winning action is executed on the device and the resulting on/off state is
  recorded. A `run_now()` lets the UI trigger an immediate evaluation after a
  schedule edit.
- **MLModelService** — loads ONNX (`onnxruntime`) or pickle (`joblib`) models
  from disk, caches them by id, and runs inference. For regressors it returns
  the raw predicted value (so the threshold comparison is meaningful); for
  classifiers it returns a boolean. It also computes **SHAP** explanations —
  `TreeExplainer` for `HistGradientBoostingRegressor`, falling back to
  `KernelExplainer` — so each decision is inspectable.
- **Input processors** — a pluggable `InputProcessor` base
  (`app/ml_processors/`) turns raw device readings into model features. The
  built-in `humidity_model` processor resamples Shelly `ison` and Enviro
  humidity to hourly, builds lag features (humidity lags + relay-state lags)
  and cyclical sin/cos encodings of month and hour — exactly the feature
  recipe the notebook trains on. Custom processors are auto-discovered.
- **Persistence** — SQLite holds the configuration (devices, schedules,
  `ml_models`, device-state history); InfluxDB holds the measurements. The
  split keeps config fast and transactional while time-series stays in a
  purpose-built store.

### 4. Frontend (control/frontend)

A React (Create React App) SPA behind a login screen.

- **Devices** — list/detail with live sensor charts (temperature, humidity,
  gas resistance, power, `ison`) served by the `/sensor-data` endpoints.
- **Schedules** — a 24-hour grid editor where each hour gets an action, plus
  per-hour ML-threshold configuration (enabled hours, threshold value, and
  the above/below actions).
- **ML Models** — register a model (type, file path, input processor, input
  devices), and a **Test** button runs it against current device data and
  returns the output with the SHAP explanation.
- **Status** — a live system log view pulled from the backend's log buffer.

### 5. The model (notebooks + models)

`ftx-auto-model.ipynb` is where the humidity predictor is trained and
exported:

- Loads the InfluxDB backup CSVs (Enviro + Shelly) and SMHI weather, resampled
  to hourly with `ison` taken as the hourly max.
- Cleans the signal — FFT deseasonalization + an `IsolationForest` to flag
  and drop temperature outliers, then interpolation.
- Engineers features: lags of humidity and relay state, plus sin/cos
  encodings of month and hour.
- Trains a two-stage stack: a **weather model**
  (`HistGradientBoostingRegressor`) from SMHI + indoor sensors, then an **FTX
  model** that corrects the weather model's residual using the dehumidifier's
  on/off state — so it learns *the effect of running the dehumidifier*.
- Evaluates with R²/MAPE on a time-series split, inspects with SHAP beeswarms,
  and exports the final estimator with `joblib` to
  `models/humidity_prediction_model_*.pkl`, which the backend's
  `MLModelService` loads at runtime.

## Technologies used

<div class="tech-grid">
  <div class="tech">
    <h4>Edge hardware</h4>
    <ul>
      <li>Pimoroni Enviro (Pico W, MicroPython)</li>
      <li>ESP32 + TFT_eSPI (humidity gauge)</li>
      <li>Shelly Plug S (relay + metering)</li>
      <li>Direct InfluxDB writes from firmware</li>
      <li>Captive-portal provisioning, NTP sync</li>
    </ul>
  </div>
  <div class="tech">
    <h4>Collection &amp; storage</h4>
    <ul>
      <li>Telegraf (inputs.http, json_v2)</li>
      <li>InfluxDB 2.x (bucket, Flux)</li>
      <li>SMHI open-data API (weather)</li>
      <li>influx_backup.sh → CSV snapshots</li>
    </ul>
  </div>
  <div class="tech">
    <h4>Backend</h4>
    <ul>
      <li>FastAPI + Uvicorn</li>
      <li>asyncio hourly scheduler</li>
      <li>SQLite (devices, schedules, ml_models)</li>
      <li>httpx (Shelly control + metering)</li>
      <li>OAuth2 / JWT auth</li>
      <li>uv (package management)</li>
    </ul>
  </div>
  <div class="tech">
    <h4>ML &amp; decisioning</h4>
    <ul>
      <li>scikit-learn (HistGradientBoostingRegressor)</li>
      <li>SHAP (TreeExplainer / KernelExplainer)</li>
      <li>ONNX Runtime + joblib</li>
      <li>Pluggable InputProcessor feature pipeline</li>
      <li>Threshold-based on/off decisions</li>
      <li>Jupyter / pandas / plotly (training)</li>
    </ul>
  </div>
  <div class="tech">
    <h4>Frontend</h4>
    <ul>
      <li>React (Create React App)</li>
      <li>React Router</li>
      <li>Devices, Schedules, ML Models, Status</li>
      <li>Sensor time-series charts</li>
      <li>24-hour schedule grid editor</li>
    </ul>
  </div>
  <div class="tech">
    <h4>Operations</h4>
    <ul>
      <li>Optional HTTPS (self-signed / Let's Encrypt)</li>
      <li>Configurable CORS origins</li>
      <li>File + buffer log streaming</li>
      <li>start.py / start.sh / start.bat launchers</li>
    </ul>
  </div>
</div>

<div class="callout">
  <p>
    <strong>Status:</strong> the full loop is live — Enviro sensors and Shelly
    plugs are ingesting into InfluxDB, the hourly scheduler runs a trained
    humidity regressor, and threshold decisions switch the dehumidifier
    automatically. The React dashboard manages devices, schedules and models,
    and SHAP explanations make every decision inspectable. Active work is on
    refining the feature pipeline and adding more ML deciders per schedule.
  </p>
</div>
