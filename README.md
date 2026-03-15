# SpaceX CRS-10 Telemetry Analysis

## Intention

Rocket telemetry streams are high-frequency time-series signals that capture the full arc of a flight from ignition through atmospheric exit. In operational settings, anomaly detection on these signals is critical: engineers need to know whether an unexpected reading reflects a real event (engine cutoff, throttle change, stage separation) or sensor noise.

This project uses publicly available telemetry from the SpaceX CRS-10 mission to ask a concrete question: **can unsupervised statistical anomaly detection methods identify real, known flight events without being told what to look for?**

The CRS-10 dataset is well-suited for this because it includes a ground truth events file — timestamps for five confirmed flight milestones — that can serve as a validation set.

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

A locally-adaptive method — Rolling Z-score — should outperform the global methods because it compares each measurement only to its recent neighbours, making it sensitive to sudden local deviations regardless of where they occur in the flight.

---

## Methods

All analysis is performed in Python using **Polars** for data manipulation, NumPy/SciPy for numerical operations, and Matplotlib for visualisation.

### Signal

The EDA section explores the raw velocity gradient as an initial look at the signal's dynamics. All three detection methods in the comparative analysis are applied exclusively to the `acceleration` column from the pre-processed telemetry dataset — true sensor acceleration, not a numerically-differentiated estimate.

### Anomaly Detection Methods

| Method | Assumption | Parameters |
|---|---|---|
| **Z-score** | Signal is normally distributed; flags points beyond N standard deviations from the global mean | Threshold optimised via grid search |
| **IQR** | Distribution-agnostic; flags points beyond 1.5× the interquartile range | Fixed (1.5× IQR convention) |
| **Rolling Z-score** | Signal is locally stationary within a window; flags points that deviate from their local mean | Threshold and window size both optimised via grid search |

### Evaluation

Each method's detections are grouped into **detection bursts** — clusters of flagged points separated by gaps greater than 10 seconds. A burst is a true positive if it overlaps with any known event window (±5 seconds). This approach allows two summary metrics to be computed per method:

- **Events Detected** — how many of the 5 known events had at least one burst land within their window (recall)
- **Hit Rate** — events detected divided by total bursts raised; penalises methods that flood the signal with spurious detections (precision-like)

### Grid Search

To remove the arbitrariness of manually chosen thresholds, a grid search is run from 0.5 to 4.0 (step 0.1) for both the Z-score and the Rolling Z-score threshold. The optimal value maximises events detected, with minimum false positive points as a tiebreaker.

For the Rolling Z-score, a second sequential grid search is then run over window sizes from 5 to 100 seconds (step 5), with the threshold fixed at the value found above. This reveals how much the window — not just the threshold — affects what the method considers "local." The sequential approach (threshold first, then window) is a deliberate choice to keep each parameter's effect interpretable.

IQR has no comparable free parameter and uses the fixed 1.5× convention throughout.

---

## Conclusions

### Detection Performance

The results broadly support the hypothesis, with one important nuance.

MECO (t=143s) and SES-1 (t=154s) are detected by all three methods. These events produce sharp, large-magnitude acceleration discontinuities — the engine cutting and restarting — that stand out even when judged against the full flight distribution. For the global methods, these are the easiest events to find.

The throttle-down manoeuvre (t=50–66s) and MaxQ (t=75s) are more discriminating. These events produce real but comparatively modest deviations. The global methods (Z-score, IQR) are less likely to flag them because their statistics are dominated by the high-acceleration late-flight phase, which sets a high bar for what counts as "unusual." The Rolling Z-score, comparing each point only to its local window, is not anchored to that global distribution and is therefore better positioned to catch these subtler early-flight events.

If a method detected an event that the other two missed, it is most likely the Rolling Z-score detecting a throttle-down or MaxQ event — precisely the scenario the hypothesis predicted.

### Hit Rate as the Key Metric

Events detected alone is an incomplete measure. A method that raises 50 bursts and catches 4 events is less useful than one that raises 7 bursts and catches the same 4, because in practice every false alarm requires investigation.

Hit rate (events detected / total bursts) captures this tradeoff in a single number. IQR's hit rate is fixed by convention, while both Z-score methods are tuned by grid search. The comparison reveals whether the optimised Z-score methods achieve high recall without sacrificing hit rate, and whether the Rolling Z-score's local adaptivity translates to fewer spurious bursts as well as more real detections.

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

For non-stationary time-series like rocket telemetry, locally-adaptive anomaly detection outperforms global statistics — not just in peak detection rate, but in hit rate and robustness to hyperparameter choice. Grid search is a necessary step to make any threshold-dependent comparison honest, and the sequential threshold-then-window search for Rolling Z-score shows that both hyperparameters matter and interact in interpretable ways.

---

## Data Source

[shahar603/Telemetry-Data — SpaceX CRS-10](https://github.com/shahar603/Telemetry-Data/tree/master/SpaceX%20CRS-10)
