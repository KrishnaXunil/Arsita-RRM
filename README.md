# RRM+ --- AI-Assisted, Client-Aware, Closed-Loop Radio Resource Management

**Arista Networks · Team 15**

Entire Submission can be accessed vua this drive link:
https://drive.google.com/file/d/1dGI2lKqFzNKRnciS8V1jckWEFdTMiIYG/view?usp=sharing

RRM+ is a closed-loop Radio Resource Management (RRM) architecture that
combines spectrum sensing, non-Wi-Fi interference classification,
statistical change detection, client-aware telemetry, safe optimization,
and automated configuration control.

The system is organized around five primary API surfaces:

-   `/sensing`
-   `/client_view`
-   `/planner/propose`
-   `/planner/commit`
-   `/metrics`

The design combines fast local control on APs with slower global
planning, while applying safety constraints, cooldowns, rollback
conditions, and auditability.

------------------------------------------------------------------------

## Table of Contents

1.  [Overview](#overview)
2.  [Problem Statement](#problem-statement)
3.  [System Architecture](#system-architecture)
4.  [Core Components](#core-components)
    -   [Spectrum Sensing](#1-spectrum-sensing)
    -   [Non-Wi-Fi Classifier](#2-non-wi-fi-classifier)
    -   [Change Detection](#3-change-detection)
    -   [Client-Aware Telemetry](#4-client-aware-telemetry)
    -   [Safe RL Planner](#5-safe-rl-planner)
    -   [Fast-Loop Controller](#6-fast-loop-controller)
5.  [End-to-End Workflow](#end-to-end-workflow)
6.  [Optimization Strategy](#optimization-strategy)
7.  [Safety and Guardrails](#safety-and-guardrails)
8.  [API Surface](#api-surface)
9.  [Metrics and Evaluation](#metrics-and-evaluation)
10. [Data Retention and Privacy](#data-retention-and-privacy)
11. [Results](#results)
12. [Repository Structure](#repository-structure)
13. [Running the System](#running-the-system)
14. [Configuration](#configuration)
15. [Limitations and Future Work](#limitations-and-future-work)

------------------------------------------------------------------------

# Overview

Traditional RRM systems primarily react to Wi-Fi telemetry such as
channel utilization, RSSI, retries, and neighboring APs. RRM+ extends
this approach by incorporating:

-   Dedicated-radio spectrum sensing.
-   Adaptive channel dwell-time scheduling.
-   Non-Wi-Fi interference classification.
-   Online change detection.
-   802.11k/v client telemetry.
-   Client-level QoE estimation.
-   Interference-graph-based optimization.
-   Safe reinforcement-learning proposals.
-   Explicit configuration guardrails.
-   Automatic rollback.
-   Multi-timescale control loops.
-   Persistent metrics and signed audit records.

The overall objective is to make RRM **closed-loop and client-aware**:

``` text
             ┌─────────────────────────┐
             │     Spectrum Sensing    │
             │                         │
             │ Scheduler + Classifier  │
             │ + Change Detection      │
             └────────────┬────────────┘
                          │
                          ▼
             ┌─────────────────────────┐
             │     Client View         │
             │                         │
             │ 802.11k/v + QoE        │
             └────────────┬────────────┘
                          │
                          ▼
             ┌─────────────────────────┐
             │    Planner / RRM+       │
             │                         │
             │ Safe RL + Interference │
             │ Graph + Policy Rules   │
             └────────────┬────────────┘
                          │
                          ▼
             ┌─────────────────────────┐
             │   Configuration Engine  │
             │                         │
             │ Channel / Width / Power │
             │ OBSS-PD / Rollback      │
             └────────────┬────────────┘
                          │
                          ▼
                  New Network State
                          │
                          └───────────────► feedback
```

------------------------------------------------------------------------

# Problem Statement

Wireless networks are affected by several types of interference and
rapidly changing client conditions:

1.  Wi-Fi contention between neighboring APs.
2.  Non-Wi-Fi interference such as BLE, Zigbee, FHSS and microwave
    sources.
3.  Hidden interference that is not visible through conventional
    AP-level measurements.
4.  Client-specific degradation despite acceptable aggregate AP metrics.
5.  Configuration changes that may improve one AP while degrading
    neighboring APs.
6.  Instability caused by repeatedly changing channels, bandwidth, or
    transmit power.

RRM+ addresses these problems through a feedback loop:

``` text
Sense → Understand → Measure Client Impact → Plan → Validate → Execute → Measure Again
```

------------------------------------------------------------------------

# System Architecture

The architecture is divided into three major execution layers:

### 1. On-AP Controller

The local controller handles fast decisions and safety-sensitive
operations.

It consumes:

-   Additional-radio RSSI/FFT information.
-   Non-Wi-Fi classifications.
-   CCA busy percentage.
-   Airtime.
-   Noise floor.
-   Client telemetry.
-   Retry rates.

The fast controller runs a scheduled interference-aware optimization
cycle.

### 2. Cloud Controller

The cloud layer maintains aggregated network information and supports
longer-timescale optimization.

### 3. Site Planner

The site-level planner reasons across APs and the broader interference
graph.

The presentation describes this hierarchy as:

``` text
RRM+
├── On-AP Controller
├── Cloud Controller
└── Site-Planner
```

This separation allows local, fast reactions without requiring every
decision to wait for centralized planning.

------------------------------------------------------------------------

# Core Components

## 1. Spectrum Sensing

### API

``` text
/sensing
```

The sensing subsystem contains three major components:

``` text
Channel Scan Scheduler
        │
        ├── Non-WiFi Classifier
        │
        └── Change Detection
```

------------------------------------------------------------------------

## 1.1 Adaptive Channel Scan Scheduler

The scheduler allocates a strict sensing-time budget across channels.

Instead of assigning identical dwell time to every channel, RRM+ uses
historical information and online observations to concentrate sensing
effort where it provides more information.

### Hybrid Offline + Online Learning

The proposed pipeline contains two phases.

### Offline phase

Historical measurements contain features such as:

-   Variance
-   Mean energy
-   SNR
-   Noise floor
-   Occupied fraction
-   Other spectrum-derived features

These observations are transformed into state/reward representations and
clustered using K-Means.

The offline stage learns:

-   State clusters.
-   Expected reward for clusters.
-   Historical dwell-time/action mappings.

This produces an offline prior that can be used when the online system
encounters a similar state.

### Online phase

At the beginning of each epoch:

1.  A strict dwell-time budget is established.
2.  Fresh spectrum sensing is performed.
3.  A new signal fingerprint is generated.
4.  The current state is matched against offline clusters.
5.  The offline utility is used as a prior.
6.  Online observations update the model.
7.  Candidate dwell-time actions are scored.
8.  A knapsack-based optimizer selects actions within the available
    budget.
9.  The selected dwell times are executed.
10. The resulting observations are fed back into the online model.

Conceptually:

``` text
Historical Data
      │
      ▼
Feature Extraction
      │
      ▼
K-Means State Clustering
      │
      ├──────────────► Expected Cluster Rewards
      │
      └──────────────► Historical Actions
                         │
                         ▼
                  Offline Prior
                         │
                         ▼
Fresh Spectrum ──► State Fingerprint
                         │
                         ▼
                  Cluster Matching
                         │
                         ▼
             Offline + Online Scoring
                         │
                         ▼
               Knapsack Optimization
                         │
                         ▼
                  Dwell Allocation
                         │
                         ▼
                    New Data
                         │
                         └──────► Online Update
```

### Why combine offline and online learning?

A purely online method must learn from scratch and may spend significant
sensing budget exploring poor actions.

The offline model supplies an initial prior, while the online component
adapts to current conditions.

This provides a practical compromise between:

-   Exploration.
-   Adaptation.
-   Computation.
-   Sensing-time constraints.

------------------------------------------------------------------------

## 1.2 Non-Wi-Fi Classifier

The non-Wi-Fi classifier identifies interference sources that can affect
Wi-Fi operation.

Target classes include:

-   BLE
-   FHSS
-   Microwave
-   Zigbee

### Processing pipeline

``` text
Spectrum Source
      │
      ▼
Spectral Acquisition
      │
      ▼
Event Detection / Segmentation
      │
      ▼
Feature Extraction
      │
      ▼
Decision Tree
      │
      ▼
Class + Confidence
```

The feature-extraction stage includes measurements such as:

-   Event duration.
-   Center frequency.
-   Bandwidth.
-   Average RSSI.
-   RSSI range.
-   Spectral spread.
-   Kurtosis.
-   Entropy.

The implementation uses a pruned Decision Tree.

### Why a Decision Tree?

The design emphasizes:

-   Low inference cost.
-   Small memory footprint.
-   Interpretability.
-   Real-time execution.

Cost-complexity pruning is used to reduce overfitting by penalizing
unnecessary tree complexity.

### Reported validation performance

The presentation reports:

-   Overall accuracy: **95.8%**
-   Macro precision: **0.958**
-   Macro recall: **0.963**
-   Macro F1: **0.961**

Per-class F1 values reported in the presentation:

  Class         Precision   Recall      F1
  ----------- ----------- -------- -------
  Microwave         0.989    0.994   0.991
  FHSS              0.944    0.983   0.963
  Zigbee            0.950    0.950   0.950
  BLE               0.951    0.926   0.938

The presentation also reports a small memory footprint for the trained
tree and emphasizes that the classifier is intended to run without
consuming airtime that should be reserved for clients.

------------------------------------------------------------------------

# 3. Change Detection

### API

``` text
/sensing
```

The change-detection system monitors network and radio telemetry for
persistent or sudden changes.

### Input metrics

The system uses:

-   CCA Busy %
-   Airtime
-   Noise Floor
-   SNR

These metrics represent different aspects of network health:

  Metric        What it indicates
  ------------- ---------------------------------------
  CCA Busy %    Channel utilization/contention
  Airtime       Amount of channel time being consumed
  Noise Floor   External/interference power
  SNR           Link quality

------------------------------------------------------------------------

## Detection algorithms

### EWMA

Exponentially Weighted Moving Average smooths high-frequency
fluctuations.

Used primarily for:

-   Sustained congestion.

### CUSUM

Cumulative Sum tracks accumulated deviations from a baseline.

Used for:

-   Sudden/persistent SNR changes.

### SPRT

Sequential Probability Ratio Testing evaluates the likelihood of
different operating states.

Used for:

-   Rapid noise-floor/interference confirmation.

### Page-Hinkley

The Page-Hinkley test tracks cumulative deviation and is useful for
detecting gradual drift.

Used across:

-   CCA.
-   Airtime.
-   SNR.
-   Noise floor.

------------------------------------------------------------------------

## False-Alarm Reduction

Individual detectors are combined using a multi-stage fusion engine.

### 2-of-4 voting

At least two detector signals must trigger within a three-frame window.

### Debounce

The condition must persist for two consecutive frames.

### Event stitching

Rapid repeated detections are merged into one continuous event.

### Cooldown

A roughly 30-second suppression period prevents repeated alerts for the
same condition.

Overall:

``` text
Raw Telemetry
     │
     ├── EWMA
     ├── CUSUM
     ├── SPRT
     └── Page-Hinkley
            │
            ▼
       2-of-4 Voting
            │
            ▼
       Debounce Filter
            │
            ▼
       Event Stitching
            │
            ▼
          Cooldown
            │
            ▼
       Validated Alert
```

The presentation reports a fusion detection rate of approximately
**95%** and a false-positive rate of approximately **3%** in its
evaluation.

------------------------------------------------------------------------

# 4. Client-Aware Telemetry

### API

``` text
/client_view
```

RRM+ does not optimize solely from AP-level radio measurements.

It also collects client-side information using IEEE 802.11k/v
mechanisms.

------------------------------------------------------------------------

## 4.1 Link Measurement Scheduler

The scheduler:

1.  Sends standardized link-measurement requests.
2.  Collects RSSI.
3.  Collects link margin.
4.  Collects client transmit power.
5.  Collects antenna configuration.
6.  Handles stale reports.
7.  Parses asynchronous responses.
8.  Updates the database.

------------------------------------------------------------------------

## 4.2 Beacon Measurement Scheduler

Beacon measurement requests contain parameters such as:

-   Operating class.
-   Channel.
-   Measurement duration.

The system extracts:

-   RCPI.
-   RSNI.
-   Channel utilization.
-   Station count.
-   AP capabilities.

Multiple measurement cycles are aggregated to reduce noise.

------------------------------------------------------------------------

# Client QoE

The client QoE model uses five factors:

### Signal Quality

RSSI is mapped linearly from approximately:

``` text
-90 dBm → 0
-30 dBm → 1
```

### Throughput

The geometric mean of TX/RX bitrates is normalized to PHY rate.

### Reliability

Combines:

-   Retry rate.
-   FCS errors.

### Latency

Inactivity time is mapped into a responsiveness score.

### Activity

Frame count is normalized against a reference load.

The resulting QoE score is updated every measurement cycle and stored
with timestamped samples.

------------------------------------------------------------------------

# 5. Safe RL Planner

### API

``` text
/planner/propose
```

The planner uses a bounded action space and explicit safety constraints.

The system tracks 15 standardized network-state metrics. Important
metrics include:

-   Client count.
-   Median RSSI.
-   P95 retry rate.
-   P95 PER.
-   Channel utilization.
-   Average throughput.
-   Edge P10 throughput.
-   Roaming rate.
-   Neighbor AP RSSI.
-   OBSS-PD.
-   TX power.
-   Noise floor.
-   Channel width.
-   Airtime usage.
-   CCA busy.

------------------------------------------------------------------------

## Action Space

The planner can propose controlled changes such as:

-   TX power ±2 dBm.
-   Channel increase/decrease.
-   Channel-width changes: 20 ↔ 40 ↔ 80 MHz.
-   OBSS-PD adjustment.
-   No-op.

Channel changes are constrained by DFS compliance.

------------------------------------------------------------------------

## Reward Function

The objective is not simply maximum throughput.

The presentation defines a weighted reward containing positive
performance terms and stability penalties.

### Positive terms

-   Edge throughput: +35%
-   Mean throughput: +15%
-   RSSI quality: +10%
-   Stability: +15%

### Penalties

-   Retry rate: −25%
-   Packet error rate: −15%
-   Configuration churn: −10%

This encourages improvements that are useful to clients while
discouraging unstable configuration changes.

------------------------------------------------------------------------

# Three-Tier Safety Model

Safe RL is surrounded by three layers of constraints.

## Tier 1 --- Hard Boundaries

Absolute regulatory and configuration constraints.

A proposed action that violates these constraints is immediately
rejected.

## Tier 2 --- Protective Shield

A safety envelope prevents the agent from proposing configurations that
are technically possible but unsafe or operationally undesirable.

## Tier 3 --- Behavioral Constraints

Soft constraints are incorporated into the reward function.

The agent is encouraged to avoid:

-   Excessive configuration churn.
-   Unnecessary changes.
-   Instability.
-   Poor client outcomes.

------------------------------------------------------------------------

# 6. Fast-Loop Controller

The fast loop is an interference-aware optimizer running on a scheduled
cycle.

The presentation specifies a **10-minute** cycle.

Its purpose is to operate between:

-   Event-level reactions occurring over seconds.
-   Long-timescale planning occurring over days.

The design principles are:

-   Do no harm.
-   Make small changes.
-   Prefer reversible actions.
-   Use cooldowns.
-   Avoid configuration thrashing.

------------------------------------------------------------------------

## Interference Graph

Every cycle builds a directed interference graph from:

-   Additional-radio RSSI.
-   FFT/spectrum information.
-   Non-Wi-Fi signatures.
-   802.11k/v client telemetry.
-   Client SINR impact.

Each AP receives an interference score based on weighted neighboring
interference.

------------------------------------------------------------------------

## MAPE Loop

The controller follows:

``` text
Monitor
   ↓
Analyze
   ↓
Plan
   ↓
Execute
   ↓
Measure
   ↺
```

### Monitor

Ingest:

-   Sensing data.
-   Retry rates.
-   CCA busy.
-   Channel information.

### Analyze

Calculate:

-   Total interference.
-   Per-channel interference.
-   Client impact.

### Plan

Select a corrective action using priority logic.

### Execute

Apply the configuration through the configuration engine.

Each change is logged with an audit record and rollback token.

------------------------------------------------------------------------

# End-to-End Workflow

The complete RRM+ loop can be summarized as:

``` text
                 ┌─────────────────────┐
                 │ Spectrum + Telemetry│
                 └──────────┬──────────┘
                            │
                            ▼
                ┌────────────────────────┐
                │ Sensing                │
                │                        │
                │ • Dwell Scheduler      │
                │ • Non-WiFi Classifier  │
                │ • Change Detection     │
                └───────────┬────────────┘
                            │
                            ▼
                ┌────────────────────────┐
                │ Client View             │
                │                        │
                │ • 802.11k/v            │
                │ • QoE                   │
                │ • Link Measurements    │
                └───────────┬────────────┘
                            │
                            ▼
                ┌────────────────────────┐
                │ State Construction      │
                │                        │
                │ Network + Client +     │
                │ Interference Features  │
                └───────────┬────────────┘
                            │
                            ▼
                ┌────────────────────────┐
                │ Planner                 │
                │                        │
                │ Safe RL + Graph +      │
                │ Policy Constraints     │
                └───────────┬────────────┘
                            │
                            ▼
                ┌────────────────────────┐
                │ Safety Validation       │
                └───────────┬────────────┘
                            │
                            ▼
                ┌────────────────────────┐
                │ Configuration Commit    │
                └───────────┬────────────┘
                            │
                            ▼
                ┌────────────────────────┐
                │ Metrics + Audit         │
                └───────────┬────────────┘
                            │
                            └──────► Feedback
```

------------------------------------------------------------------------

# Optimization Strategy

RRM+ uses multiple timescales rather than forcing every decision through
one optimizer.

  Loop         Timescale      Purpose
  ------------ -------------- -----------------------------------------
  Event loop   Seconds        Detect and react to sudden interference
  Fast loop    \~10 minutes   Incremental AP-level optimization
  Slow loop    Hours/days     Global network/site planning

### Event loop

Examples:

-   Microwave interference burst.
-   BLE/Zigbee activity.
-   Sudden SNR degradation.
-   Abrupt channel loading.

### Fast loop

Typical actions:

-   Bandwidth adjustment.
-   OBSS-PD tuning.
-   Channel changes when significant.
-   Interference-aware local corrections.

### Slow loop

Suitable for:

-   Global channel planning.
-   Network-wide interference optimization.
-   Longer-term policy/model updates.

------------------------------------------------------------------------

# Safety and Guardrails

The presentation defines explicit operational guardrails.

### Regulatory safety

-   Prefer safe non-DFS channels by default.
-   Strictly cap 2.4 GHz bandwidth at 20 MHz.

### OBSS-PD

The system restricts OBSS-PD thresholds to:

``` text
-82 dBm to -62 dBm
```

### Stability

A per-AP cooldown of approximately **10 minutes** prevents rapid
repeated changes.

### Automatic rollback

The system can roll back a configuration if:

``` text
Retry rate increases by >10%
OR
Throughput decreases by >20%
```

This makes configuration changes reversible rather than permanently
committing potentially harmful decisions.

------------------------------------------------------------------------

# API Surface

RRM+ exposes five conceptual API areas.

## `/sensing`

Provides:

-   Channel sensing.
-   Spectrum fingerprints.
-   Non-Wi-Fi classification.
-   Change-detection events.
-   Channel/dwell information.

## `/client_view`

Provides:

-   Client telemetry.
-   802.11k measurements.
-   Beacon reports.
-   QoE measurements.
-   Client/network relationships.

## `/planner/propose`

Generates a candidate RRM action.

Example conceptual response:

``` json
{
  "ap_id": "AP-01",
  "proposed_action": {
    "channel_width": 40,
    "tx_power_delta_db": -2,
    "obss_pd_delta_db": 3
  },
  "reason": "high_interference",
  "safety_status": "validated"
}
```

> The exact production API schema should be aligned with the
> implementation. The example above documents the architecture rather
> than claiming to be the exact wire format.

## `/planner/commit`

Commits a previously validated configuration proposal.

The commit path is expected to maintain:

-   Audit information.
-   Rollback information.
-   Configuration history.

## `/metrics`

Provides:

-   Network metrics.
-   QoE metrics.
-   Planner outcomes.
-   Historical aggregates.
-   Audit-related information.

------------------------------------------------------------------------

# Metrics and Evaluation

The system evaluates multiple dimensions rather than optimizing a single
metric.

Important metrics include:

-   Throughput.
-   Edge throughput.
-   Retry rate.
-   Packet error rate.
-   RSSI.
-   QoE.
-   Channel utilization.
-   CCA busy.
-   Airtime.
-   Roaming.
-   Configuration churn.
-   Action success rate.
-   Detection rate.
-   False-positive rate.

------------------------------------------------------------------------

# Results

## Spectrum Scheduler

The presentation compares the proposed scheduling approach against:

-   UCB.
-   Equal round robin.
-   Random allocation.
-   Optimal allocation.

The displayed evaluation shows the proposed UCB-based scheduler
outperforming equal and random allocation in cumulative reward, while an
optimal reference remains above it.

The exact numerical values should be interpreted in the context of the
simulation used for the evaluation.

------------------------------------------------------------------------

## Non-Wi-Fi Classification

Reported validation results:

``` text
Accuracy       : 95.8%
Macro Precision: 95.8%
Macro Recall   : 96.3%
Macro F1       : 96.1%
```

------------------------------------------------------------------------

## Change Detection

The fusion engine evaluation reports approximately:

``` text
Detection rate    : 95%
False-positive rate: 3%
```

The fusion mechanism is intended to reduce false alarms compared with
individual detectors.

------------------------------------------------------------------------

## Fast-Loop Controller

Across **4,320 cycles**, the presentation reports:

``` text
Total changes: 5,904

Bandwidth adjustments : 58%
OBSS-PD tuning        : 26%
Channel changes       : 15%
```

The reported positive-action rate is:

``` text
94.3%
```

The remaining action distribution is not explicitly detailed in the
presentation.

------------------------------------------------------------------------

## Pilot Site Simulation

The presentation includes three-day simulation-log comparisons between
configurations with and without RRM+.

One reported pilot result shows:

``` text
P95 retry-rate improvement : 18.2%
Average throughput         : 10.04%
P95 QoE                    : 7.9%
```

Another reported simulation result shows:

``` text
P95 retry-rate improvement : 18.3%
Average throughput         : 17.22%
P95 QoE                    : 10.9%
```

These are presentation-reported simulation results rather than
independently reproduced benchmarks.

------------------------------------------------------------------------

# Data Retention and Privacy

The metrics subsystem uses category-specific retention periods.

The design principles include:

-   Raw telemetry is retained only for the period needed to calculate
    operational metrics.
-   Aggregated percentiles are retained longer for trend analysis and
    model stability.
-   Audit logs are retained longer for accountability.
-   Pseudonymization keys are rotated periodically.
-   Retired keys are invalidated.
-   Automated purge jobs remove expired data.
-   Purge operations are themselves recorded and signed.
-   Devices that opt out are excluded from per-device storage and are
    represented only through coarse aggregate statistics.

------------------------------------------------------------------------

# Repository Structure

A recommended implementation layout is:

``` text
rrm-plus/
│
├── README.md
│
├── sensing/
│   ├── scheduler/
│   ├── classifier/
│   ├── change_detection/
│   └── feature_extraction/
│
├── client_view/
│   ├── link_measurements/
│   ├── beacon_measurements/
│   ├── telemetry/
│   └── qoe/
│
├── planner/
│   ├── safe_rl/
│   ├── interference_graph/
│   ├── policy/
│   ├── optimizer/
│   └── safety/
│
├── controller/
│   ├── fast_loop/
│   ├── event_loop/
│   └── configuration_engine/
│
├── metrics/
│   ├── aggregation/
│   ├── retention/
│   └── audit/
│
├── data/
│   ├── raw/
│   ├── processed/
│   └── models/
│
├── experiments/
│   ├── scheduler/
│   ├── classifier/
│   ├── change_detection/
│   └── pilot/
│
└── tests/
    ├── unit/
    ├── integration/
    └── simulation/
```

This structure is a documentation recommendation; the presentation
itself does not specify a complete source-code directory tree.

------------------------------------------------------------------------

# Running the System

Because the presentation describes the architecture and evaluation
rather than a complete executable repository, the exact build/run
commands are implementation-dependent.

A typical development workflow is:

``` bash
# Clone the repository
git clone <repository-url>
cd rrm-plus

# Create environment
python -m venv .venv

# Activate on Linux/macOS
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run unit tests
pytest

# Start the API service
python <entrypoint>.py
```

Replace `<repository-url>` and `<entrypoint>.py` with the actual project
values.

------------------------------------------------------------------------

# Configuration

Important configuration categories include:

### Sensing

``` text
sensing_budget
channel_list
minimum_dwell_time
maximum_dwell_time
scheduler_policy
```

### Classifier

``` text
classifier_model
confidence_threshold
event_duration_threshold
feature_configuration
```

### Change Detection

``` text
ewma_parameters
cusum_parameters
sprt_parameters
page_hinkley_parameters
voting_window
debounce_frames
cooldown_seconds
```

### Planner

``` text
reward_weights
action_space
safety_limits
change_budget
cooldown
rollback_thresholds
```

### Metrics

``` text
raw_telemetry_retention
aggregate_retention
audit_retention
purge_frequency
pseudonymization_rotation
```

------------------------------------------------------------------------

# Testing Strategy

A production implementation should test the system at multiple levels.

## Unit Tests

Test independently:

-   Feature extraction.
-   Reward calculation.
-   Decision-tree inference.
-   EWMA/CUSUM/SPRT/Page-Hinkley detectors.
-   Voting logic.
-   QoE calculation.
-   Safety constraints.
-   Rollback logic.

## Integration Tests

Test complete flows:

``` text
Telemetry
   ↓
Sensing
   ↓
State
   ↓
Planner
   ↓
Safety
   ↓
Commit
   ↓
Metrics
```

## Simulation Tests

Evaluate:

-   Dense AP deployments.
-   High channel utilization.
-   Non-Wi-Fi interference.
-   Sudden interference bursts.
-   Gradual degradation.
-   Client mobility.
-   Hidden-node conditions.
-   Repeated configuration changes.

------------------------------------------------------------------------

# Limitations and Future Work

The presentation establishes the architecture and simulation/pilot
evaluation, but a complete production deployment would require further
validation.

Potential future work includes:

1.  Larger real-world datasets for non-Wi-Fi classification.
2.  More extensive multi-site validation.
3.  Hardware-in-the-loop evaluation.
4.  More realistic client mobility models.
5.  Continuous model calibration.
6.  Stronger uncertainty estimation for planner proposals.
7.  Distributed coordination between AP controllers.
8.  More detailed SLA-aware optimization.
9.  Automated rollback verification under real traffic.
10. Long-term evaluation of configuration churn and network stability.

------------------------------------------------------------------------

# Design Principles

RRM+ is built around several principles:

### Client-aware

AP-level radio statistics are combined with client-level measurements.

### Interference-aware

The system considers both Wi-Fi and non-Wi-Fi interference.

### Closed-loop

Actions are followed by measurement and feedback.

### Safe by construction

Hard constraints, safety envelopes, cooldowns, and rollback protect the
network from unsafe or unstable actions.

### Multi-timescale

Fast events and slow global optimization are handled by different
control loops.

### Lightweight

Several components, particularly the non-Wi-Fi classifier, are designed
for low computational and memory overhead.

### Auditable

Automated configuration changes are logged and associated with rollback
information.

------------------------------------------------------------------------

# Summary

RRM+ combines sensing, client telemetry, machine learning, optimization,
and policy enforcement into a single closed-loop RRM architecture.

The central pipeline is:

``` text
             SENSE
               │
               ▼
        UNDERSTAND RADIO
               │
               ▼
        MEASURE CLIENTS
               │
               ▼
          BUILD STATE
               │
               ▼
       PROPOSE SAFE ACTION
               │
               ▼
        VALIDATE GUARDRAILS
               │
               ▼
             COMMIT
               │
               ▼
       MEASURE OUTCOME
               │
               └──────────► repeat
```

The architecture combines:

-   Hybrid offline/online spectrum scheduling.
-   Non-Wi-Fi classification.
-   Multi-detector change detection.
-   802.11k/v client telemetry.
-   QoE modeling.
-   Interference-graph reasoning.
-   Safe RL.
-   Fast-loop optimization.
-   Automatic rollback.
-   Metrics and auditability.

Together, these components provide the foundation for a client-aware,
interference-aware, closed-loop RRM system.
