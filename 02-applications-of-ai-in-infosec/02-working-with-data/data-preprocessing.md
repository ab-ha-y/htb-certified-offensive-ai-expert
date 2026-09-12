# Data Preprocessing

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Applications of AI in InfoSec | Section: Data Preprocessing

---

## Table of Contents

1. [Why Raw Data Is Never Ready](#1-why-raw-data-is-never-ready)
2. [Cleaning: Duplicate Records](#2-cleaning-duplicate-records)
3. [Cleaning: Missing Values](#3-cleaning-missing-values)
4. [Imputing Missing Values](#4-imputing-missing-values)
5. [Encoding Categorical Variables (Preview)](#5-encoding-categorical-variables-preview)
6. [Handling Skewed Distributions](#6-handling-skewed-distributions)
7. [Handling Imbalanced Classes](#7-handling-imbalanced-classes)
8. [Worked Example: Cleaning a Network Log](#8-worked-example-cleaning-a-network-log)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. Why Raw Data Is Never Ready

### The Analogy

Imagine you are handed a stack of witness statements to build a case. Some statements are duplicated because two officers filed the same report. Some are missing a field because the witness refused to answer. Some use "yes"/"no" while others use "1"/"0" for the same question. If you feed this stack directly into your analysis without first standardizing it, your conclusions will be wrong -- not because your reasoning was bad, but because the raw material was inconsistent.

Machine learning models are far less forgiving than a human analyst. They cannot silently "figure out" that a missing value should be ignored, or that "TCP" and "tcp" mean the same protocol. Every inconsistency either crashes the code or, worse, gets learned as if it were meaningful. **Data preprocessing** is the set of steps that turns messy, real-world data into a form a model can safely learn from.

```
    THE PREPROCESSING STAGE OF THE ML PIPELINE
    ============================================

    Raw Data
      |
      v
    +------------------+     +-------------------+     +--------------------+
    | Remove           | --> | Handle Missing    | --> | Encode Categorical |
    | Duplicates        |     | Values (impute/   |     | Variables           |
    |                    |     | drop)              |     |                     |
    +------------------+     +-------------------+     +--------------------+
                                                                   |
                                                                   v
                                                  +--------------------------+
                                                  | Fix Skew / Class Balance |
                                                  +--------------------------+
                                                                   |
                                                                   v
                                                        Model-ready dataset
```

Skipping or rushing this stage is one of the most common reasons a model performs poorly, or worse, performs *deceptively well* in testing and then fails in the real world.

---

## 2. Cleaning: Duplicate Records

### Why Duplicates Are a Problem

If the same sample appears multiple times in your dataset, the model effectively "hears" that example more than once, giving it disproportionate influence over what the model considers normal. Worse, if a duplicate row ends up split across both your training set and test set, your test results become an illusion -- the model is not generalizing, it is simply remembering something it already memorized during training.

### Detecting and Removing Duplicates

```python
import pandas as pd

df = pd.read_csv("network_flows.csv")

# How many exact duplicate rows are there?
print("Duplicate rows:", df.duplicated().sum())

# Look at a few of them
print(df[df.duplicated()].head())

# Drop them, keeping the first occurrence
df = df.drop_duplicates(keep="first")

print("Shape after dropping duplicates:", df.shape)
```

### Duplicates Based on a Subset of Columns

Sometimes two rows are not byte-for-byte identical, but are still effectively the same event (e.g., identical source IP, destination IP, and timestamp, but a slightly different byte count due to logging noise).

```python
# Consider rows duplicates if these three columns match
df = df.drop_duplicates(subset=["src_ip", "dst_ip", "timestamp"], keep="first")
```

| Approach | When to Use |
|----------|-------------|
| `df.duplicated()` (all columns) | Exact copy-paste duplicates, e.g., a log line ingested twice by mistake |
| `df.duplicated(subset=[...])` | Logical duplicates that differ slightly in unimportant columns |
| Keep `first` vs. `last` | Usually `first` (keep the earliest record), unless later records are known to be corrections |

> **Security note**: In network and log data, exact duplicates are often themselves an *artifact* worth investigating -- e.g., a monitoring agent that retransmits the same event due to a bug, or an attacker deliberately flooding logs with copies to bury a real event ("log flooding" as an evasion technique). Cleaning duplicates for modeling purposes does not mean ignoring why they occurred operationally.

---

## 3. Cleaning: Missing Values

### Detecting Missing Values

```python
print(df.isnull().sum())
```

```
duration              0
protocol_type         0
src_bytes             0
dst_bytes             0
sender_domain_age  120
label                  0
dtype: int64
```

### Why Data Is Missing

| Reason | Example |
|--------|---------|
| **Field simply not collected** | An older log format did not record a "domain age" field |
| **Sensor/parser failure** | A malformed packet caused a field extraction to fail |
| **Legitimately not applicable** | "Attachment size" is missing because the email had no attachment |
| **Deliberately withheld or corrupted** | An attacker stripped metadata to evade a feature-based detector |

Knowing *why* data is missing matters, because it changes the right fix. If "attachment size" is missing because there was no attachment, filling it with 0 is exactly correct. If "domain age" is missing because of a sensor bug, filling it with 0 would be actively misleading (0 usually means "brand new domain," a strong phishing signal) -- a smarter imputation strategy is needed (Section 4).

### Options for Handling Missing Data

```
    HANDLING MISSING DATA
    =======================

    Missing values found
              |
              v
    Is the missing rate very low (e.g., <1%) and
    are the rows otherwise not special?
              |
      +-------+-------+
      | YES           | NO
      v               v
    Drop rows      Impute (fill in a reasonable
    (dropna)       estimate) -- see Section 4
```

```python
# Option A: Drop rows with ANY missing value
df_dropped = df.dropna()

# Option B: Drop rows missing a SPECIFIC critical column
df_dropped = df.dropna(subset=["label"])   # never guess a missing label!

# Option C: Drop a column entirely if it is mostly missing
print(df.isnull().mean())   # fraction missing per column
df = df.drop(columns=["mostly_empty_column"])
```

**Rule of thumb**: Never impute the **label** column. If you do not know the true answer for a training example, that example cannot teach the model anything reliable -- drop it instead.

---

## 4. Imputing Missing Values

**Imputation** means filling in missing values with a reasonable estimate rather than discarding the row entirely. This preserves more of your dataset, which matters a lot when data is expensive to collect (as most labeled security data is).

### Strategies

| Strategy | How It Works | Best For |
|----------|--------------|----------|
| **Mean imputation** | Fill with the column's average value | Numeric features with a roughly symmetric (non-skewed) distribution |
| **Median imputation** | Fill with the column's middle value | Numeric features that are skewed or have outliers (median is more robust than mean) |
| **Mode imputation** | Fill with the most frequent value | Categorical features |
| **Constant fill** | Fill with a fixed value (e.g., 0, "unknown") | When missing itself has a specific, known meaning |
| **Model-based imputation** | Predict the missing value using the other features (e.g., KNN imputation) | When missingness is complex and other features are strongly predictive of the missing one |

### Using scikit-learn's `SimpleImputer`

```python
from sklearn.impute import SimpleImputer
import pandas as pd

df = pd.read_csv("phishing_emails.csv")

# Numeric imputation: fill sender_domain_age with the median
num_imputer = SimpleImputer(strategy="median")
df["sender_domain_age"] = num_imputer.fit_transform(df[["sender_domain_age"]])

# Categorical imputation: fill a missing "tld" (top-level domain) with the mode
cat_imputer = SimpleImputer(strategy="most_frequent")
df["tld"] = cat_imputer.fit_transform(df[["tld"]]).ravel()

print(df.isnull().sum())   # should now show 0 for both columns
```

### A Worked Comparison

Suppose `sender_domain_age` (in days) has these values, with two missing entries marked `NaN`:

```
[5, 12, NaN, 4000, 3800, 9, NaN, 4200]
```

```
    Mean   = (5+12+4000+3800+9+4200) / 6  = 2004.3
    Median = sorted -> [4, 5, 9, 12, 3800, 4000, 4200]... 
             (using the 6 known values: [5,9,12,3800,4000,4200])
             sorted: [5, 9, 12, 3800, 4000, 4200]
             median = (12 + 3800) / 2 = 1906.0
```

Even the median is dragged high here because this feature is naturally **bimodal** (a cluster of very young domains used for phishing, and a cluster of very old, established domains) -- neither mean nor median imputation is perfect. This is a case where a domain-aware constant (e.g., filling with a value near the "young domain" cluster, since missing metadata often correlates with sketchy, newly-registered infrastructure) or a model-based imputer would do better than a blind statistical fill. Always sanity-check what imputation strategy makes sense for the *meaning* of the feature, not just its numbers.

### KNN Imputation (Model-Based)

```python
from sklearn.impute import KNNImputer

# Fill missing values using the average of the 5 most similar rows
knn_imputer = KNNImputer(n_neighbors=5)
df_numeric = df.select_dtypes(include="number")
df_imputed = pd.DataFrame(
    knn_imputer.fit_transform(df_numeric),
    columns=df_numeric.columns
)
```

---

## 5. Encoding Categorical Variables (Preview)

Models operate on numbers, not text categories like `"tcp"`, `"udp"`, or `"icmp"`. **Encoding** converts categorical values into numeric form. This topic gets a full deep dive (including one-hot encoding) in the next file, Data Transformation -- here is the short version so preprocessing feels complete on its own.

```python
import pandas as pd

df = pd.DataFrame({"protocol": ["tcp", "udp", "tcp", "icmp", "udp"]})

# Quick one-hot encoding with pandas
encoded = pd.get_dummies(df["protocol"], prefix="protocol")
print(encoded)
```

```
   protocol_icmp  protocol_tcp  protocol_udp
0              0             1             0
1              0             0             1
2              0             1             0
3              1             0             0
4              0             0             1
```

> Full coverage of *why* one-hot encoding works this way, its trade-offs, and alternative encodings (label encoding, ordinal encoding, target encoding) is in the **Data Transformation** file.

---

## 6. Handling Skewed Distributions

A **skewed** distribution is one that is not symmetric -- it has a long tail stretching toward either very large (right/positive skew) or very small (left/negative skew) values. Byte counts, file sizes, and transaction amounts are almost always right-skewed in security data: most events are small/normal, and a rare few are enormous.

```
    RIGHT-SKEWED DISTRIBUTION
    ===========================

    Count
      |
    ##|
    ##|##
    ##|####
    ##|######
    ##|########......................
     0+---+---+---+---+---+---+---+---+
        small values          long tail of
        (most of the data)    large outliers
```

### Why Skew Is a Problem

Many algorithms (linear models, distance-based methods like KNN, and neural networks trained with standard optimizers) implicitly assume features are on a comparable, roughly symmetric scale. A heavily skewed feature can:

- Dominate distance calculations simply because its raw numbers are much larger than other features.
- Cause a model to be overly influenced by rare extreme values instead of the typical pattern.
- Slow down or destabilize training for gradient-based models.

### Fixing Skew

| Technique | Formula / Method | Effect |
|-----------|--------------------|--------|
| **Log transform** | `log(x + 1)` | Compresses large values much more than small ones, pulling in the long tail |
| **Square root transform** | `sqrt(x)` | A gentler compression than log, for moderate skew |
| **Box-Cox transform** | A family of power transforms with a tuned parameter | Automatically finds a good transform to make data closer to normal (requires positive values) |
| **Clipping / capping** | Cap values above a chosen percentile (e.g., 99th) | Limits the influence of extreme outliers without discarding the samples |
| **Scaling (after transform)** | `StandardScaler`, `MinMaxScaler` | Puts features on a comparable numeric range once skew is addressed |

### Applying a Log Transform

```python
import numpy as np
import pandas as pd

df["src_bytes_log"] = np.log1p(df["src_bytes"])   # log1p = log(x + 1), safe for x = 0

print(df[["src_bytes", "src_bytes_log"]].describe())
```

```
       src_bytes      src_bytes_log
count  10000.00       10000.00
mean     452.12            4.21
std     3210.55            1.42
min        0.00            0.00
max   180000.00           12.10
```

Notice how the log-transformed column has a far smaller standard deviation relative to its range -- the extreme values no longer dominate the scale of the feature.

```
    BEFORE LOG TRANSFORM              AFTER LOG TRANSFORM
    =======================          =======================
    ##|                                    ####
    ##|##                                ########
    ##|####                            ##############
    ##|######                        ##################
    ##|########.....                ######################
     0+-----------------+             0+-------------------+
        0        180K                   0          12
     (long thin tail)                (roughly symmetric, "bell"-like)
```

---

## 7. Handling Imbalanced Classes

Security datasets are almost always **imbalanced**: attacks, malware, and phishing emails are rare events compared to the flood of normal, benign traffic. A dataset that is 99% benign and 1% malicious will let a lazy model achieve 99% accuracy by predicting "benign" for everything -- while being completely useless.

### Techniques for Imbalanced Data

| Technique | How It Works | Trade-off |
|-----------|---------------|-----------|
| **Random undersampling** | Randomly remove samples from the majority class until classes are balanced | Fast and simple, but throws away potentially useful data |
| **Random oversampling** | Randomly duplicate samples from the minority class | Keeps all data, but duplicated samples add no new information and can encourage overfitting to those exact points |
| **SMOTE** (Synthetic Minority Oversampling Technique) | Generates new, synthetic minority-class samples by interpolating between existing minority samples | Adds genuinely new (if artificial) data points, generally better than plain duplication |
| **Class weighting** | Tell the algorithm to penalize mistakes on the minority class more heavily during training (no resampling needed) | Very easy to apply in scikit-learn (`class_weight="balanced"`), no risk of overfitting to duplicated points |
| **Choosing the right metric** | Use precision/recall/F1 (and inspect the confusion matrix) instead of trusting accuracy | Does not change the data, but changes how you judge model quality |

```
    ORIGINAL IMBALANCED DATA          AFTER SMOTE OVERSAMPLING
    ============================      ============================
    Benign:    ################       Benign:    ################
    Malicious: #                      Malicious: ################
              (99% vs 1%)                       (roughly 50/50, with
                                                  synthetic malicious
                                                  samples added)
```

### Applying Class Weighting in scikit-learn

```python
from sklearn.ensemble import RandomForestClassifier

# The model automatically weights the minority class more heavily
# based on how rare it is in y_train
clf = RandomForestClassifier(class_weight="balanced", random_state=42)
clf.fit(X_train, y_train)
```

### Applying SMOTE (via the `imbalanced-learn` library)

```python
from imblearn.over_sampling import SMOTE

smote = SMOTE(random_state=42)
X_resampled, y_resampled = smote.fit_resample(X_train, y_train)

print("Before:", y_train.value_counts().to_dict())
print("After: ", pd.Series(y_resampled).value_counts().to_dict())
```

```
Before: {'benign': 9900, 'malicious': 100}
After:  {'benign': 9900, 'malicious': 9900}
```

> **Important**: Always resample (SMOTE / undersampling / oversampling) only on the **training set**, after the train/test split (covered fully in the next file). If you resample before splitting, synthetic or duplicated copies of the same underlying event can leak into both the training and test sets, artificially inflating your reported performance.

---

## 8. Worked Example: Cleaning a Network Log

Bringing every technique in this file together on one realistic dataset.

```python
import pandas as pd
import numpy as np
from sklearn.impute import SimpleImputer

# 1. Load
df = pd.read_csv("network_flows_raw.csv")
print("Initial shape:", df.shape)

# 2. Remove exact duplicate rows
before = df.shape[0]
df = df.drop_duplicates()
print(f"Removed {before - df.shape[0]} duplicate rows")

# 3. Drop rows with a missing label -- never guess the answer key
df = df.dropna(subset=["label"])

# 4. Impute missing numeric feature with the median (robust to skew/outliers)
imputer = SimpleImputer(strategy="median")
df["duration"] = imputer.fit_transform(df[["duration"]])

# 5. Impute missing categorical feature with the most frequent value
df["protocol_type"] = df["protocol_type"].fillna(df["protocol_type"].mode()[0])

# 6. Fix skew in byte-count features with a log transform
df["src_bytes_log"] = np.log1p(df["src_bytes"])
df["dst_bytes_log"] = np.log1p(df["dst_bytes"])

# 7. Check class balance
print(df["label"].value_counts(normalize=True))

# 8. If imbalanced, note it for later: use class_weight="balanced" during
#    training, or apply SMOTE on the training split only.

print("Final shape:", df.shape)
print(df.isnull().sum())   # confirm nothing is missing anymore
```

Sample output:

```
Initial shape: (12000, 8)
Removed 340 duplicate rows
label
normal    0.83
attack    0.17
Name: label, dtype: float64
Final shape: (11648, 10)
duration          0
protocol_type      0
src_bytes           0
dst_bytes            0
label                0
src_bytes_log        0
dst_bytes_log         0
dtype: int64
```

The dataset went from 12,000 raw, messy rows to 11,648 clean rows with no missing values, log-transformed byte features, and a clearly documented 83/17 class imbalance to account for during model training and evaluation.

---

## 9. Key Takeaways

- **Preprocessing turns messy raw data into a form models can safely learn from** -- skipping it leads to crashes, silently wrong models, or misleadingly optimistic results.
- **Duplicates** inflate the influence of repeated events and can leak between train/test splits if not removed first.
- **Missing values** should never be guessed for the label column (drop those rows); for feature columns, choose between dropping and imputing based on how much data is missing and why.
- **Imputation strategies** (mean, median, mode, constant, model-based/KNN) each fit different situations -- median is generally safer than mean for skewed security data.
- **Skewed distributions** (common for byte counts, file sizes, transaction amounts) are commonly fixed with log or Box-Cox transforms so models are not dominated by rare extreme values.
- **Imbalanced classes** are the norm in security data. Use class weighting, SMOTE, or careful metric selection (precision/recall/F1, not accuracy) rather than ignoring the imbalance.
- **Always resample training data only, after splitting**, to avoid leaking information from synthetic or duplicated samples into your test set.

*Next up: Data Transformation -- a deep dive into one-hot encoding and the strategies for splitting data into training, validation, and test sets.*
