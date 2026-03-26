# Multi-Device User Identification Through Forensic Analysis

A heuristic classification pipeline that identifies which users operate multiple physical devices, using a 5-layer evidence hierarchy built from first principles — no labeled training data, no ML models.

## The Problem

Given 7,915 activity records across 7 users and 20 device identifiers — each with an IP address, device ID, ID type, and operating system — determine which users are operating more than one physical device.

The challenge: multiple device IDs don't mean multiple devices. Cookies get cleared, advertising IDs get reset, apps get reinstalled. A naive count of IDs per user would produce a flood of false positives.

## Approach

The pipeline applies five independent analytical layers, each adding evidence that either upgrades or downgrades a user's multi-device classification:

### Layer 1: Cross-Platform OS Divergence (Decisive)
An Android device cannot emit iOS telemetry. If a user has records on both operating systems, they have multiple devices — proven by the laws of software architecture.

### Layer 2: Device ID Lifespan Analysis
For each user's device IDs, compute the row-index lifespan (first appearance to last appearance) and measure pairwise overlap. **Sequential** lifespans (one ends, another starts) suggest ID resets on a single device. **Parallel** lifespans (both active simultaneously) suggest multiple devices.

### Layer 3: Temporal Concurrency ("Two Hands" Test)
Can a single device produce two requests at the same instant? Near-simultaneous activity across different IDs would prove parallel devices. (Limited by timestamp granularity in this dataset — documented as a methodology constraint.)

### Layer 4: IP Address Jaccard Similarity
Compare the set of IP addresses used by each device ID pair. High Jaccard similarity (shared Wi-Fi networks, same carrier NAT pools) suggests co-located devices or the same device. Low similarity suggests devices in different network environments.

### Layer 5: Volume Threshold (Noise Filter)
Device IDs with fewer than 8 records are filtered as noise — stale cookies, redirect artifacts, or testing sessions that don't represent real device footprints.

## Results

| User | Classification | Confidence | Key Evidence |
|:---:|---|---|---|
| 1 | Single Device | Likely | 3 sequential GAIDs, no overlap |
| 2 | **Multi-Device** | High | 5 overlapping IDFVs with substantial volume |
| 3 | Single Device | Likely | Sequential WEB_IDs, consistent with cookie resets |
| 4 | Single Device | Decisive | Only 1 device ID observed |
| 5 | **Multi-Device** | Moderate | 2 overlapping IDFAs (hardware-bound) |
| 6 | **Multi-Device** | Decisive | iOS + Android activity — cross-platform proof |
| 7 | **Multi-Device** | Moderate | 3 parallel WEB_IDs across full dataset span |

## Project Structure

```
├── Multi_Device_Detection_Analysis.ipynb      # Full analysis notebook
├── device_activity_logs.csv                   # Input: 7,915 activity records
├── multi_device_classification_results.csv    # Output: per-record classifications
└── README.md
```

## Tech Stack

- **Python** — pandas, numpy
- **Visualization** — matplotlib, seaborn (Gantt charts, heatmaps, stacked bars)
- **Statistics** — Jaccard Index for set similarity

## Key Techniques

- **Evidence-layered heuristic classification** — 5 independent signals combined through a decision tree, not a scoring formula
- **Lifespan overlap analysis** — pairwise overlap ratio computation for device ID temporal windows
- **Jaccard similarity** — IP address neighborhood comparison between device IDs
- **ID type weighting** — different confidence levels for GAID (hardware-bound) vs WEB_ID (cookie-volatile)
- **Volume-based noise filtering** — minimum record threshold to separate real device footprints from artifacts

## How to Run

```bash
pip install pandas numpy matplotlib seaborn
jupyter notebook Multi_Device_Detection_Analysis.ipynb
```

## Author

**Asaf Gartenberg** — March 2026
