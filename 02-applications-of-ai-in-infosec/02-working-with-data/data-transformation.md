# Data Transformation

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Applications of AI in InfoSec | Section: Data Transformation

---

## Table of Contents

1. [What is Data Transformation?](#1-what-is-data-transformation)
2. [One-Hot Encoding In Depth](#2-one-hot-encoding-in-depth)
3. [Other Encoding Schemes](#3-other-encoding-schemes)
4. [Train / Validation / Test Splitting Strategies](#4-train--validation--test-splitting-strategies)
5. [Stratified Splitting](#5-stratified-splitting)
6. [Cross-Validation](#6-cross-validation)
7. [Worked Example: Encoding and Splitting a Phishing Dataset](#7-worked-example-encoding-and-splitting-a-phishing-dataset)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. What is Data Transformation?

**Data transformation** covers the steps that reshape already-cleaned data into the exact numeric structure a model needs -- converting categories into numbers, and dividing the dataset into the portions used for training, tuning, and final evaluation.

### The Analogy

Preprocessing (the previous file) is like washing and sorting your ingredients -- removing spoiled items, filling in a missing spice with a reasonable substitute. Data transformation is the next step: chopping everything into the exact shapes the recipe calls for, and portioning out what goes into tonight's dinner versus what gets saved to test the recipe again next week. Both matter, but they are different jobs.

```
    WHERE THIS FITS IN THE PIPELINE
    =================================

    Raw Data --> [Cleaning / Imputation]  --> [Encoding] --> [Splitting] --> Model Training
                  (previous file)              THIS FILE      THIS FILE
```

---

## 2. One-Hot Encoding In Depth

### The Problem It Solves

Suppose you have a `protocol_type` column with values `"tcp"`, `"udp"`, and `"icmp"`. A model needs numbers, so your first instinct might be to just assign each category a number:

```
tcp  -> 0
udp  -> 1
icmp -> 2
```

This seems reasonable, but it silently introduces a fake fact: it tells the model that `icmp` (2) is "twice as much" as `udp` (1), and that `udp` is "between" `tcp` and `icmp`. There is no real ordering or magnitude relationship between protocol types -- this numbering is arbitrary, but many algorithms (especially linear models and distance-based methods) will treat those numbers as if the ordering and spacing were meaningful.

### The Solution: One-Hot Encoding

**One-hot encoding** solves this by creating a separate binary (0/1) column for every possible category. Exactly one column is "hot" (set to 1) for each row -- hence the name.

```
    ONE-HOT ENCODING
    =================

    BEFORE:                         AFTER:
    +---------------+               +---------------+---------------+---------------+
    | protocol_type |               | protocol_tcp  | protocol_udp  | protocol_icmp |
    +---------------+               +---------------+---------------+---------------+
    | tcp           |    ------>    |       1       |       0       |       0       |
    | udp           |               |       0       |       1       |       0       |
    | icmp          |               |       0       |       0       |       1       |
    | tcp           |               |       1       |       0       |       0       |
    +---------------+               +---------------+---------------+---------------+

    No column implies order or magnitude. Each category gets an equal,
    independent "flag."
```

Now no category is treated as larger, smaller, or "between" any other -- they are just independent yes/no flags, exactly matching the true nature of the data.

### Doing It in pandas

```python
import pandas as pd

df = pd.DataFrame({
    "protocol_type": ["tcp", "udp", "icmp", "tcp", "udp"],
    "src_bytes": [500, 300, 64, 480, 310],
})

encoded = pd.get_dummies(df, columns=["protocol_type"])
print(encoded)
```

```
   src_bytes  protocol_type_icmp  protocol_type_tcp  protocol_type_udp
0        500                   0                  1                  0
1        300                   0                  0                  1
2         64                   1                  0                  0
3        480                   0                  1                  0
4        310                   0                  0                  1
```

### Doing It in scikit-learn (Fit/Transform Pattern)

The scikit-learn version is preferred in real pipelines because it can be `fit` on training data and consistently `transform` new/unseen data (including categories it must handle gracefully), and it plugs directly into a `Pipeline` object.

```python
from sklearn.preprocessing import OneHotEncoder
import numpy as np

encoder = OneHotEncoder(sparse_output=False, handle_unknown="ignore")

protocols = np.array([["tcp"], ["udp"], ["icmp"], ["tcp"]])
encoded = encoder.fit_transform(protocols)

print(encoder.categories_)     # [array(['icmp', 'tcp', 'udp'], dtype=object)]
print(encoded)
```

```
[array(['icmp', 'tcp', 'udp'], dtype=object)]
[[0. 1. 0.]
 [0. 0. 1.]
 [1. 0. 0.]
 [0. 1. 0.]]
```

`handle_unknown="ignore"` matters a lot in security applications: if your test data (or, worse, live production traffic) contains a protocol value never seen during training, the encoder will not crash -- it simply produces all-zero columns for that unseen category instead of raising an error.

### The Dummy Variable Trap

When a categorical feature is one-hot encoded, the resulting columns are **collinear**: if you know the values of all-but-one column, you can always deduce the last one (if `protocol_tcp=0` and `protocol_udp=0`, then `protocol_icmp` must be 1). This is called the **dummy variable trap**, and it can cause instability in some linear models (like plain linear/logistic regression using matrix inversion internally).

```python
# drop_first=True removes exactly one column per category to avoid
# the redundancy -- the dropped category is implied when all others are 0
encoded = pd.get_dummies(df, columns=["protocol_type"], drop_first=True)
```

| Situation | Recommendation |
|-----------|-------------------|
| Using tree-based models (Random Forest, Decision Trees) | Collinearity is not a real problem -- `drop_first` is optional |
| Using linear/logistic regression or neural networks with regularization | `drop_first=True` is a safer default |

### When One-Hot Encoding is a Bad Fit

One-hot encoding works best when the number of categories (**cardinality**) is small. If a categorical feature has thousands of unique values (e.g., raw IP addresses, or full domain names), one-hot encoding would create thousands of new columns -- mostly zeros, exploding memory usage and diluting the signal. In those high-cardinality cases, other encodings (Section 3) or feature hashing are used instead.

```
    LOW CARDINALITY                    HIGH CARDINALITY
    (good fit for one-hot)             (bad fit for one-hot)
    ========================          ==========================
    protocol_type: {tcp, udp, icmp}   src_ip: {198.51.100.4,
    3 categories -> 3 new columns      203.0.113.9, 192.0.2.55, ...}
                                       thousands of unique values ->
                                       thousands of mostly-zero columns
```

---

## 3. Other Encoding Schemes

| Encoding | How It Works | Best For | Watch Out For |
|----------|---------------|-----------|-----------------|
| **One-Hot Encoding** | One binary column per category | Low-cardinality, unordered categories (protocol type, file extension) | Explodes in size with high cardinality |
| **Label Encoding** | Each category gets a single integer (0, 1, 2, ...) | The *label* column itself for classification (order does not matter there since models treat classes as identities, not magnitudes), or tree-based models on features | Implies false ordering/magnitude if used on features fed to linear/distance-based models |
| **Ordinal Encoding** | Each category gets an integer that reflects a real, meaningful order | Genuinely ordered categories (e.g., severity: low=0, medium=1, high=2) | Only valid when a true order exists -- misusing it on unordered data reintroduces the exact problem one-hot encoding solves |
| **Target / Mean Encoding** | Replace each category with the average target value observed for that category in training data | High-cardinality categorical features (e.g., domain name, ASN) | Risk of **target leakage** if not computed carefully with cross-validation folds; can overfit on rare categories |
| **Frequency Encoding** | Replace each category with how often it appears in the training data | High-cardinality features where frequency itself is informative | Loses category identity -- two different, equally-rare categories become indistinguishable |
| **Feature Hashing** | Map categories to a fixed number of columns using a hash function | Extremely high-cardinality features (e.g., URLs, raw domains) where even target encoding is impractical | Hash collisions merge unrelated categories into the same column |

### Label Encoding Example

```python
from sklearn.preprocessing import LabelEncoder

le = LabelEncoder()
y = ["malicious", "benign", "benign", "malicious"]
y_encoded = le.fit_transform(y)

print(le.classes_)      # ['benign' 'malicious']
print(y_encoded)        # [1 0 0 1]
```

### Ordinal Encoding Example

```python
from sklearn.preprocessing import OrdinalEncoder

severity = [["low"], ["high"], ["medium"], ["low"]]
encoder = OrdinalEncoder(categories=[["low", "medium", "high"]])
encoded = encoder.fit_transform(severity)

print(encoded)   # [[0.] [2.] [1.] [0.]]
```

Note the explicit `categories=[["low", "medium", "high"]]` -- without specifying the true order, `OrdinalEncoder` would default to alphabetical order (`high, low, medium`), which would be wrong here.

### Decision Guide

```
    Is there a genuine, real-world ORDER to the categories?
    |
    +-- YES --> Ordinal Encoding (e.g., low/medium/high severity)
    |
    +-- NO  --> How many unique categories are there?
                |
                +-- FEW (roughly < 15-20) --> One-Hot Encoding
                |
                +-- MANY (hundreds/thousands) --> Target Encoding,
                                                   Frequency Encoding,
                                                   or Feature Hashing
```

---

## 4. Train / Validation / Test Splitting Strategies

### Why Split at All (Recap)

A model must be evaluated on data it has never seen, or you are only measuring memorization. This was introduced in Module 1; here we go deeper into *how* the split should actually be done in practice.

### The Basic Split

```python
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(
    X, y,
    test_size=0.2,      # 20% held out for testing
    random_state=42,    # makes the split reproducible
)
```

`random_state` fixes the randomness used to shuffle and split the data. Using the same `random_state` every time means you (and anyone reproducing your work) get the exact same split -- essential for fair comparisons between different models or preprocessing choices.

### Three-Way Split (Train / Validation / Test)

When you need to tune hyperparameters (e.g., how many trees in a Random Forest, or the learning rate of a neural network), you need a middle set to tune on -- the **validation set** -- so the **test set** stays completely untouched until the final evaluation.

```python
from sklearn.model_selection import train_test_split

# First split off the test set (e.g., 15%)
X_train_val, X_test, y_train_val, y_test = train_test_split(
    X, y, test_size=0.15, random_state=42
)

# Then split the remainder into train and validation (e.g., 70% / 15% of original)
X_train, X_val, y_train, y_val = train_test_split(
    X_train_val, y_train_val, test_size=0.176, random_state=42
    # 0.176 of the remaining 85% is approximately 15% of the original total
)

print(X_train.shape, X_val.shape, X_test.shape)
```

```
    +-------------------------------- Full Dataset ---------------------------------+
    |                                                                                |
    |----------------- Train (70%) -----------|-- Validation (15%) --|-- Test (15%)--|
    |     Learn model parameters here          |  Tune hyperparameters |  Final,      |
    |                                            |  here, compare       |  one-time    |
    |                                            |  model variants      |  evaluation  |
    +--------------------------------------------------------------------------------+
```

| Split Purpose | Touched During... | Rule |
|----------------|---------------------|------|
| **Training set** | Every training run | The model directly learns from this |
| **Validation set** | Every time you compare models or tune settings | Used repeatedly during development, but the model never trains directly on it |
| **Test set** | Exactly once, at the very end | If you look at test performance and then go back and change your model, it is no longer a fair, unbiased estimate |

> **Common mistake**: repeatedly checking test-set performance while iterating on your model design turns the test set into a de facto validation set -- you end up unconsciously overfitting your modeling *decisions* to it, even if the model itself never directly trains on those rows.

---

## 5. Stratified Splitting

### The Problem with a Plain Random Split on Imbalanced Data

If your dataset is 90% benign / 10% malicious, a plain random split might, just by chance, put a disproportionate share of the (already rare) malicious samples into one split -- leaving the other split with too few malicious examples to reliably train or evaluate on.

### The Fix: Stratification

**Stratified splitting** ensures that each split preserves the same class proportions as the original dataset.

```python
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(
    X, y,
    test_size=0.2,
    stratify=y,          # <-- preserves class ratio in both splits
    random_state=42,
)

print("Original ratio:  ", y.value_counts(normalize=True).to_dict())
print("Train ratio:      ", y_train.value_counts(normalize=True).to_dict())
print("Test ratio:        ", y_test.value_counts(normalize=True).to_dict())
```

```
Original ratio:   {'benign': 0.90, 'malicious': 0.10}
Train ratio:       {'benign': 0.90, 'malicious': 0.10}
Test ratio:         {'benign': 0.90, 'malicious': 0.10}
```

Without `stratify=y`, you might see something like `{'benign': 0.93, 'malicious': 0.07}` in the test set purely due to random chance -- a small but real distortion that makes your reported metrics less trustworthy, especially for a rare minority class where every sample counts.

```
    NON-STRATIFIED SPLIT                    STRATIFIED SPLIT
    (risk of skewed splits)                (preserves original ratio)
    =========================              ============================
    Full: 90% benign, 10% malicious        Full: 90% benign, 10% malicious
      |                                       |
      v (random luck)                          v (forced proportional split)
    Train: 88% / 12%                         Train: 90% / 10%
    Test:  95% / 5%   <-- distorted!         Test:  90% / 10%   <-- matches
```

**Rule of thumb**: For any classification task with class imbalance (which, in security, is nearly every task), always use `stratify=y` when splitting.

---

## 6. Cross-Validation

A single train/validation split gives you one estimate of performance, which can be a bit noisy -- you got a bit lucky or unlucky with which rows ended up where. **K-fold cross-validation** repeats the split-train-evaluate cycle multiple times on different slices of the data and averages the results, giving a far more reliable performance estimate.

### How K-Fold Works

```
    5-FOLD CROSS-VALIDATION
    =========================

    Full training data split into 5 equal folds: [1][2][3][4][5]

    Round 1: Train on [2][3][4][5]  ->  Validate on [1]  -> score_1
    Round 2: Train on [1][3][4][5]  ->  Validate on [2]  -> score_2
    Round 3: Train on [1][2][4][5]  ->  Validate on [3]  -> score_3
    Round 4: Train on [1][2][3][5]  ->  Validate on [4]  -> score_4
    Round 5: Train on [1][2][3][4]  ->  Validate on [5]  -> score_5

    Final performance estimate = average(score_1, ..., score_5)
    Also report the standard deviation -- a measure of how stable
    the model's performance is across different data slices.
```

Every row gets to be in the validation fold exactly once, and in the training set for the other four rounds -- squeezing the most reliable signal possible out of a limited amount of labeled data (very common in security, where labeled attack data is expensive to obtain).

### Using scikit-learn

```python
from sklearn.model_selection import cross_val_score, StratifiedKFold
from sklearn.ensemble import RandomForestClassifier

clf = RandomForestClassifier(n_estimators=100, random_state=42)

# Stratified K-Fold preserves class ratio within EVERY fold, not just
# the overall train/test split
skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)

scores = cross_val_score(clf, X_train, y_train, cv=skf, scoring="f1")

print("F1 scores per fold:", scores)
print("Mean F1:", scores.mean(), " Std:", scores.std())
```

```
F1 scores per fold: [0.91 0.89 0.93 0.90 0.88]
Mean F1: 0.902   Std: 0.018
```

A small standard deviation (0.018) across folds indicates the model's performance is fairly stable regardless of which slice of data it happens to see -- a good sign that the reported average is trustworthy.

---

## 7. Worked Example: Encoding and Splitting a Phishing Dataset

Bringing the whole file together on one dataset.

```python
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import OneHotEncoder, OrdinalEncoder
from sklearn.compose import ColumnTransformer

# 1. Load already-cleaned data (missing values handled in a prior step)
df = pd.read_csv("phishing_clean.csv")

# Columns:
#   url_length (numeric), num_links (numeric),
#   tld (categorical, low cardinality: com/net/org/xyz/...),
#   ssl_certificate_status (ordinal: none < self-signed < valid),
#   label (target: phishing / legitimate)

X = df.drop(columns=["label"])
y = df["label"]

# 2. Build a column-wise transformer:
#    - one-hot encode the low-cardinality "tld" column
#    - ordinal encode "ssl_certificate_status" (there IS a real order here)
#    - leave numeric columns unchanged
preprocessor = ColumnTransformer(transformers=[
    ("tld_ohe", OneHotEncoder(handle_unknown="ignore"), ["tld"]),
    ("ssl_ord", OrdinalEncoder(categories=[["none", "self-signed", "valid"]]),
        ["ssl_certificate_status"]),
], remainder="passthrough")   # numeric columns pass through unchanged

# 3. Split BEFORE fitting the encoder, so no information about the
#    test set's categories leaks into training
X_train_raw, X_test_raw, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

# 4. Fit the encoder on training data only, then transform both sets
X_train = preprocessor.fit_transform(X_train_raw)
X_test = preprocessor.transform(X_test_raw)

print("Train shape:", X_train.shape)
print("Test shape:", X_test.shape)
print("Train class balance:", y_train.value_counts(normalize=True).to_dict())
print("Test class balance:", y_test.value_counts(normalize=True).to_dict())
```

```
Train shape: (4000, 8)
Test shape: (1000, 8)
Train class balance: {'legitimate': 0.62, 'phishing': 0.38}
Test class balance: {'legitimate': 0.62, 'phishing': 0.38}
```

### Why the Order of Operations Matters

Notice that the split happens **before** fitting the encoders, and the encoders are fit only on `X_train_raw`. If you had fit the `OneHotEncoder` on the full dataset first, categories seen only in the test set could subtly influence encoding decisions (and, for encodings like target encoding, this would cause outright data leakage -- the test set's answers would leak backward into the features). Split first, fit second is the safe default for every transformation in this file.

---

## 8. Key Takeaways

- **Data transformation** turns cleaned data into the exact numeric shape a model requires: encoded categories and properly divided train/validation/test sets.
- **One-hot encoding** creates one binary column per category, avoiding the false sense of order or magnitude that plain integer labels would introduce. It works best for low-cardinality categorical features.
- **Other encodings** -- label, ordinal, target, frequency, and feature hashing -- exist for cases one-hot encoding does not fit well, especially high-cardinality features or genuinely ordered categories.
- **Always split data into train/validation/test**, and treat the test set as sacred: touch it only once, at the very end.
- **Stratified splitting** (`stratify=y`) preserves class proportions across splits, which matters enormously for the imbalanced datasets typical of security data.
- **K-fold cross-validation** gives a more reliable performance estimate than a single split by rotating which slice of data is held out, and also reveals how stable a model's performance is (via the standard deviation across folds).
- **Fit transformations only on training data**, then apply (transform) them to validation/test data -- fitting on the full dataset before splitting is a subtle but serious source of data leakage.

*Next up: Spam Classification -- a full worked project applying everything from Environment Setup through Data Transformation to build a real Naive Bayes email spam filter.*
