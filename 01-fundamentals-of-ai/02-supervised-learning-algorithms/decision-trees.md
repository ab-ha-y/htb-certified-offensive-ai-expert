# Decision Trees

## What are Decision Trees?

A decision tree is a model that makes predictions by asking a series of **yes/no questions** about the data, one question at a time, until it reaches a final answer.

### The 20 Questions Game Analogy

Think of the game "20 Questions." One person thinks of something, and the other asks yes/no questions to narrow it down:

1. "Is it alive?" --> Yes
2. "Is it bigger than a breadbox?" --> Yes
3. "Does it live in water?" --> No
4. "Does it have four legs?" --> Yes
5. "Is it a pet?" --> Yes
6. **Answer: Dog!**

Each question eliminates possibilities and narrows down the answer. A decision tree works EXACTLY like this -- it learns which questions to ask and in what order to most efficiently classify or predict.

```
    Is it alive?
    |
    +-- YES --> Is it bigger than a breadbox?
    |           |
    |           +-- YES --> Does it have 4 legs?
    |           |           |
    |           |           +-- YES --> DOG
    |           |           +-- NO  --> HUMAN
    |           |
    |           +-- NO  --> Does it fly?
    |                       |
    |                       +-- YES --> BIRD
    |                       +-- NO  --> CAT
    |
    +-- NO  --> Is it electronic?
                |
                +-- YES --> COMPUTER
                +-- NO  --> ROCK
```

---

## How Splits Work

At each node (question), the decision tree picks the feature and threshold that **best separates** the data. But how does it decide which question is "best"?

### The Goal: Purity

A "pure" node contains only one class. The tree wants each split to move toward purity -- separating the classes as cleanly as possible.

```
    IMPURE (mixed)              PURE (separated)
    +----------+               +---------+  +---------+
    | o o x x  |    SPLIT -->  | o o o o |  | x x x x |
    | x o x o  |               | o o o o |  | x x x x |
    +----------+               +---------+  +---------+
    50% each class             100% Class 0  100% Class 1
```

### Gini Impurity (Simple Explanation)

Gini impurity measures how "mixed" a node is. Think of it as: **"If I randomly pick two items from this node, how likely are they to be different classes?"**

```
    Gini = 1 - (p1^2 + p2^2)

    Where p1 and p2 are the proportions of each class.
```

| Scenario | Class Distribution | Gini | Meaning |
|----------|-------------------|------|---------|
| Pure node | 100% Class A, 0% Class B | 1 - (1.0^2 + 0.0^2) = 0.0 | Perfect! No mixing. |
| Worst case | 50% Class A, 50% Class B | 1 - (0.5^2 + 0.5^2) = 0.5 | Maximum confusion. |
| Mostly one class | 80% Class A, 20% Class B | 1 - (0.8^2 + 0.2^2) = 0.32 | Fairly pure. |

**Lower Gini = better split.** The tree picks the split that results in the lowest weighted average Gini across child nodes.

### Information Gain (Simple Explanation)

Information gain is another way to measure split quality. It uses **entropy** (a measure of disorder/uncertainty from information theory).

```
    Entropy = -SUM(p_i * log2(p_i)) for each class i
```

| Scenario | Entropy | Meaning |
|----------|---------|---------|
| Pure (100/0) | 0.0 | No uncertainty -- we know the class |
| Maximally mixed (50/50) | 1.0 | Maximum uncertainty |
| Mostly one class (80/20) | 0.72 | Some uncertainty |

**Information Gain = Entropy(parent) - Weighted Average Entropy(children)**

Higher information gain = the split reduces uncertainty the most = better question to ask.

> **For the exam:** You do not need to calculate these by hand. Just know that Gini and Information Gain both measure how well a split separates classes, and the tree greedily picks the best split at each step.

---

## ASCII Art: A Decision Tree for Network Traffic

```
    [Root: Packet Rate > 1000/sec?]
    |
    +-- YES ---[Port == 80 or 443?]
    |          |
    |          +-- YES ---[Payload Size > 5KB?]
    |          |          |
    |          |          +-- YES --> MALICIOUS (DDoS)
    |          |          |          [12 samples, Gini=0.0]
    |          |          |
    |          |          +-- NO  --> NORMAL (Web browsing)
    |          |                     [8 samples, Gini=0.0]
    |          |
    |          +-- NO  ---[Source IPs > 100?]
    |                     |
    |                     +-- YES --> MALICIOUS (Port Scan)
    |                     |          [6 samples, Gini=0.0]
    |                     |
    |                     +-- NO  --> NORMAL (Internal traffic)
    |                                [4 samples, Gini=0.1]
    |
    +-- NO  ---[Connection Duration > 30min?]
               |
               +-- YES --> SUSPICIOUS (C2 Beacon)
               |           [3 samples, Gini=0.0]
               |
               +-- NO  --> NORMAL
                           [15 samples, Gini=0.05]

    Reading the tree:
    - Start at the root
    - Follow YES/NO branches based on the data
    - Leaf nodes give the final classification
```

---

## Overfitting in Decision Trees

Decision trees are particularly prone to **overfitting** -- memorizing the training data instead of learning general patterns.

### What Overfitting Looks Like

```
    Underfitting              Just Right              Overfitting
    (too simple)              (good generalization)   (memorized training data)

    Tree depth: 1             Tree depth: 4           Tree depth: 20
    Accuracy (train): 60%     Accuracy (train): 92%   Accuracy (train): 100%
    Accuracy (test):  58%     Accuracy (test):  90%   Accuracy (test):  65%
                                                       ^^ Big gap = overfitting!
```

An overfit tree creates extremely specific rules that match the noise in the training data:

```
    Overfit tree might learn:
    "If packet_size == 347 AND source_port == 54821 AND time == 14:32:07
     THEN malicious"

    This is too specific -- it memorized individual training examples
    instead of learning general attack patterns.
```

### How to Prevent Overfitting

| Technique | How It Works |
|-----------|-------------|
| **Max depth** | Limit how deep the tree can grow (e.g., max 5 levels) |
| **Min samples per leaf** | Require at least N samples in each leaf node |
| **Min samples per split** | Require at least N samples to allow a split |
| **Pruning** | Grow the full tree, then remove branches that do not improve test accuracy |
| **Random Forests** | Use many trees together (see below) |

---

## Random Forests

A **Random Forest** is a collection of many decision trees that work together. Each tree is slightly different, and the forest takes a **majority vote** to make the final prediction.

### Why Multiple Trees?

One tree can be wrong. But if you build 100 slightly different trees and most of them agree, the group prediction is usually much better than any individual tree.

```
    Single Decision Tree:         Random Forest (5 trees):

    [Tree] --> "Malicious"        [Tree 1] --> "Malicious"
                                  [Tree 2] --> "Benign"
    Might be wrong!               [Tree 3] --> "Malicious"
                                  [Tree 4] --> "Malicious"
                                  [Tree 5] --> "Benign"

                                  Vote: 3 Malicious vs 2 Benign
                                  Final: "Malicious" (majority wins)
```

### How Trees Become Different

Each tree is trained on:
1. A **random subset of the training data** (called bagging/bootstrap sampling)
2. A **random subset of features** at each split

This randomness makes each tree unique, and their combined wisdom is more reliable.

| Aspect | Single Decision Tree | Random Forest |
|--------|---------------------|---------------|
| Accuracy | Moderate | High |
| Overfitting risk | High | Low |
| Interpretability | Easy to read | Hard to interpret (many trees) |
| Training speed | Fast | Slower (building many trees) |
| Prediction speed | Very fast | Slightly slower |

---

## Worked Example: Classifying Network Traffic

### The Data

We are classifying network connections as **Malicious (1)** or **Benign (0)**.

| Connection | Packets/sec (x1) | Avg Payload (bytes) (x2) | Unique Dest IPs (x3) | Label |
|------------|------------------|--------------------------|-----------------------|-------|
| A | 50 | 500 | 3 | Benign |
| B | 2000 | 100 | 200 | Malicious |
| C | 30 | 800 | 1 | Benign |
| D | 5000 | 50 | 500 | Malicious |
| E | 100 | 400 | 5 | Benign |
| F | 3000 | 80 | 300 | Malicious |

### Step 1: Find the Best First Split

The algorithm tests every feature and threshold. Let us try **Packets/sec > 500?**

```
    Split: Packets/sec > 500?

    YES (>500):              NO (<=500):
    B: Malicious             A: Benign
    D: Malicious             C: Benign
    F: Malicious             E: Benign

    Gini(YES) = 1 - (1.0^2 + 0.0^2) = 0.0   (pure!)
    Gini(NO)  = 1 - (1.0^2 + 0.0^2) = 0.0   (pure!)

    This is a PERFECT split!
```

### Step 2: Build the Tree

```
    [Packets/sec > 500?]
    |
    +-- YES --> MALICIOUS
    |           (3/3 samples are malicious)
    |
    +-- NO  --> BENIGN
                (3/3 samples are benign)
```

In this simplified example, one question was enough. Real trees need many levels.

### Step 3: Classify a New Connection

A new connection arrives: 1500 packets/sec, 200-byte payload, 150 unique destinations.

```
    Packets/sec > 500?  -->  1500 > 500?  -->  YES

    Prediction: MALICIOUS
```

### Step 4: A More Realistic Tree

With messier data, the tree would need more splits:

```
    [Packets/sec > 500?]
    |
    +-- YES --[Unique Dest IPs > 100?]
    |         |
    |         +-- YES --> MALICIOUS (DDoS/Scan)
    |         +-- NO  --[Payload > 1000?]
    |                    |
    |                    +-- YES --> BENIGN (Large file transfer)
    |                    +-- NO  --> MALICIOUS (C2 traffic)
    |
    +-- NO  --[Payload > 200?]
              |
              +-- YES --> BENIGN (Normal browsing)
              +-- NO  --[Duration > 3600s?]
                         |
                         +-- YES --> MALICIOUS (Slow C2 beacon)
                         +-- NO  --> BENIGN
```

---

## Security Angle

### Where Decision Trees Are Used in Security

**1. Web Application Firewalls (WAFs)**
- Decision trees classify HTTP requests as attack or legitimate
- Features: URL length, number of special characters, presence of SQL keywords, HTTP method
- The tree structure makes rules human-readable (important for security analysts)

**2. Malware Classification**
- Classify executables based on static and dynamic features
- Features: imported DLLs, file entropy, section names, API call sequences
- Random forests are particularly popular for malware family classification

**3. Network Intrusion Detection**
- Classify network flows by attack type
- The C4.5 and CART algorithms (decision tree variants) were among the first ML algorithms used in IDS research

**4. Alert Triage**
- Prioritize security alerts by predicted severity
- Features: alert type, source reputation, time of day, historical context
- Helps SOC analysts focus on the most important alerts

### How Attackers Exploit Decision Trees

**1. Rule Extraction**

Decision trees are **interpretable** -- you can read the rules. If an attacker can access or reconstruct the tree, they know EXACTLY what to avoid:

```
    If an attacker learns the WAF tree:

    [URL contains "UNION"?]
    |
    +-- YES --> BLOCK

    The attacker simply avoids "UNION" and uses
    equivalent SQL syntax: "UniOn", "UN/**/ION", etc.
```

**2. Adversarial Feature Engineering**

| Tree Rule | Attacker's Evasion |
|-----------|-------------------|
| "Block if URL > 200 chars" | Split the attack across multiple shorter requests |
| "Block if special chars > 5" | Use URL encoding to hide special characters |
| "Block if source is in blacklist" | Rotate through many source IPs (proxy chains) |
| "Block if payload entropy > 7.5" | Add padding to reduce entropy |

**3. Decision Boundary Walking**

Because decision trees create axis-aligned splits (straight horizontal/vertical cuts), attackers can probe the model to find exactly where each split is:

```
    Send request with URL_length = 100 --> Allowed
    Send request with URL_length = 200 --> Blocked
    Send request with URL_length = 150 --> Allowed
    Send request with URL_length = 175 --> Blocked
    Send request with URL_length = 162 --> Allowed
    Send request with URL_length = 168 --> Blocked

    Now the attacker knows: the split is at URL_length = 165
    Keep all attacks under 165 characters.
```

**4. Poisoning Training Data**

If the attacker can influence the training data:
- Submit many benign-looking requests that contain attack patterns
- The tree learns that those patterns are "normal"
- Future attacks using those patterns slip through

---

## Key Terminology

| Term | Definition |
|------|-----------|
| **Root node** | The topmost node; the first question asked |
| **Internal node** | A node that asks a question and has children |
| **Leaf node** | A terminal node that gives a final prediction (no more questions) |
| **Split** | The condition at each internal node (e.g., "Packets > 500?") |
| **Depth** | The number of levels from root to the deepest leaf |
| **Gini impurity** | Measure of how mixed the classes are at a node (0 = pure) |
| **Information gain** | How much a split reduces uncertainty (higher = better) |
| **Entropy** | Measure of disorder/uncertainty in a set of labels |
| **Pruning** | Removing branches that do not improve generalization |
| **Bagging** | Training multiple models on random subsets of data (used in Random Forests) |
| **Ensemble** | A group of models working together (Random Forest is an ensemble) |
| **CART** | Classification and Regression Trees -- a specific decision tree algorithm |
| **Feature importance** | A score showing which features the tree relied on most for splits |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|-----------|-----------|
| Easy to understand and visualize | Prone to overfitting (especially deep trees) |
| No feature scaling needed | Sensitive to small changes in data (unstable) |
| Handles both numerical and categorical data | Axis-aligned splits cannot capture diagonal boundaries |
| Can model non-linear relationships | Greedy algorithm -- may not find the globally optimal tree |
| Feature importance built in | Single trees have lower accuracy than ensembles |
| Fast prediction (just follow the branches) | Can create biased trees if classes are imbalanced |
| Works well with Random Forests/boosting | Interpretability decreases as depth increases |

---

## Key Takeaways

1. **Decision trees ask a series of yes/no questions** to classify data. Each question splits the data into purer groups, like playing 20 Questions.

2. **Gini impurity and information gain** measure how good a split is. The tree greedily picks the split that best separates classes at each step.

3. **Overfitting is the main risk.** Deep trees memorize training data. Control this with max depth, min samples per leaf, or pruning.

4. **Random Forests fix the overfitting problem** by building many different trees and taking a majority vote. This is one of the most popular ML algorithms in practice.

5. **Decision trees are highly interpretable** -- you can read the rules. In security, this is both a strength (analysts can understand the model) and a weakness (attackers can reverse-engineer the rules).

6. **Attackers exploit decision trees** by probing to find split thresholds, then crafting inputs that stay on the "allowed" side of each split.

7. **For the exam:** Understand how splits work conceptually, why overfitting happens in deep trees, how Random Forests improve on single trees, and how an attacker can probe a decision tree-based WAF to discover its rules.
