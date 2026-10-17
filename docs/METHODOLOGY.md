# Methodology and statistical notes

[← Back to README](../README.md) · [Results and limitations](RESULTS_AND_LIMITATIONS.md)

This page explains the choices behind the notebook: how I handled the data, why I used these models and metrics, and what the evaluation can actually tell me.

## 1. The dataset and class imbalance

The published ULB dataset contains **284,807 transactions**, including **492 fraud cases** (about **0.173%**). It has 31 columns: `Time`, `Amount`, `V1`–`V28`, and the target `Class`.

- `Class = 0`: legitimate transaction; `Class = 1`: fraud.
- `V1`–`V28` are anonymized PCA components; their original business meanings are not available.
- `Time` is the number of seconds since the first recorded transaction, **not** a clock hour.

A classifier that always predicts the majority class would score roughly **99.83% accuracy** on the original data while catching **zero fraud cases**. This is why accuracy is not my main measure.

## 2. Data quality and splitting

I check the expected columns, numeric types, missing values, labels, and duplicates before modeling.

In the saved run, I removed **1,081 exact duplicate rows**, including **19 fraud rows**. This avoids putting identical full observations in different splits, but it is a **modeling assumption**, not proof that the duplicated transactions are erroneous. Without transaction identifiers, distinct real transactions may look identical.

I then use a stratified split:

| Subset | Approximate share | Purpose |
|---|---:|---|
| Training | 60% | Fit preprocessing and models; explore feature patterns |
| Validation | 20% | Compare models and select the illustrative threshold |
| Test | 20% | Final evaluation after choices are fixed |

Stratification helps preserve the rare fraud rate across subsets. It does not remove the limitation of using a random split of historical transactions. A **time-based split** would be a better next step for testing future generalization.

## 3. Exploratory statistics

I perform exploratory analysis on the **training set only**.

**Mean, median, and skewness.** A few large transaction amounts can pull the mean upward. The median is more resistant to extremes. The notebook uses `log(1 + Amount)` for a readable histogram, **not** as a model transformation.

**Interquartile range (IQR).** IQR = Q3 − Q1 describes the middle half of a distribution. The common 1.5 × IQR rule is useful for identifying unusual values, but in fraud detection those unusual cases may be the most informative. I do not delete them automatically.

**Pearson correlation.** I compare each feature with the binary target. With a 0/1 target, this is related to point-biserial correlation. Correlation measures linear association, not causation; it also misses some nonlinear relationships. The selected PCA features have very small pairwise correlations in the training sample, which is consistent with the PCA representation.

## 4. The two models

### Logistic Regression baseline

Logistic Regression is a relatively simple linear classifier. I use a pipeline with median imputation, standard scaling, and `class_weight="balanced"`.

Its predicted score follows a logistic transformation of a linear combination of features:

$$
P(y=1\mid x) = \frac{1}{1+e^{-(\beta_0+\beta^T x)}}
$$

The dataset has no missing feature values in the saved run, so the imputer acts as a safeguard. I did not optimize the linear model or transform the strongly skewed `Amount` feature; it is a **reference**, not a fully tuned competitor.

### Random Forest

A decision tree repeatedly splits the data to make its child nodes more homogeneous. The notebook uses **Gini impurity** to explain that idea:

$$
Gini = 1 - \sum_{k=1}^{K}p_k^2
$$

For two classes, a pure node has Gini impurity 0; a 50/50 node has impurity 0.5. A tree chooses splits that reduce weighted impurity.

A Random Forest adds two sources of variation: each tree sees a **bootstrap sample** of training rows, and each split considers a **random subset of features**. Averaging across many trees usually makes the result less dependent on any one tree.

The saved model uses:

| Setting | Value |
|---|---|
| `n_estimators` | `150` |
| `min_samples_leaf` | `2` |
| `max_features` | `sqrt` (default) |
| `class_weight` | `balanced_subsample` |
| `random_state` | `42` |

These are practical choices, **not the result of a systematic hyperparameter search**. The forest uses median imputation but no scaling, because tree splits are not sensitive to monotonic changes in feature scale.

## 5. Evaluation metrics

| Metric | Formula / idea | Why I use it |
|---|---|---|
| Precision | TP / (TP + FP) | How many alerts are actually fraud? |
| Recall | TP / (TP + FN) | How many fraud cases are found? |
| F1 | Harmonic mean of precision and recall | Balance at one chosen threshold |
| Average precision (AP) | Precision averaged over recall increases | Ranking quality for the rare class |
| ROC-AUC | Area under the ROC curve | General ranking discrimination |

Here, **TP** means a fraud case caught, **FP** means a legitimate transaction flagged, and **FN** means a fraud case missed.

**AP is not automatically the same number as trapezoidal PR-AUC.** I report scikit-learn's `average_precision_score` by its correct name.

Logistic Regression has higher validation ROC-AUC, but Random Forest has higher validation AP. Those metrics emphasize different aspects of the ranking; with fraud this rare, I pay close attention to AP, precision–recall curves, and the number of false alerts.

## 6. Choosing the threshold without test leakage

A model's score ranking and its binary alerts are different things. The alert threshold controls the precision–recall trade-off.

The notebook uses an **illustrative** validation recall target of **80%**. Among validation thresholds that meet this target, it selects the one with the highest precision; ties favor the higher threshold.

In the saved run:

- Selected threshold: **0.5463**.
- Validation precision: **92.86%**.
- Validation recall: **82.98%**.
- Thresholds in approximately **(0.5460, 0.5463]** produce the same validation alerts.

The 80% target is a learning exercise, **not a bank policy or an optimized business objective**. The test set is not used to choose the threshold. The notebook then checks both the default and selected operating points on the held-out test set.

## 7. Uncertainty in recall

The test set contains only **95 actual fraud cases**, even though it contains tens of thousands of transactions overall. This makes recall uncertain.

The notebook calculates an approximate **95% Wilson confidence interval** for the proportion of fraud cases caught. It is a binomial approximation and assumes independent, similarly distributed observations. It does **not** account for changing fraud patterns, correlations between transactions, or the earlier model/threshold choices.

The interval is useful context, not a guarantee about future performance. See [Results and limitations](RESULTS_AND_LIMITATIONS.md) for the saved values.
