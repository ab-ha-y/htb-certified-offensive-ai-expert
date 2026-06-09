# Unsupervised Learning Algorithms

## What Is Unsupervised Learning?

Unsupervised learning is a category of machine learning where the algorithm receives
**data without any labels or answers** and must discover structure, patterns, or
relationships on its own.

### The "Learning Without a Teacher" Analogy

Imagine you are a new student at a school in a foreign country. Nobody explains the
social groups to you. There is no teacher saying "these kids are athletes, those are
musicians, and that group is the science club." Instead, you observe everyone during
lunch for a few weeks. You notice:

- Some kids always sit together and wear jerseys.
- Another cluster carries instrument cases.
- A third group is always reading thick textbooks.

Over time, you **discover the groups yourself** based on patterns you observed --
without anyone ever telling you the labels. That is unsupervised learning.

Compare this to supervised learning, where a teacher hands you a class roster with
every student's club membership already filled in. Supervised learning has the answers;
unsupervised learning does not.

```
  SUPERVISED LEARNING                  UNSUPERVISED LEARNING
  =====================                ======================

  Input Data + Labels                  Input Data (NO Labels)
  +------------------+                 +------------------+
  | Feature | Label  |                 | Feature          |
  |---------|--------|                 |------------------|
  | 5.1     | Cat    |                 | 5.1              |
  | 2.3     | Dog    |                 | 2.3              |
  | 4.7     | Cat    |                 | 4.7              |
  | 1.9     | Dog    |                 | 1.9              |
  +------------------+                 +------------------+
       |                                    |
       v                                    v
  "Learn the mapping                  "Find hidden structure
   from features to                    in the data -- groups,
   known labels"                       patterns, anomalies"
```

---

## No Labels -- Finding Hidden Patterns

The defining characteristic of unsupervised learning is the **absence of a target
variable**. The algorithm is not trying to predict a known outcome. Instead, it
answers questions like:

- Are there natural groupings in this data?
- Which features are most important?
- Which data points are unusual?
- Which items tend to appear together?

This makes unsupervised learning powerful for **exploration and discovery** -- you use
it when you do not know what you are looking for yet.

---

## The Three Main Types of Unsupervised Learning

### 1. Clustering

**Goal:** Group similar data points together.

Think of sorting a pile of mixed coins. You do not need anyone to tell you that
quarters, dimes, nickels, and pennies are different -- you can group them by size,
color, and weight.

**Common Algorithms:**
- K-Means Clustering
- DBSCAN (Density-Based Spatial Clustering)
- Hierarchical Clustering
- Gaussian Mixture Models (GMM)

```
  CLUSTERING EXAMPLE
  ==================

  Raw Data:                     After Clustering:

     x  x                         [A] [A]
        x                             [A]
                x                          [B]
  x                               [A]
             x  x                       [B] [B]
                   x                           [B]

  The algorithm found two natural groups (A and B)
  without being told how many groups exist or what
  they represent.
```

### 2. Dimensionality Reduction

**Goal:** Reduce the number of features (dimensions) while preserving as much
information as possible.

Imagine you have a dataset with 500 columns. Most machine learning algorithms
struggle with that many features (the "curse of dimensionality"). Dimensionality
reduction compresses those 500 columns down to, say, 10 -- keeping the essential
patterns and discarding noise.

**Common Algorithms:**
- PCA (Principal Component Analysis)
- t-SNE (t-distributed Stochastic Neighbor Embedding)
- UMAP (Uniform Manifold Approximation and Projection)
- Autoencoders (deep learning approach)

```
  DIMENSIONALITY REDUCTION
  ========================

  Original: 5 features             Reduced: 2 features
  +----+----+----+----+----+       +------+------+
  | F1 | F2 | F3 | F4 | F5 |       | PC1  | PC2  |
  |----|----|----|----|----|       |------|------|
  | 2  | 4  | 3  | 5  | 1  |  ->  | 7.2  | 1.1  |
  | 8  | 7  | 9  | 6  | 8  |  ->  | 18.3 | 0.4  |
  | 1  | 2  | 1  | 3  | 2  |  ->  | 4.1  | 0.9  |
  +----+----+----+----+----+       +------+------+

  5 dimensions compressed to 2, preserving ~90% of variance
```

### 3. Association Rule Learning

**Goal:** Discover rules that describe relationships between variables.

The classic example is market basket analysis: "Customers who buy bread and butter
also tend to buy milk." The algorithm finds frequent item combinations and the rules
connecting them.

**Common Algorithms:**
- Apriori
- FP-Growth
- Eclat

```
  ASSOCIATION RULES
  =================

  Transaction Data:
  +-------+---------------------------+
  | TxnID | Items Purchased           |
  |-------|---------------------------|
  | 001   | Bread, Butter, Milk       |
  | 002   | Bread, Butter             |
  | 003   | Milk, Eggs                |
  | 004   | Bread, Butter, Milk, Eggs |
  | 005   | Bread, Milk               |
  +-------+---------------------------+

  Discovered Rule:
  {Bread, Butter} => {Milk}   (Confidence: 67%, Support: 40%)

  "If someone buys bread AND butter, there is a 67% chance
   they also buy milk."
```

---

## Supervised vs. Unsupervised: Comparison Table

| Aspect | Supervised Learning | Unsupervised Learning |
|---|---|---|
| **Training Data** | Labeled (input + known output) | Unlabeled (input only) |
| **Goal** | Predict a specific target | Discover hidden structure |
| **Human Effort** | High -- someone must label data | Low -- no labeling needed |
| **Output** | Predictions / classifications | Clusters, components, rules, anomalies |
| **Evaluation** | Clear metrics (accuracy, F1, etc.) | Harder to evaluate (silhouette score, etc.) |
| **Example Task** | "Is this email spam or not?" | "Group these emails into similar topics" |
| **When to Use** | You know what you want to predict | You want to explore or find unknowns |
| **Algorithms** | Linear Regression, SVM, Random Forest, Neural Networks | K-Means, PCA, DBSCAN, Apriori |
| **Risk** | Overfitting to labeled data | Discovering meaningless patterns |
| **Data Requirement** | Needs labeled examples (expensive) | Any raw data works |

---

## Semi-Supervised and Self-Supervised Learning (Brief Note)

In practice, you will also encounter these hybrid approaches:

- **Semi-supervised learning:** Uses a small amount of labeled data combined with a
  large amount of unlabeled data. The labeled data guides the algorithm, and the
  unlabeled data helps it generalize.

- **Self-supervised learning:** The algorithm creates its own labels from the data
  structure. For example, masking a word in a sentence and training the model to
  predict it. This is how large language models (LLMs) are trained.

---

## Security and Offensive Angle

Unsupervised learning is **critical in cybersecurity** because attackers constantly
evolve their techniques. Supervised models can only detect threats they were trained
on. Unsupervised models can find **unknown, never-before-seen threats**.

### Defensive Uses

| Application | How It Works |
|---|---|
| **Threat Hunting** | Cluster network traffic to find groups that do not match known patterns -- potential zero-day attacks |
| **Zero-Day Detection** | Anomaly detection flags behavior that deviates from the baseline, even if no signature exists |
| **Insider Threat Detection** | Cluster employee behavior profiles; flag users whose behavior suddenly shifts to a different cluster |
| **Malware Family Grouping** | Cluster malware samples by behavior to identify new variants of known families |
| **Network Segmentation Analysis** | Use clustering to discover actual communication patterns vs. intended network segments |
| **Log Analysis** | Reduce millions of log entries to a manageable number of clusters for analyst review |

### Offensive / Red Team Uses

| Application | How It Works |
|---|---|
| **Reconnaissance** | Cluster open-source intelligence (OSINT) data to identify target organization structure |
| **Evading Detection** | Understand what "normal" looks like to anomaly detection systems, then mimic it |
| **Attack Surface Mapping** | Use dimensionality reduction to visualize and prioritize large attack surfaces |
| **Credential Analysis** | Cluster leaked credential dumps to identify password patterns and reuse |
| **Social Engineering** | Association rules on social media data to map relationships between targets |

### Why This Matters for the Exam

The HTB Certified Offensive AI Expert exam expects you to understand:

1. **When to use unsupervised vs. supervised learning** -- if you have no labels (which
   is common in security), unsupervised is your tool.
2. **How attackers exploit unsupervised models** -- poisoning the training data so
   anomaly detection learns the wrong "normal."
3. **How defenders deploy unsupervised learning** -- clustering, anomaly detection, and
   dimensionality reduction are the backbone of modern SOC tooling.

---

## Key Terminology

| Term | Definition |
|---|---|
| **Unsupervised Learning** | ML where the algorithm learns from unlabeled data, discovering patterns without guidance |
| **Clustering** | Grouping similar data points together based on feature similarity |
| **Dimensionality Reduction** | Reducing the number of features while preserving important information |
| **Association Rules** | Discovering if-then relationships between variables in a dataset |
| **Label** | A known answer or category assigned to a data point (absent in unsupervised learning) |
| **Feature** | A measurable property of a data point (e.g., packet size, login time) |
| **Anomaly** | A data point that significantly deviates from the expected pattern |
| **Curse of Dimensionality** | The problem where algorithms perform poorly as the number of features grows very large |
| **Silhouette Score** | A metric measuring how well-separated clusters are (-1 to 1, higher is better) |
| **Inertia** | The sum of squared distances from each point to its assigned cluster center |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|---|---|
| No labeled data required -- saves time and cost | Results can be hard to interpret and validate |
| Can discover unknown patterns and anomalies | No ground truth to measure accuracy against |
| Works well for exploratory data analysis | May find patterns that are meaningless noise |
| Scales to large, unlabeled datasets | Sensitive to feature scaling and data preprocessing |
| Essential for zero-day and novel threat detection | Requires domain expertise to interpret results |
| Can be combined with supervised methods | Algorithm choice and hyperparameter tuning are tricky |

---

## Key Takeaways

1. **Unsupervised learning finds patterns in data without labels.** Think of it as
   exploring unknown territory without a map.

2. **Three main types:** Clustering (grouping), Dimensionality Reduction (compressing),
   and Association Rule Learning (finding relationships).

3. **Security is a prime use case** because threats evolve faster than labels can be
   created. Unsupervised learning catches what signature-based systems miss.

4. **Evaluation is harder** than supervised learning. There is no "answer key" to check
   against, so domain expertise matters.

5. **Attackers study unsupervised defenses** to learn how to blend into "normal"
   behavior and evade anomaly detection.

6. **Preprocessing matters enormously.** Feature scaling, outlier handling, and
   feature selection dramatically affect results.

7. **For the exam:** Know when to apply unsupervised vs. supervised learning, understand
   the three main categories, and be able to explain the security implications of each.
