# Results, interpretation, and limitations

[← Back to README](../README.md) · [Methodology](METHODOLOGY.md)

These numbers come from the **saved execution of the notebook** on the ULB source CSV, after removing exact duplicate rows. They are observations from one historical random split, not independent benchmarks or claims of production performance.

## 1. Validation: ranking performance

| Metric | Logistic Regression | Random Forest |
|---|---:|---:|
| Average precision (AP) | 0.7868 | **0.8742** |
| ROC-AUC | **0.9790** | 0.9554 |
| Precision at 0.5 | 0.0538 | **0.9186** |
| Recall at 0.5 | **0.9043** | 0.8404 |
| F1 at 0.5 | 0.1016 | **0.8778** |

At threshold 0.5, Logistic Regression caught **85/94** validation fraud cases but flagged **1,494** legitimate transactions. Random Forest caught **79/94** and flagged **7** legitimate transactions.

This is a useful illustration of alert workload, but **not a fair threshold-by-threshold comparison**: Logistic Regression uses balanced class weights, and the models' score distributions differ. I compare ranking with AP and the precision–recall curves instead of declaring a winner solely from the 0.5 rows.

## 2. Threshold selection on validation

I set an illustrative goal of at least **80% recall** on validation and chose the threshold with the highest precision among those meeting it.

| Validation measure | Saved result |
|---|---:|
| Selected Random Forest threshold | **0.5463** |
| Precision at selected threshold | **0.9286** |
| Recall at selected threshold | **0.8298** |
| Next lower score | approximately 0.5460 |

The exact threshold depends on this validation sample. It is not an operational recommendation.

## 3. Final Random Forest test evaluation

| Measure | Saved result |
|---|---:|
| Test transactions | 56,746 |
| Actual fraud cases | 95 |
| Average precision (AP) | **0.8149** |
| ROC-AUC | **0.9338** |
| Precision | **0.9714** |
| Recall | **0.7158** |
| F1 | **0.8242** |
| True positives (fraud caught) | **68** |
| False negatives (fraud missed) | **27** |
| False positives (legitimate transactions flagged) | **2** |
| Approximate 95% Wilson interval for recall | **61.8%–79.7%** |

The default threshold (0.5) and the validation-selected threshold (approximately 0.5463) happened to produce **identical predictions on this test split**. No test score fell in the interval needed to change an alert. The notebook shows one confusion matrix, shaded by the percentage within each actual class, to keep the rare fraud row readable. This is not evidence that threshold selection is always irrelevant.

## 4. What I take from these results

**The validation result did not fully carry over.** Random Forest recall fell from **83.0%** on validation to **71.6%** on test, and AP fell from **0.8742** to **0.8149**. Precision increased from **92.9%** to **97.1%**, because the model raised only two false alerts on the test set.

**A high precision score is not the whole story.** The model missed 27 of the 95 actual fraud cases. For a real review team, the relative costs of missed fraud and false alerts would have to guide the operating threshold.

**The 80% recall target was not demonstrated on test.** The observed test recall is 71.6%, and the approximate Wilson interval is **61.8%–79.7%**. That interval describes only sampling uncertainty under its assumptions; it does not include time drift or uncertainty from selecting the model and threshold.

## 5. Limitations

- **Historical and random split:** The dataset covers a short period in 2013; a chronological holdout would better test changes in fraud patterns.
- **Exact duplicates:** Removing 1,081 exact rows also removed 19 fraud cases. Without unique transaction IDs, I cannot verify that these rows were erroneous or truly repeated events.
- **Few positive test cases:** With only 95 fraud cases, a handful of misses can change recall substantially.
- **Untuned models:** The forest uses practical fixed settings, and the Logistic Regression baseline is not optimized. Results are not a comprehensive model benchmark.
- **Calibration and costs:** The predicted scores have not been shown to be calibrated probabilities, and the threshold does not use a real monetary cost model.
- **Limited interpretability:** PCA features are anonymized; impurity-based feature importance is not a causal explanation and can favor features with many potential splits.

## 6. Next experiments

I would first test a **chronological holdout** and examine how stable the threshold is across validation samples. After that, I would compare a better-tuned Logistic Regression baseline, try modest Random Forest cross-validation, and assess probability calibration. If business costs were available, I would use them to evaluate alert thresholds rather than relying on an illustrative recall target.

## Reproducing these results

Follow the [setup instructions in the README](../README.md#run-the-notebook). The source CSV is not distributed in this repository. Library versions and execution environments may cause small differences from the saved outputs.
