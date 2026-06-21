# Multi-Device User Detection

A forensic data-analysis project that determines which users in an activity log are operating **more than one physical device** — separating genuine multi-device usage from the "phantom identities" (cleared cookies, reset advertising IDs, app reinstalls) that a naive identifier count would mistake for extra hardware.

![Python](https://img.shields.io/badge/python-3.9%2B-blue)
![pandas](https://img.shields.io/badge/pandas-data%20analysis-150458)
![Jupyter](https://img.shields.io/badge/Jupyter-notebook-orange)
![Status](https://img.shields.io/badge/status-complete-success)

---

## Why this exists

Counting each user's device identifiers and flagging anyone with more than one sounds simple — and is almost always wrong. Multiple device IDs do **not** mean multiple devices: cookies get cleared, advertising IDs get reset, and apps get reinstalled, each event minting a fresh identifier on the *same* piece of hardware. A naive count floods you with false positives.

This project defines "a different device" strictly — a separate physical phone, tablet, or computer, not just a new identifier or browser session — and then builds a **layered evidence hierarchy** where each signal upgrades or downgrades confidence based on how tightly it binds to real hardware. Every classification is traceable back to the specific signals and thresholds that produced it.

## The dataset

7,915 activity records spanning **7 users** and **20 device identifiers**, with these fields:

| Field | Meaning |
|-------|---------|
| `REQUEST_TIME` | Timestamp — `MM:SS.s` only, no date or hour (a key constraint) |
| `DEVICE_IP` | Source IP (IPv4 / IPv6) |
| `ID` | Device identifier value |
| `USER_ID` | The user the activity belongs to |
| `ID_TYPE` | Identifier technology: `GAID`, `IDFA`, `IDFV`, or `WEB_ID` |
| `OPERATING_SYSTEM` | `android` or `ios`, derived from the User-Agent |

A useful simplification surfaced during profiling: each user stays within a single `ID_TYPE`, so each user can be reasoned about through one identifier technology and its known volatility.

## Approach — a 5-layer evidence hierarchy

Each layer is applied in order of forensic strength. A decisive layer can close a case on its own; weaker layers only corroborate.

| Layer | Signal | Role |
|-------|--------|------|
| **1 — OS divergence** | Same user emitting both `ios` and `android` telemetry | **Decisive** — an Android device can't emit iOS telemetry and vice-versa; closes the case alone |
| **2 — ID lifespan overlap** | Do a user's IDs run *sequentially* (resets on one device) or *in parallel* (active in the same window)? | **Primary signal** |
| **3 — Temporal concurrency** | "Two hands" test — near-simultaneous events from different IDs | **Rejected** — timestamps lack dates, making concurrency unreliable |
| **4 — IP Jaccard similarity** | Set-overlap of IP addresses between IDs | **Corroborative only** — never standalone (VPNs, CGNAT, Wi-Fi↔cellular handoff all distort it) |
| **5 — Volume threshold** | Drop IDs with fewer than 8 records | **Noise filter** — screens out stale-cookie and redirect artifacts |

### Identifier forensic weight

The same number of IDs means very different things depending on the identifier type — this weighting drives Layer 2:

| ID type | Platform | Persistence | Weight | Multiples suggest… |
|---------|----------|-------------|--------|--------------------|
| `GAID` | Android | Hardware-bound ad ID; resettable but uncommon | Strong | Overlapping lifespans → strong multi-device signal |
| `IDFA` | iOS | Hardware-level ad ID | Strong | Same as above |
| `IDFV` | iOS | Resets only when all vendor apps are uninstalled | Strong | Simultaneous active IDFVs → likely separate devices |
| `WEB_ID` | Web | Cookie-based; dies on cache clear | Weak | Easily produced by one device (tabs, incognito, cache clears) |

### Confidence tiers

| Tier | Evidence | Classification |
|------|----------|----------------|
| **Decisive** | Cross-platform OS divergence | Multi-device — no ambiguity |
| **High** | Same OS + overlapping lifespans + substantial volume + strong ID type | Likely multi-device |
| **Moderate** | Overlap but weak ID type, or strong ID type at low volume, or mixed signals | Possibly multi-device — both scenarios documented |
| **Low** | Sequential IDs, a single ID, or low-volume `WEB_ID`s only | Likely single device |

## Results

| User | Platform | Identifiers | Verdict |
|------|----------|-------------|---------|
| **6** | iOS + Android | mixed | **Multi-Device — Decisive** (783 iOS + 79 Android records) |
| **2** | iOS | 5 IDFVs | **Multi-Device — High** (heavy lifespan overlap, substantial volume) |
| **5** | iOS | 2 IDFAs | **Multi-Device — Moderate** (two hardware-level IDs, overlapping) |
| **7** | Android | 3 WEB_IDs | **Multi-Device — Moderate** (sustained parallel activity; cookie volatility limits certainty) |
| **1** | Android | 3 GAIDs | Likely Single Device (sequential resets) |
| **3** | Android | 4 WEB_IDs | Likely Single Device (low volume, normal cookie churn) |
| **4** | — | 1 ID | Single Device — Decisive |

The output file maps the final classification onto all 7,915 rows for downstream use. Where evidence is ambiguous (e.g. Users 1 and 7), the notebook documents **both** scenarios rather than overstating certainty.

## Project structure

```
.
├── multi_device_analysis.ipynb   # The full analysis, narrated layer by layer
├── activity_log.csv              # Input: 7,915 activity records
├── classified_output.csv         # Output: every row tagged with its classification
└── decision_funnel.html          # Standalone visual of the 5-layer hierarchy
```

> The notebook is self-documenting: each layer is introduced, justified, applied, and interpreted per user, with an appendix of methods that were considered and rejected.

## Running it

```bash
pip install pandas numpy matplotlib seaborn jupyter
jupyter notebook multi_device_analysis.ipynb
```

Run the cells top to bottom. The notebook loads the activity log, builds each layer in sequence, prints per-user interpretations, renders the supporting charts, and writes `classified_output.csv`.

## Methods considered and rejected

Part of the rigor here is being explicit about what *didn't* make the cut and why:

- **Temporal concurrency ("two hands" test)** — would catch two IDs firing within seconds of each other, but the timestamps carry no date or hour, so co-occurrence can't be trusted.
- **Impossible-travel / geolocation** — the data has no coordinates, and IP-to-geo lookups are too coarse (especially for IPv6 and carrier IPs) to support travel-time reasoning.
- **Unsupervised clustering (DBSCAN / K-Means)** — with only 7 users and 20 identifiers, there isn't enough data for clustering to be meaningful over transparent, explainable rules.

## Tech stack

Python · pandas · NumPy · Matplotlib · Seaborn · Jupyter

## Limitations

- **No absolute timeline.** Timestamps lack dates/hours, so chronology is approximated by row index; concurrency-based reasoning is deliberately avoided.
- **IP is a weak fingerprint.** VPNs, CGNAT, and network handoffs mean IP overlap can only ever support a conclusion, never establish one.
- **Small sample.** Findings describe these 7 users; the approach generalizes, but the specific verdicts do not.
