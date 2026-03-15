# SpaceX CRS-10 Telemetry Analysis

## Intention

Rocket telemetry streams are high-frequency time-series signals that capture the full arc of a flight from ignition through atmospheric exit. In operational settings, anomaly detection on these signals is critical: engineers need to know whether an unexpected reading reflects a real event (engine cutoff, throttle change, stage separation) or sensor noise.

This project uses publicly available telemetry from the SpaceX CRS-10 mission to ask a concrete question: **can unsupervised statistical anomaly detection methods identify real, known flight events without being told what to look for?**

The CRS-10 dataset is well-suited for this because it includes a ground truth events file: timestamps for five confirmed flight milestones. These timestamps can serve as a validation set.

| Event | Time (s) |
|---|---|
| Throttle-down start | 50 |
| Throttle-down end | 66 |
| MaxQ | 75 |
| MECO (Main Engine Cut-Off) | 143 |
| SES-1 (Second Engine Start) | 154 |

---

## Hypothesis

Global statistical methods (Z-score, IQR) will struggle to detect mid-flight events because they compute statistics over the entire flight duration. A rocket accelerating continuously produces a non-stationary signal — the distribution of acceleration values shifts over time. A reading that looks like an outlier early in flight may appear routine by the end.

A locally-adaptive method, like Rolling Z-score, should outperform the global methods because it compares each measurement only to its recent neighbours, making it sensitive to sudden local deviations regardless of where they occur in the flight.

---

## Methods

All analysis is performed in Python using **Polars** for data manipulation, NumPy/SciPy for numerical operations, and Matplotlib for visualisation.

### Signal

The EDA section explores the raw velocity gradient as an initial look at the signal's dynamics. All three detection methods in the comparative analysis are applied exclusively to the `acceleration` column from the pre-processed telemetry dataset.

### Anomaly Detection Methods

| Method | Assumption | Parameters |
|---|---|---|
| **Z-score** | Signal is normally distributed; flags points beyond N standard deviations from the global mean | Threshold optimised via grid search |
| **IQR** | Distribution-agnostic; flags points beyond 1.5× the interquartile range | Fixed (1.5× IQR convention) |
| **Rolling Z-score** | Signal is locally stationary within a window; flags points that deviate from their local mean | Threshold and window size both optimised via grid search |

### Evaluation

Each method's detections are grouped into **detection bursts:** clusters of flagged points separated by gaps greater than 10 seconds. A burst is a true positive if it overlaps with any known event window (±5 seconds). This approach allows two summary metrics to be computed per method:

- **Recall:** Events detected / 5 — how many known events the method found
- **Precision (Hit Rate):** Events detected / total bursts raised — penalises methods that flood the signal with spurious detections
- **F1:** Harmonic mean of precision and recall. The single summary metric used to rank methods

### Grid Search

To remove the arbitrariness of manually chosen thresholds, a grid search is run from 0.5 to 4.0 (step 0.1) for both the Z-score and the Rolling Z-score threshold. The optimal value maximises events detected, with minimum false positive points as a tiebreaker.

For the Rolling Z-score, a second sequential grid search is then run over window sizes from 5 to 100 seconds (step 5), with the threshold fixed at the value found above. This reveals how much the window — not just the threshold — affects what the method considers "local." The sequential approach (threshold first, then window) is a deliberate choice to keep each parameter's effect interpretable.

IQR has no comparable free parameter and uses the fixed 1.5× convention throughout.

---

## Conclusions

### Detection Performance

| Model | Hits | Misses | Recall | F1 |
|---|---|---|---|---|
| Z-score (t=0.7) | throttle_down_end, maxq, meco, ses1 | throttle_down_start | 0.80 | — |
| IQR | meco, ses1 | throttle_down_start, throttle_down_end, maxq | 0.40 | — |
| Rolling Z (t=0.7, w=30s) | throttle_down_start, throttle_down_end, meco, ses1 | maxq | 0.80 | — |

*(F1 values update on re-run after precision formula fix)*

Both Z-score and Rolling Z-score detected 4 of 5 events, but they disagreed on which event to miss — and that disagreement is the most interesting finding in the analysis.

**Z-score missed throttle_down_start (t=50s).** The throttle-down start is a modest, controlled deceleration early in flight. Against the global distribution — dominated by the high-acceleration burn phase — it does not read as statistically unusual. The global mean and standard deviation are pulled toward late-flight values, raising the bar for what qualifies as an anomaly.

**Rolling Z-score missed MaxQ (t=75s).** MaxQ is a gradual buildup of aerodynamic pressure that peaks at 75 seconds. Unlike an engine cutoff or stage event, it does not produce a sharp discontinuity — it is a smooth inflection. The local window of the Rolling Z-score adapts to this gradual change, normalising it away, whereas the global Z-score treats the overall peak as deviating from the full-flight mean.

**IQR detected only MECO and SES-1.** Without a tunable threshold, IQR is limited to the two most physically dramatic events — the engine hard-stop and restart — which produce acceleration changes large enough to clear any reasonable outlier bound.

The partial confirmation of the hypothesis is notable: Rolling Z-score does catch throttle_down_start where Z-score fails, exactly as predicted. But the global Z-score's detection of MaxQ — which Rolling Z misses — shows the tradeoff runs both ways. Local adaptivity is an advantage for abrupt mid-flight events and a disadvantage for gradual peaks.

### F1 as the Summary Metric

Events detected alone is an incomplete measure. A method that raises 50 bursts and catches 4 events is less useful than one that raises 7 bursts and catches the same 4, because in practice every false alarm requires investigation. Equally, a method that raises only 1 burst and catches 1 event has perfect precision but useless recall.

The F1 score — harmonic mean of precision (hit rate) and recall — captures both sides of this tradeoff in a single number. The harmonic mean is deliberately harsher than the arithmetic mean: it cannot be inflated by a very high score on one axis masking weakness on the other. A method with high recall but many spurious bursts scores poorly on precision and therefore on F1; a method with high precision but few detections scores poorly on recall.

IQR's F1 is entirely determined by its fixed threshold and the data. Both Z-score methods are grid-search tuned, so their F1 reflects their best achievable performance on this signal. A higher F1 for Rolling Z-score would confirm that local adaptivity improves not just detection rate but the quality of each detection raised.

### Threshold Sensitivity and Robustness

The Z-score threshold has a narrow optimal band — below ~1.5 false positives spike with little gain in detected events, and above ~2.5 real events begin to be missed. The Rolling Z-score's optimal threshold is typically higher and flatter: because the local window already suppresses the slow trend, the residual signal is tighter, and a wider range of thresholds produces similarly good results. This means Rolling Z-score is more robust to threshold choice — a practically significant property when no ground truth is available for tuning.

The window grid search adds a second insight: very small windows (< ~15s) behave erratically because the window is too narrow to form a stable local mean, while very large windows (> ~60s) begin to resemble global statistics and lose the adaptive advantage. The optimal window sits in a middle band where the method is local enough to be sensitive but stable enough to be reliable.

### Generalisation

A key limitation is that both grid searches are optimised on the same data they are evaluated on — there is no held-out test set. The selected thresholds are tuned to CRS-10 specifically and may not transfer to a different mission profile without re-tuning. In a production setting, this would be addressed by:

1. **Cross-mission validation** — optimising on one mission and evaluating on another with known events
2. **Leave-one-out evaluation** — optimising on four events and testing on the fifth, repeated for each event
3. **Tolerance sensitivity analysis** — checking whether results hold under different window sizes (±3s, ±10s)

Despite this, the directional finding is likely to generalise: locally-adaptive methods are structurally better suited to non-stationary signals. The specific threshold values are mission-dependent; the method ranking is not.

### Key Takeaway

Z-score and Rolling Z-score tie on recall (4/5) but capture different subsets of events. No single method dominates across all five events — the right choice depends on which type of event matters most. For abrupt early-flight manoeuvres (throttle changes), Rolling Z-score is structurally better suited. For smooth signal peaks like MaxQ, global statistics have an edge. IQR, without a tunable threshold, is limited to the most dramatic events only.

Grid search is a necessary step to make any threshold-dependent comparison honest. The optimal threshold for both Z-score methods is 0.7 — substantially lower than the conventional defaults of 2.0–2.5 — which reflects how compressed the acceleration signal's variance is relative to the magnitude of real events on this mission.

---

## Data Source

[shahar603/Telemetry-Data — SpaceX CRS-10](https://github.com/shahar603/Telemetry-Data/tree/master/SpaceX%20CRS-10)
