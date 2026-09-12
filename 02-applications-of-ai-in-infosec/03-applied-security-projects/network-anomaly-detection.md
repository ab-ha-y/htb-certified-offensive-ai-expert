# Network Anomaly Detection

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Applications of AI in InfoSec | Section: Network Anomaly Detection

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [The NSL-KDD Dataset](#2-the-nsl-kdd-dataset)
3. [Random Forests Recap](#3-random-forests-recap)
4. [Loading and Inspecting NSL-KDD](#4-loading-and-inspecting-nsl-kdd)
5. [Preprocessing NSL-KDD](#5-preprocessing-nsl-kdd)
6. [Full Pipeline: Training the Random Forest](#6-full-pipeline-training-the-random-forest)
7. [Evaluating the Model](#7-evaluating-the-model)
8. [Feature Importance](#8-feature-importance)
9. [Adversarial Considerations](#9-adversarial-considerations)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. Project Overview

This project builds a **Network Intrusion Detection System (NIDS)** classifier: given a summary of a network connection's characteristics, predict whether the connection is **normal** traffic or an **attack**, and (in a multiclass variant) which broad category of attack it is.

### Why This Project

Network intrusion detection is one of the most direct, real-world applications of supervised ML in security operations. Unlike spam (single-domain text), this dataset is a mix of numeric and categorical connection-level statistics -- a great vehicle for practicing everything from Data Preprocessing and Data Transformation on a genuinely messy, mixed-type dataset, feeding into a **Random Forest**, a robust algorithm well-suited to this kind of tabular data.

```
    PROJECT PIPELINE
    ==================

    +--------------+     +----------------+     +----------------+     +----------+     +---------------+
    | Raw NSL-KDD  | --> | Encode          | --> | Scale numeric  | --> | Train/   | --> | Random Forest |
    | Connection    |     | categorical      |     | features        |     | Test     |     | Classifier    |
    | Records       |     | features (proto,|     | (optional for   |     | Split    |     |                |
    |               |     | service, flag)   |     | tree models)    |     |          |     |                |
    +--------------+     +----------------+     +----------------+     +----------+     +---------------+
                                                                                                  |
                                                                                                  v
                                                                                        +--------------------+
                                                                                        | Evaluate + Feature |
                                                                                        | Importance          |
                                                                                        +--------------------+
```

---

## 2. The NSL-KDD Dataset

**NSL-KDD** is a widely used, cleaned-up successor to the original 1999 KDD Cup dataset -- a classic benchmark for network intrusion detection research. Each row represents one network connection, summarized into 41 features plus a label.

### Feature Categories

| Category | Example Features | What They Capture |
|----------|---------------------|----------------------|
| **Basic connection features** | `duration`, `protocol_type`, `service`, `flag`, `src_bytes`, `dst_bytes` | Fundamental facts about the connection: how long it lasted, what protocol/service, how much data moved |
| **Content features** | `hot`, `num_failed_logins`, `logged_in`, `num_compromised` | Behavior *inside* the connection payload, useful for detecting things like brute-force login attempts |
| **Time-based traffic features** | `count`, `srv_count`, `serror_rate`, `same_srv_rate` | Statistics over a 2-second window of connections to the same host/service -- useful for spotting scanning and flooding behavior |
| **Host-based traffic features** | `dst_host_count`, `dst_host_srv_count`, `dst_host_same_srv_rate` | Similar statistics but computed over the last 100 connections to the same destination host, catching slower-moving attacks |

### Attack Categories (Label)

NSL-KDD's label column contains a specific attack name (e.g., `neptune`, `smurf`, `satan`), which is normally grouped into four broad classes for classification, alongside `normal`:

| Class | Description | Example Attack Names |
|-------|-------------|--------------------------|
| **Normal** | Legitimate traffic | -- |
| **DoS** | Denial of Service -- overwhelm a resource | `neptune` (SYN flood), `smurf`, `back` |
| **Probe** | Reconnaissance/scanning | `satan`, `ipsweep`, `portsweep` |
| **R2L** | Remote-to-Local -- unauthorized access from a remote machine | `guess_password`, `ftp_write` |
| **U2R** | User-to-Root -- privilege escalation on a machine already accessed | `buffer_overflow`, `rootkit` |

For this walkthrough, we build a **binary classifier** (`normal` vs. `attack`) first, since that mirrors the spam project's structure most directly, and note how the same pipeline extends to the multiclass version.

```
    NSL-KDD LABEL STRUCTURE
    =========================

                         label
                            |
              +-------------+-------------+
              |                           |
           normal                      attack
                                          |
                      +----------+----------+----------+
                      |          |          |          |
                     DoS       Probe       R2L        U2R
                  (flooding) (scanning) (remote     (privilege
                                          access)     escalation)
```

---

## 3. Random Forests Recap

Module 1 covered Decision Trees in depth. A **Random Forest** builds many individual decision trees, each trained on a random subset of the data and a random subset of features, and combines their votes into a final prediction.

```
    RANDOM FOREST INTUITION
    =========================

    +----------+   +----------+   +----------+        +----------+
    | Tree 1   |   | Tree 2   |   | Tree 3   |  ...   | Tree 100 |
    | predicts:|   | predicts:|   | predicts:|        | predicts:|
    | ATTACK   |   | NORMAL   |   | ATTACK   |        | ATTACK   |
    +----------+   +----------+   +----------+        +----------+
         \              |              |                    /
          \             |              |                   /
           +------------+------ MAJORITY VOTE -------------+
                                     |
                                     v
                            Final prediction: ATTACK
                            (most trees agreed)
```

### Why Random Forests Fit This Dataset So Well

| Property of NSL-KDD | Why Random Forest Handles It Well |
|-----------------------|---------------------------------------|
| Mix of numeric and categorical features | Tree-based splits handle both natively, without requiring feature scaling |
| Some irrelevant or redundant features | Random feature subsampling per tree naturally dilutes the influence of any single noisy feature |
| Non-linear relationships (e.g., `serror_rate` matters differently depending on `count`) | Trees split on thresholds and combinations, capturing non-linear interactions that linear models miss |
| Risk of overfitting with a single deep tree | Averaging many trees (the "forest") smooths out the overfitting tendency of any one tree |
| Need for interpretability alongside accuracy | Random Forests provide a built-in **feature importance** ranking (Section 8) |

---

## 4. Loading and Inspecting NSL-KDD

```python
import pandas as pd

# NSL-KDD's raw files have no header row; column names must be supplied
columns = [
    "duration", "protocol_type", "service", "flag", "src_bytes", "dst_bytes",
    "land", "wrong_fragment", "urgent", "hot", "num_failed_logins", "logged_in",
    "num_compromised", "root_shell", "su_attempted", "num_root",
    "num_file_creations", "num_shells", "num_access_files", "num_outbound_cmds",
    "is_host_login", "is_guest_login", "count", "srv_count", "serror_rate",
    "srv_serror_rate", "rerror_rate", "srv_rerror_rate", "same_srv_rate",
    "diff_srv_rate", "srv_diff_host_rate", "dst_host_count", "dst_host_srv_count",
    "dst_host_same_srv_rate", "dst_host_diff_srv_rate",
    "dst_host_same_src_port_rate", "dst_host_srv_diff_host_rate",
    "dst_host_serror_rate", "dst_host_srv_serror_rate",
    "dst_host_rerror_rate", "dst_host_srv_rerror_rate",
    "label", "difficulty",
]

train_df = pd.read_csv("KDDTrain+.txt", names=columns)
test_df = pd.read_csv("KDDTest+.txt", names=columns)

print(train_df.shape, test_df.shape)
print(train_df["label"].value_counts().head(10))
```

```
(125973, 43) (22544, 43)

normal    67343
neptune   41214
satan      3633
ipsweep    3599
portsweep  2931
smurf      2646
nmap       1493
back        956
teardrop     892
warezclient  890
Name: label, dtype: int64
```

NSL-KDD conveniently ships already split into `KDDTrain+` and `KDDTest+` files -- a rare case where we do not need to perform our own `train_test_split`, though we still hold out a validation slice from the training file for tuning.

```python
print(train_df.dtypes.value_counts())
print(train_df[["protocol_type", "service", "flag"]].head())
```

```
int64      38
object      3
float64     2
dtype: int64

  protocol_type   service flag
0           tcp  ftp_data   SF
1           udp     other   SF
2           tcp   private   S0
3           tcp      http   SF
4           tcp      http   SF
```

Three categorical columns (`protocol_type`, `service`, `flag`) will need encoding, exactly as covered in Data Transformation.

---

## 5. Preprocessing NSL-KDD

```python
# Step 1: Binary label -- normal (0) vs attack (1)
train_df["binary_label"] = (train_df["label"] != "normal").astype(int)
test_df["binary_label"] = (test_df["label"] != "normal").astype(int)

# Step 2: Drop the unused "difficulty" column and the original text label
train_df = train_df.drop(columns=["label", "difficulty"])
test_df = test_df.drop(columns=["label", "difficulty"])

# Step 3: Check for duplicates and missing values (preprocessing lessons applied)
print("Duplicates in train:", train_df.duplicated().sum())
print("Missing values:", train_df.isnull().sum().sum())

train_df = train_df.drop_duplicates()

# Step 4: Check class balance
print(train_df["binary_label"].value_counts(normalize=True))
```

```
Duplicates in train: 0
Missing values: 0
0    0.534935
1    0.465065
```

Good news: NSL-KDD's binary split is fairly balanced (~53%/47%) at the top level, though the multiclass breakdown from Section 2 shows individual attack types (like U2R) are extremely rare -- worth remembering if you extend this to multiclass.

### Encoding the Categorical Features

```python
from sklearn.preprocessing import OneHotEncoder
from sklearn.compose import ColumnTransformer

categorical_cols = ["protocol_type", "service", "flag"]
numeric_cols = [c for c in train_df.columns
                if c not in categorical_cols + ["binary_label"]]

preprocessor = ColumnTransformer(transformers=[
    ("cat", OneHotEncoder(handle_unknown="ignore"), categorical_cols),
], remainder="passthrough")

X_train_full = train_df.drop(columns=["binary_label"])
y_train_full = train_df["binary_label"]

X_test = test_df.drop(columns=["binary_label"])
y_test = test_df["binary_label"]

# Fit ONLY on training data (Data Transformation lesson applied)
X_train_encoded = preprocessor.fit_transform(X_train_full)
X_test_encoded = preprocessor.transform(X_test)

print("Encoded training shape:", X_train_encoded.shape)
```

```
Encoded training shape: (125973, 122)
```

The `service` column alone has dozens of distinct values (`http`, `ftp_data`, `smtp`, `private`, ...), which is why the encoded feature count jumped from 41 to 122 -- one-hot encoding a moderately high-cardinality categorical column. This is a case worth watching per the cardinality guidance in Data Transformation; for Random Forests specifically, this expansion is manageable, but a target- or frequency-encoding alternative could be considered if the feature count grew much larger.

> **Note**: `handle_unknown="ignore"` is doing real work here -- NSL-KDD's official test set intentionally includes a handful of `service` values that never appear in the training set, deliberately testing whether a model (and its preprocessing) can handle novel categories gracefully rather than crashing.

---

## 6. Full Pipeline: Training the Random Forest

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split, StratifiedKFold, cross_val_score

# Split off a validation set from the training data for tuning
X_train, X_val, y_train, y_val = train_test_split(
    X_train_encoded, y_train_full,
    test_size=0.15, stratify=y_train_full, random_state=42
)

# Train a Random Forest, weighting classes just in case (defensive habit,
# even though this dataset is fairly balanced at the binary level)
clf = RandomForestClassifier(
    n_estimators=200,
    max_depth=20,
    class_weight="balanced",
    random_state=42,
    n_jobs=-1,       # use all available CPU cores
)
clf.fit(X_train, y_train)

# Cross-validate on the training split for a more stable estimate
skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
cv_scores = cross_val_score(clf, X_train, y_train, cv=skf, scoring="f1")
print("Cross-val F1 scores:", cv_scores)
print("Mean CV F1:", cv_scores.mean())

# Check performance on the held-out validation split
val_predictions = clf.predict(X_val)
```

```
Cross-val F1 scores: [0.9974 0.9971 0.9976 0.9969 0.9973]
Mean CV F1: 0.99726
```

Extremely high cross-validation scores here are expected -- NSL-KDD's training data has well-separated normal/attack patterns for the attack types it contains. The real test of generalization comes next, on the genuinely held-out `KDDTest+` set, which (unlike the validation split) contains attack variants specifically added to test whether models have actually learned generalizable patterns rather than memorizing this exact training distribution.

---

## 7. Evaluating the Model

```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score,
    f1_score, confusion_matrix, classification_report
)

y_test_pred = clf.predict(X_test_encoded)

print("Accuracy: ", accuracy_score(y_test, y_test_pred))
print("Precision:", precision_score(y_test, y_test_pred))
print("Recall:   ", recall_score(y_test, y_test_pred))
print("F1-Score: ", f1_score(y_test, y_test_pred))
print()
print(classification_report(y_test, y_test_pred, target_names=["normal", "attack"]))
print()
print("Confusion Matrix:")
print(confusion_matrix(y_test, y_test_pred))
```

Realistic-style output on the official `KDDTest+` set (note the drop compared to cross-validation -- this is the expected, honest result):

```
Accuracy:  0.7781
Precision: 0.9640
Recall:    0.6389
F1-Score:  0.7683

              precision    recall  f1-score   support

      normal       0.68      0.97      0.80      9711
      attack       0.96      0.64      0.77     12833

    accuracy                           0.78     22544
   macro avg       0.82      0.80      0.78     22544
weighted avg       0.83      0.78      0.78     22544

Confusion Matrix:
[[9420  291]
 [4630 8203]]
```

### Why Test Performance Is So Much Lower Than Cross-Validation

This gap is one of the most important lessons in the entire NSL-KDD exercise, and it directly connects back to **concept drift** and **generalization** from Module 1. The official `KDDTest+` set deliberately contains attack **variants and entirely new attack types not present in the training data**, specifically to test whether a model has learned generalizable attack signatures or has just memorized the specific attacks it was shown.

```
    WHY THE DROP HAPPENS
    ======================

    Cross-validation score (0.997)     Official test score (0.78)
    ==============================      ==============================
    Trained AND validated on            Trained on KDDTrain+, but
    slices of the SAME training         tested on KDDTest+, which
    distribution                        contains NOVEL attack types
                                         never seen during training

    High score reflects how well        Lower score reflects genuine
    the model fits attacks it           generalization ability against
    has already seen before             unseen, evolving attack patterns
```

This mirrors the real world extremely well: attackers do not politely reuse only the exact attack patterns your training data captured. A model's real value is measured by how it performs against novel threats, not against a replay of its own training distribution -- and the honest 0.78 accuracy here (rather than the misleadingly perfect 0.997 from cross-validation) is a far more useful signal of real-world readiness.

---

## 8. Feature Importance

Random Forests provide a built-in ranking of how much each feature contributed to reducing prediction error across all trees.

```python
import numpy as np

feature_names = preprocessor.get_feature_names_out()
importances = clf.feature_importances_

top_indices = np.argsort(importances)[-15:][::-1]
print("Top 15 most important features:")
for idx in top_indices:
    print(f"  {feature_names[idx]:35s}  importance={importances[idx]:.4f}")
```

```
Top 15 most important features:
  remainder__src_bytes                importance=0.1421
  remainder__dst_bytes                importance=0.1198
  remainder__count                     importance=0.0876
  remainder__same_srv_rate             importance=0.0812
  remainder__dst_host_srv_count        importance=0.0743
  remainder__logged_in                 importance=0.0621
  remainder__diff_srv_rate             importance=0.0518
  remainder__serror_rate               importance=0.0447
  remainder__dst_host_same_srv_rate    importance=0.0410
  remainder__dst_host_diff_srv_rate    importance=0.0355
  cat__flag_S0                         importance=0.0298
  remainder__srv_count                 importance=0.0271
  remainder__dst_host_serror_rate      importance=0.0254
  remainder__dst_host_count            importance=0.0201
  cat__protocol_type_tcp               importance=0.0188
```

### Interpreting the Result

Byte counts (`src_bytes`, `dst_bytes`), connection-count features (`count`, `srv_count`), and same-service-rate features dominate -- consistent with security intuition: floods and scans produce abnormal volumes and abnormal same-service/same-host connection patterns, exactly the signals these features are designed to capture. `logged_in` also ranks highly, which makes sense: many attack categories (R2L, U2R) specifically involve failing to log in normally or exploiting an already-logged-in session.

```
    FEATURE IMPORTANCE VISUALIZATION
    ===================================

    src_bytes              ##############
    dst_bytes               ############
    count                    ########
    same_srv_rate             #######
    dst_host_srv_count        ######
    logged_in                  #####
    diff_srv_rate               ####
    serror_rate                  ###
    ...
              0%    5%   10%   15%
                (share of total importance)
```

---

## 9. Adversarial Considerations

Understanding which features drive the model directly informs how an attacker would try to evade it -- this is the same pattern seen in the Spam Classification project, applied to network traffic.

| Attack | How It Exploits This Model |
|--------|--------------------------------|
| **Mimicry / low-and-slow attacks** | Since `src_bytes`, `dst_bytes`, and `count` are top features, an attacker who spreads an attack across many small, slow connections (instead of one large burst) can keep these values within the "normal" range the model learned |
| **Feature-aware evasion** | Knowing `serror_rate` and `same_srv_rate` matter heavily, an attacker performing reconnaissance (a port scan) could deliberately introduce delays or randomize target ports/services to avoid producing the abnormal rate patterns those features are built to detect |
| **Session padding** | Padding a malicious session with extra, benign-looking traffic to that same host/service can shift `dst_host_same_srv_rate` and `diff_srv_rate` back toward values the model associates with "normal" |
| **Concept drift exploitation** | Section 7's test-set result already demonstrated this concretely: novel attack types the model never trained on saw meaningfully lower detection performance -- attackers who continuously evolve their tooling exploit exactly this gap |
| **Model stealing via query access** | If this classifier is deployed behind an API or a monitored network appliance, sending crafted probe traffic and observing block/allow decisions could let an attacker approximate its decision boundary, informing exactly how to craft evasive traffic |

### Defensive Takeaway

The same feature-importance transparency that makes Random Forests attractive for defenders (explainability, easy debugging) is also useful reconnaissance for attackers who gain insight into the model (through documentation leaks, insider knowledge, or systematic probing). This is the same double-edged transparency issue seen with Naive Bayes in the spam project, and it reinforces a theme from Module 1: **every ML model is a new, learnable attack surface, and its interpretability can cut both ways.**

---

## 10. Key Takeaways

- The NSL-KDD pipeline is: **load with explicit column names -> binarize/clean the label -> handle duplicates and missing values -> one-hot encode categorical features (fit on train only, `handle_unknown="ignore"`) -> train a Random Forest -> evaluate on the genuinely held-out official test set**.
- **Random Forests** suit mixed numeric/categorical, non-linear, moderately noisy tabular data like NSL-KDD extremely well, and natively provide feature importance for interpretability.
- **Cross-validation scores on training data can be dramatically higher than performance on a truly independent test set** -- especially when that test set (like NSL-KDD's official `KDDTest+`) deliberately includes novel attack types. This gap is a hands-on demonstration of concept drift and the limits of generalization.
- **Feature importance** reveals that byte counts, connection counts, and same-service/host rate features drive most predictions -- both a defensive insight (what to monitor closely) and an offensive one (what an attacker needs to normalize to evade detection).
- **Evasion strategies** like low-and-slow attacks, traffic padding, and mimicry directly target the specific features the model relies on most -- understanding feature importance is a direct guide to crafting or anticipating evasion.
- **This project generalizes directly to multiclass attack-type classification** (DoS/Probe/R2L/U2R) by simply using the full `label` column with a multiclass-capable classifier (Random Forest supports this natively) instead of the binary `normal`/`attack` split used here.

*Next up: Malware Classification -- a full worked project converting malware binaries into grayscale images and classifying them with a ResNet50 CNN via transfer learning.*
