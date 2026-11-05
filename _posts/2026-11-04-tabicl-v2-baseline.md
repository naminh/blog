---
layout: post
title:  "TabICLv2 is your new default baseline"
date:   2026-11-04 09:30:00 +0000
categories: "ML-Posts"
---

## TabICLv2: the baseline you should run first

For years the tabular playbook was: try logistic regression, then XGBoost or CatBoost, tune for a day, ship. That default is changing. **Tabular foundation models** now beat gradient-boosted trees on the big benchmarks, and **TabICLv2** is one of the best, and it's open source.

### What it is

TabICLv2 comes from the SODA team at Inria. It's a transformer pretrained entirely on synthetic tables. At prediction time you hand it your training rows as *context* and it predicts straight away, with no gradient descent on your data. That's in-context learning, as with LLMs, but for tables. The paper is [arXiv 2602.11139](https://arxiv.org/abs/2602.11139) (ICML 2026).

It's a drop-in scikit-learn style estimator:

```python
# pip install tabicl
from tabicl import TabICLClassifier

clf = TabICLClassifier()
clf.fit(X_train, y_train)      # downloads the checkpoint on first use, otherwise cheap
proba = clf.predict_proba(X_test)
```

No hyperparameters to tune, and a regressor (`TabICLRegressor`) is included.

### How it compares

| | Tuning needed | Strengths | Weak spots |
|---|---|---|---|
| **TabICLv2** | None | Strong out of the box, small to mid-sized data | Needs a GPU for larger data, pretrained for limited size and width |
| **XGBoost / LightGBM** | A fair bit | Fast, scalable, mature tooling | Needs tuning for top results |
| **CatBoost** | Less than the others | Great with categoricals | Slower to train |
| **Tabular deep learning** (RealMLP, TabM) | Moderate to heavy | Can shine on big data | Fiddly, often beaten by trees |
| **Other foundation models** (TabPFN) | None | Same idea | TabICLv2 reports being ahead and much faster |

What the authors report:

- Without any tuning, TabICLv2 beats RealTabPFN-2.5 on the TabArena and TALENT benchmarks. RealTabPFN-2.5 was hyperparameter-tuned, ensembled and fine-tuned.
- The project README says it beats heavily tuned XGBoost, CatBoost or LightGBM on about **80% of TabArena datasets**.
- It's reported as roughly 10x faster than TabPFN-2.5 on an H100 (50K rows, 100 features).

A bit of honesty: these are the authors' own numbers, and I couldn't read the full per-model result tables. Also, on TabArena an extreme 4-hour AutoGluon ensemble still had a better average rank than TabICLv2 in the paper, so it isn't a magic bullet.

### Why make it your baseline

- **Zero tuning effort.** You get a strong number in minutes, which shows how much a tuned GBM has to beat to be worth the work.
- **Honest ceiling check.** If your fancy pipeline can't beat it, that's useful to know early.
- **Good with little data.** In the few-shot (16 sample) results in the paper it ranked near the top.

### Caveats

- **Size limits.** The README says it was pretrained on 300 to 48K rows and 2 to 100 columns. It reports good results up to about 100K rows and says it can degrade past ~100 features. Bigger data may need offloading.
- **GPU recommended** for larger datasets. Inference does the heavy lifting at predict time.
- **Ignores column names.** It doesn't use text semantics.
- **Serving.** Trees are still much cheaper and easier to deploy at high QPS. A common outcome is TabICLv2 as the benchmark and a GBM in production, unless the lift justifies the cost.

### Take-away

Add `TabICLClassifier` next to your logistic regression and XGBoost in every new tabular project. It costs five lines and minutes of compute, and it tells you how good "good" is.

**Sources:** [TabICLv2 paper](https://arxiv.org/abs/2602.11139), [soda-inria/tabicl](https://github.com/soda-inria/tabicl)
