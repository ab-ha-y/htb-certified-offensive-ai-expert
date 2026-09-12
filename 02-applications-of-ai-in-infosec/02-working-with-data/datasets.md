# Datasets

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Applications of AI in InfoSec | Section: Datasets

---

## Table of Contents

1. [What is a Dataset?](#1-what-is-a-dataset)
2. [Anatomy of a Tabular Dataset](#2-anatomy-of-a-tabular-dataset)
3. [Loading Data with pandas](#3-loading-data-with-pandas)
4. [Inspecting a Dataset](#4-inspecting-a-dataset)
5. [Summary Statistics](#5-summary-statistics)
6. [Visualizing Distributions](#6-visualizing-distributions)
7. [Worked Example: A First Look at a Security Dataset](#7-worked-example-a-first-look-at-a-security-dataset)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. What is a Dataset?

A **dataset** is simply an organized collection of data used to train and evaluate a machine learning model.

### The Analogy

Think of a dataset as a spreadsheet a detective keeps of past cases. Each row is one case (one email received, one network connection observed, one file scanned). Each column is one piece of information recorded about that case (how many suspicious links were in the email, how many bytes the connection transferred, how large the file is). Somewhere, usually in the last column, sits the verdict: was this case spam or not, an attack or not, malware or not.

In ML terms:

| Detective's Notebook | ML Term |
|------------------------|---------|
| Each case (row) | **Sample** / **instance** / **record** |
| Each recorded detail (column) | **Feature** |
| The verdict column | **Label** / **target** |
| The whole notebook | **Dataset** |

### Structured vs. Unstructured Data

| Type | Description | Security Examples |
|------|-------------|--------------------|
| **Structured / Tabular** | Neatly organized into rows and columns, each column with a consistent type | Network flow logs (CSV), authentication logs, the NSL-KDD dataset |
| **Unstructured** | No fixed row/column shape -- text, images, raw bytes | Raw email bodies, malware binaries, PCAP files, screenshots |

This module works with both: Sections on Spam Classification and Network Anomaly Detection use tabular/text data that fits nicely into rows and columns (or gets converted into that form), while Malware Classification converts raw unstructured binary bytes into a structured image-like grid specifically so it *can* be handled by the tools built for structured data.

---

## 2. Anatomy of a Tabular Dataset

```
                          FEATURES (columns)                          LABEL
              +------------+------------+------------+------------+------------+
              | word_count | link_count | has_urgent | sender_new |   class    |
    +---------+------------+------------+------------+------------+------------+
    | Sample 1|     42     |     4      |     1      |     1      |   spam     |
    | Sample 2|     18     |     0      |     0      |     0      |    ham     |
    | Sample 3|     95     |     6      |     1      |     1      |   spam     |
    | Sample 4|     30     |     1      |     0      |     0      |    ham     |
    +---------+------------+------------+------------+------------+------------+
       ROWS = Samples (instances)          |               |
                                            |               |
                                     each column is a    the special column
                                     FEATURE              we are trying to
                                                            predict: the LABEL
```

### Key Terms Recap

| Term | Definition |
|------|-----------|
| **Feature matrix (X)** | The table of all feature columns, without the label -- what the model sees as input |
| **Label vector (y)** | The single column of correct answers -- what the model tries to predict |
| **Row / Sample / Instance** | One individual data point |
| **Column / Feature / Attribute** | One measured property shared across all samples |
| **Dimensionality** | The number of features (columns) in X -- "high-dimensional data" means many columns |
| **Cardinality (of a category)** | The number of distinct values a categorical feature can take (e.g., "protocol" might have cardinality 3: TCP, UDP, ICMP) |

By convention, when you load a dataset for ML, you almost always end up splitting it into `X` (everything except the label) and `y` (just the label column) before training anything.

---

## 3. Loading Data with pandas

**pandas** is the standard Python library for working with tabular data. Its core object is the **DataFrame** -- essentially a spreadsheet living inside Python, with rows, named columns, and a rich set of operations for filtering, transforming, and summarizing.

### Loading a CSV File

```python
import pandas as pd

# Load a comma-separated values file into a DataFrame
df = pd.read_csv("emails.csv")

# Peek at the first 5 rows
print(df.head())
```

```
                                                text label
0                Congratulations! You won $1,000,000!!!   spam
1                          Meeting moved to 3pm tomorrow    ham
2                  Buy cheap pills now, limited offer!!   spam
3                 Here is the Q3 report you requested     ham
4                  URGENT: Verify your account immediately   spam
```

### Common Loading Options

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `sep` | Specify a different delimiter (not comma) | `pd.read_csv("data.tsv", sep="\t")` |
| `header` | Tell pandas which row (if any) has column names | `pd.read_csv("data.csv", header=None)` (no header row) |
| `names` | Provide column names manually | `pd.read_csv("data.csv", names=["duration", "protocol", "label"])` |
| `usecols` | Load only specific columns | `pd.read_csv("data.csv", usecols=["duration", "label"])` |
| `nrows` | Load only the first N rows (useful for quickly previewing a huge file) | `pd.read_csv("data.csv", nrows=1000)` |
| `na_values` | Tell pandas which strings should be treated as missing | `pd.read_csv("data.csv", na_values=["?", "N/A"])` |

### Other Common Formats

```python
# JSON
df = pd.read_json("logs.json")

# Excel
df = pd.read_excel("report.xlsx", sheet_name="Sheet1")

# Parquet (a compressed, columnar format common for large datasets)
df = pd.read_parquet("flows.parquet")
```

---

## 4. Inspecting a Dataset

Before doing anything else with a dataset, you should always look at it from several angles. Skipping this step is one of the most common sources of bugs and bad models in ML work.

### Shape

```python
print(df.shape)
# (10000, 2)   --> 10,000 rows, 2 columns
```

### Column Names and Data Types

```python
print(df.dtypes)
```

```
text     object
label    object
dtype: object
```

`object` in pandas usually means "text" (technically, it means "Python object," but for CSV-loaded columns it is almost always strings). Numeric columns will show as `int64` or `float64`.

### Overview with `.info()`

```python
df.info()
```

```
<class 'pandas.core.frame.DataFrame'>
RangeIndex: 10000 entries, 0 to 9999
Data columns (total 2 columns):
 #   Column  Non-Null Count  Dtype
---  ------  --------------  -----
 0   text    9985 non-null   object
 1   label   10000 non-null  object
dtypes: object(2)
memory usage: 156.4+ KB
```

This single command tells you row count, column count, data type per column, and -- critically -- how many **non-null** values each column has. Here, `text` has 9,985 non-null out of 10,000 rows, meaning 15 rows are missing text. That is your first clue that preprocessing (Section 4 of this module) will be needed.

### Unique Values and Class Balance

```python
print(df["label"].value_counts())
```

```
ham     7000
spam    3000
Name: label, dtype: int64
```

This immediately tells you the dataset is **imbalanced**: roughly 70% ham, 30% spam. This single fact will influence which evaluation metrics you trust later (recall accuracy alone can be misleading, as covered in Module 1).

---

## 5. Summary Statistics

For numeric columns, `.describe()` gives you a fast statistical snapshot.

```python
import pandas as pd

df = pd.read_csv("network_flows.csv")
print(df.describe())
```

```
          duration     src_bytes      dst_bytes
count   10000.0000   10000.000000   10000.000000
mean       12.4300     452.120000     390.870000
std        45.1200    3210.550000    2894.310000
min         0.0000       0.000000       0.000000
25%         0.0000      45.000000      32.000000
50%         1.0000     181.000000     150.000000
75%         5.0000     512.000000     440.000000
max      1200.0000  180000.000000  165000.000000
```

### Reading This Table

| Statistic | What It Tells You |
|-----------|---------------------|
| **count** | How many non-missing values -- useful for spotting missing data again |
| **mean** | The average value |
| **std** | Standard deviation -- how spread out the values are |
| **min / max** | The range of the data -- huge max values relative to the mean can be a sign of outliers |
| **25% / 50% / 75%** | The quartiles -- 50% is the median. If mean is much larger than the median, the data is likely **right-skewed** (a long tail of large values) |

In the table above, `src_bytes` has a mean of 452 but a median (50%) of only 181, and a max of 180,000 -- a huge gap that signals a **skewed distribution** with some very large outliers (perhaps a few connections transferring enormous amounts of data). This is exactly the kind of finding that leads into the Data Preprocessing section.

### Categorical Summary Statistics

```python
print(df["protocol_type"].describe())
```

```
count       10000
unique          3
top           tcp
freq         7200
Name: protocol_type, dtype: object
```

For a categorical (non-numeric) column, `.describe()` instead reports the count, the number of unique categories, the most frequent category (`top`), and how often it appears (`freq`).

---

## 6. Visualizing Distributions

Numbers in a table are useful, but a picture makes patterns and outliers immediately obvious. `matplotlib` and `seaborn` are the standard visualization libraries used alongside pandas.

### Histogram of a Numeric Feature

```python
import matplotlib.pyplot as plt

df["src_bytes"].hist(bins=50)
plt.xlabel("Source Bytes")
plt.ylabel("Number of Connections")
plt.title("Distribution of Source Bytes per Connection")
plt.show()
```

```
    A RIGHT-SKEWED DISTRIBUTION (typical for byte counts)
    =======================================================

    Count
      |
    5K|##
      |####
    3K|######
      |########
    1K|############
      |#################........................
     0+---+---+---+---+---+---+---+---+---+---+---+
       0  1K  2K  5K 10K 20K 50K 100K            180K
                                                (Source Bytes)

    Most connections transfer very little data (tall bars near 0),
    but a long tail stretches out to rare, very large transfers.
```

### Bar Chart of a Categorical Feature / Class Balance

```python
df["label"].value_counts().plot(kind="bar")
plt.xlabel("Class")
plt.ylabel("Count")
plt.title("Class Balance: Spam vs Ham")
plt.show()
```

```
    CLASS BALANCE BAR CHART
    =========================

    Count
    7000| ########
        | ########
    3000| ######## ####
        | ######## ####
       0 +--------+----+
             ham   spam
```

### Boxplot for Spotting Outliers

```python
import seaborn as sns

sns.boxplot(x=df["src_bytes"])
plt.title("Boxplot of Source Bytes (Outlier Check)")
plt.show()
```

```
    BOXPLOT ANATOMY
    =================

         Q1   median   Q3
          |     |       |
    |------[=====|======]-----------------o  o    o
    |      box                whiskers    outliers

    Points far to the right of the whisker are outliers --
    exactly the kind of values that showed up as the huge
    max() in the summary statistics table above.
```

### Correlation Heatmap (Relationships Between Features)

```python
numeric_df = df.select_dtypes(include="number")
sns.heatmap(numeric_df.corr(), annot=True, cmap="coolwarm")
plt.title("Feature Correlation Heatmap")
plt.show()
```

```
    CORRELATION HEATMAP (values from -1 to +1)
    ============================================

                 duration  src_bytes  dst_bytes
    duration        1.00      0.12       0.09
    src_bytes       0.12      1.00       0.81   <-- strong positive
    dst_bytes       0.09      0.81       1.00       correlation

    Values near +1: features rise and fall together.
    Values near -1: one rises as the other falls.
    Values near  0: little to no linear relationship.
```

Strongly correlated features (like `src_bytes` and `dst_bytes` above) are worth noting -- they may carry redundant information, which becomes relevant when you get to feature selection and dimensionality reduction later in the course.

---

## 7. Worked Example: A First Look at a Security Dataset

Let's put every technique in this file together on a small, realistic phishing-email dataset.

```python
import pandas as pd
import matplotlib.pyplot as plt

# Step 1: Load
df = pd.read_csv("phishing_emails.csv")

# Step 2: Shape and structure
print("Shape:", df.shape)
print(df.dtypes)

# Step 3: Peek at the data
print(df.head())

# Step 4: Missing values
print("\nMissing values per column:")
print(df.isnull().sum())

# Step 5: Class balance
print("\nClass balance:")
print(df["label"].value_counts(normalize=True))

# Step 6: Summary statistics for numeric features
print("\nSummary statistics:")
print(df.describe())

# Step 7: Visualize one feature's distribution
df["num_links"].hist(bins=20)
plt.title("Distribution of Link Count per Email")
plt.xlabel("Number of Links")
plt.ylabel("Number of Emails")
plt.show()
```

Suppose this produces:

```
Shape: (5000, 6)

url_length         int64
num_links          int64
has_ip_address      int64
num_special_chars   int64
sender_domain_age  float64
label               object
dtype: object

Missing values per column:
url_length              0
num_links               0
has_ip_address          0
num_special_chars       0
sender_domain_age     120
label                   0
dtype: int64

Class balance:
phishing    0.62
legitimate  0.38
Name: label, dtype: float64
```

### What This Tells Us Before Writing a Single Line of Model Code

1. **5,000 samples, 6 columns** -- a modest dataset, manageable for classical ML.
2. **`sender_domain_age` has 120 missing values** -- we will need to handle this in preprocessing (Section 4).
3. **The dataset is moderately imbalanced** (62% phishing / 38% legitimate) -- accuracy alone will not be a reliable metric; we should watch precision and recall for both classes.
4. **All other features are complete and numeric** -- good news, less cleanup needed there.

This kind of quick, disciplined inspection -- shape, dtypes, missingness, class balance, summary stats, and one or two plots -- should be the very first thing you do with any new dataset, before any modeling begins.

---

## 8. Key Takeaways

- A **dataset** is a structured collection of samples (rows) and features (columns), with an optional label column that supervised learning tries to predict.
- **pandas** `DataFrame` objects and `pd.read_csv()` (plus JSON/Excel/Parquet equivalents) are the standard way to load tabular data into Python.
- Always inspect a new dataset with `.shape`, `.dtypes`, `.info()`, `.head()`, `.isnull().sum()`, and `.value_counts()` before doing anything else -- this catches missing data, wrong types, and class imbalance early.
- `.describe()` gives fast summary statistics; a large gap between the mean and the median is an early warning sign of skew and outliers.
- Histograms, bar charts, boxplots, and correlation heatmaps turn statistical summaries into visual patterns that are far easier to notice and reason about.
- Every worked project later in this module (spam classification, network anomaly detection, malware classification) starts with exactly this inspection ritual before any preprocessing or training happens.

*Next up: Data Preprocessing -- cleaning duplicates, handling missing values, encoding categories, and dealing with skewed or imbalanced data before it ever reaches a model.*
