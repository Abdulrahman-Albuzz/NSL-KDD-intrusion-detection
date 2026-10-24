# Network Intrusion Detection with Machine Learning (NSL-KDD)

A beginner-friendly AI security project: train models to tell normal network traffic from attacks, then find out where they fail and why.

Built in Google Colab with Python, pandas and scikit-learn.

## What this project does

I trained machine learning models on the NSL-KDD benchmark dataset to classify each network connection as **normal** or **attack**. The interesting part was not the first score, but what happened when I looked closer: the models missed whole families of attacks, I tested two explanations for why, and I ended up fixing the problem with a decision threshold chosen on a proper validation set.

## Dataset

[NSL-KDD](https://www.unb.ca/cic/datasets/nsl.html) is a cleaned-up version of the older KDD Cup 1999 intrusion dataset. Each row is one network connection described by 41 features (protocol, bytes sent, error rates and so on) plus a label.

| File | Rows | Used for |
|---|---|---|
| `KDDTrain+_20Percent.txt` | 25,192 | Main training set (a 20% sample of the full training file) |
| `KDDTrain+.txt` | 125,973 | Full training file, used to check whether more data helps |
| `KDDTest+.txt` | 22,544 | Official test set (contains attack types not seen in training) |

Attacks fall into four families: **DoS** (flooding), **Probe** (scanning), **R2L** (remote attacks such as password guessing) and **U2R** (privilege escalation).

## Method

1. Labels turned into binary (normal = 0, attack = 1).
2. Dropped the `difficulty` column, since it is not a real network feature.
3. One-hot encoded the text columns (protocol, service, flag) and scaled the numeric ones. Preprocessing was fitted on training data only to avoid data leakage.
4. Trained Logistic Regression and a Decision Tree as baselines, then a Random Forest.
5. Broke results down by attack type and attack family instead of relying on one overall score.
6. Tuned the alert threshold using a validation set that hides some attack types from training, then evaluated on the test set once.

## Results

### Baselines (default settings)

| Model | Accuracy | Attack recall | Attack precision |
|---|---|---|---|
| Logistic Regression | 0.75 | 0.62 | 0.92 |
| Decision Tree | 0.81 | 0.70 | 0.96 |
| Random Forest | 0.78 | 0.63 | 0.97 |

The models were very careful: when they flagged an attack they were nearly always right, but they let 30 to 38% of real attacks through.

### Detection by attack family

| Family | Default threshold (0.5) | Tuned threshold (0.02) |
|---|---|---|
| DoS | 83% | 99% |
| Probe | 72% | 100% |
| R2L | 6% | 81% |
| U2R | 13% | 87% (only 67 samples) |

Final test performance with the tuned threshold: **attack recall 0.949, precision 0.911, false alarm rate 0.123.**

## Key findings

- **Overall accuracy hid the problem.** Breaking results down by attack type showed that R2L and U2R attacks were almost completely missed.
- **"Not enough data" was wrong.** Training on five times more data left R2L detection at about 6%.
- **"The features can't see these attacks" was also wrong.** Lowering the alert threshold lifted R2L detection to over 50%, so the model had the signal but was too cautious at the default 50% cutoff.
- **Threshold choice is a real decision.** Catching more attacks costs more false alarms, and the right balance depends on what a missed attack costs.
- **Evaluation design matters.** Hiding four attack types (`nmap`, `teardrop`, `warezclient`, `pod`) from training gave a validation set that imitates facing unseen attacks, which a random split cannot do.

## Limitations

- The validation false alarm rate did not carry over: the rule targeted 5% or less, but the test set showed 12.3%. The test set's normal traffic differs from the training distribution.
- The chosen threshold (0.02) was the lowest value in my search grid, so a wider grid might choose differently.
- A 12% false alarm rate would be unusable in practice. Attacks are 57% of this test set but a tiny share of real traffic, so real alerts would be buried.
- NSL-KDD is based on old traffic and does not reflect modern attacks or encrypted traffic.
- The model predicts attack or normal only, not which attack.
- U2R has very few test samples, so its numbers are rough.
- One model setting and one random seed; no repeated runs.

## How to run it

1. Download NSL-KDD (for example from the Kaggle dataset `hassan06/nslkdd`) and unzip it.
2. Open the notebook in Google Colab (`NSL_KDD.ipynb`).
3. Upload `KDDTrain+_20Percent.txt`, `KDDTest+.txt` and `KDDTrain+.txt` when the notebook asks for them.
4. Run the cells in order.

Requirements: Python 3, pandas, scikit-learn, matplotlib (all preinstalled in Colab).

## Next steps

- Wider threshold search and cross-validation for a more reliable choice.
- Multi-class classification to predict the attack type.
- Repeat on a more modern dataset such as UNSW-NB15 or CIC-IDS2017.
- Adversarial test: slightly alter attack samples and check whether the model stops detecting them.

## About

Created by Abdulrahman, IT student at the University of Jordan, as a first hands-on project in AI security.
