# Introduction to Machine Learning

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Fundamentals of AI | Section: Introduction to Machine Learning

---

## Table of Contents

1. [What is Machine Learning?](#1-what-is-machine-learning)
2. [Types of Machine Learning](#2-types-of-machine-learning)
3. [Key Terminology](#3-key-terminology)
4. [The ML Pipeline](#4-the-ml-pipeline)
5. [How ML Differs from Traditional Programming](#5-how-ml-differs-from-traditional-programming)
6. [Common ML Metrics](#6-common-ml-metrics)
7. [Security Angle -- Why This Matters for Offensive Security](#7-security-angle----why-this-matters-for-offensive-security)
8. [Real-World Example -- Spam Detection Walkthrough](#8-real-world-example----spam-detection-walkthrough)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What is Machine Learning?

Machine Learning (ML) is a branch of artificial intelligence where computers **learn patterns from data** instead of being explicitly told what to do.

### The Analogy

Think about how you learned to recognize dogs as a child. Nobody handed you a 50-page specification document listing every possible breed, color, and size. Instead, people pointed at dogs and said "dog." After seeing enough examples, you could recognize a dog you had never seen before -- even an unusual breed.

Machine learning works the same way. You feed the computer thousands of examples, and it figures out the underlying patterns on its own.

### Another Way to Think About It

Imagine you are training a new security analyst:

- **Traditional programming** = giving them a 200-page manual with exact rules for every scenario.
- **Machine learning** = showing them 10,000 past security incidents and letting them learn to recognize patterns themselves.

The ML approach is powerful because it can discover patterns that humans might miss, and it can scale to volumes of data no human could process.

---

## 2. Types of Machine Learning

There are three major categories. Each gets its own dedicated section later in this course, so this is just an orientation.

```
+-------------------------------------------------------------+
|                   MACHINE LEARNING                          |
+-------------------------------------------------------------+
|                    |                    |                    |
|   SUPERVISED       |   UNSUPERVISED     |   REINFORCEMENT   |
|   LEARNING         |   LEARNING         |   LEARNING        |
|                    |                    |                    |
| "Learn from        | "Find hidden       | "Learn by trial   |
|  labeled examples" |  structure"        |  and error"       |
|                    |                    |                    |
| Examples:          | Examples:          | Examples:          |
| - Spam detection   | - Customer         | - Game-playing     |
| - Malware          |   segmentation     |   agents           |
|   classification   | - Anomaly          | - Autonomous       |
| - Price prediction |   detection        |   navigation       |
|                    | - Dimensionality   | - Adaptive         |
|                    |   reduction        |   network defense  |
+--------------------+--------------------+--------------------+
```

| Type | Input | Goal | Security Example |
|------|-------|------|------------------|
| **Supervised** | Data with known answers (labels) | Learn to predict the correct answer for new data | Classify network traffic as malicious or benign |
| **Unsupervised** | Data without labels | Discover hidden groupings or patterns | Detect anomalous behavior in logs without prior attack examples |
| **Reinforcement** | An environment with rewards/penalties | Learn a strategy that maximizes reward over time | An agent learning to navigate and exploit a network |

---

## 3. Key Terminology

This glossary covers every foundational term you need. Come back to this section as a reference.

### Core Concepts

| Term | Definition | Plain English |
|------|-----------|---------------|
| **Model** | A mathematical structure that has learned patterns from data. | The "brain" that the machine builds during training. It takes in data and produces predictions. |
| **Algorithm** | The procedure or set of rules the computer follows to learn from data. | The recipe the computer uses to build the model. |
| **Training** | The process of feeding data to an algorithm so it can learn patterns. | Teaching the model by showing it examples. |
| **Inference** | Using a trained model to make predictions on new, unseen data. | Asking the trained model a question and getting an answer. |
| **Parameters** | Internal values the model learns during training (e.g., weights in a neural network). | The knobs the model adjusts on its own while learning. |
| **Hyperparameters** | Settings you configure *before* training begins (e.g., learning rate, number of layers). | The knobs *you* set before letting the model learn. |

### Data Terminology

| Term | Definition | Plain English |
|------|-----------|---------------|
| **Dataset** | A collection of data used for training and/or evaluation. | The textbook the model studies from. |
| **Features** | The input variables (columns) used to make a prediction. | The individual pieces of information the model looks at. For a house price model: square footage, number of bedrooms, zip code. |
| **Labels** | The known correct answers in supervised learning. | The answer key. For email classification: "spam" or "not spam." |
| **Sample / Instance** | A single data point (row) in the dataset. | One example the model learns from. |
| **Training Set** | The portion of data used to train the model. | The practice problems. |
| **Validation Set** | Data used to tune hyperparameters during development. | The practice quiz you take to check how you are doing. |
| **Test Set** | Data held out until the very end to get an unbiased performance estimate. | The final exam. The model has never seen this data. |

### Training Concepts

| Term | Definition | Plain English |
|------|-----------|---------------|
| **Epoch** | One complete pass through the entire training dataset. | Reading the entire textbook once. Training often involves many epochs. |
| **Batch** | A subset of the training data processed together in one step. | Reading one chapter at a time instead of the whole book. |
| **Batch Size** | The number of samples in one batch. | How many pages you read before pausing to take notes. |
| **Learning Rate** | How much the model adjusts its parameters after each batch. | How big of a step you take when correcting mistakes. Too big and you overshoot; too small and you learn too slowly. |
| **Loss Function** | A function that measures how wrong the model's predictions are. | The scoring system that tells the model how badly it messed up. |
| **Gradient Descent** | The optimization method used to minimize the loss function. | Rolling a ball downhill to find the lowest point -- the model adjusts to reduce its error. |

### Model Quality

| Term | Definition | Plain English |
|------|-----------|---------------|
| **Overfitting** | The model memorizes the training data and performs poorly on new data. | A student who memorizes every answer in the textbook but cannot solve new problems on the exam. |
| **Underfitting** | The model is too simple to capture the patterns in the data. | A student who barely studied and cannot answer any questions well, even from the textbook. |
| **Bias** | Error from overly simple assumptions in the model (leads to underfitting). | The model is too stubborn -- it has strong preconceptions and ignores the data. |
| **Variance** | Error from being too sensitive to small fluctuations in training data (leads to overfitting). | The model is too impressionable -- it treats noise as signal. |
| **Generalization** | The model's ability to perform well on unseen data. | What actually matters -- can it handle the real world, not just the training data? |
| **Regularization** | Techniques to prevent overfitting (e.g., L1, L2, dropout). | Guardrails that keep the model from memorizing instead of learning. |

```
    THE BIAS-VARIANCE TRADEOFF

    High Bias                              High Variance
    (Underfitting)                         (Overfitting)
    
    +-------------+                        +-------------+
    |  .  .       |                        | ...***...   |
    |     .  .    |    <-- Goal -->         |.*..*..*..*. |
    |  .     .    |    Just Right           | .**.*..**.. |
    | ______      |                        | ~~~~~~~~~~~~|
    | simple line |                        | wiggly line |
    +-------------+                        +-------------+
    
    Model is too rigid.                    Model fits every point,
    Misses the real pattern.               including the noise.
    
                       SWEET SPOT
                    +-------------+
                    |  .  .       |
                    |    ~.~ .    |
                    |  .~   ~.   |
                    | smooth curve|
                    +-------------+
                    
                    Captures the real
                    pattern, ignores noise.
```

---

## 4. The ML Pipeline

Every ML project follows roughly the same lifecycle. Understanding this pipeline is critical both for building models and for attacking them.

```
+----------------+     +----------------+     +----------------+
|                |     |                |     |                |
|  1. PROBLEM    |---->|  2. DATA       |---->|  3. DATA       |
|  DEFINITION    |     |  COLLECTION    |     |  PREPROCESSING |
|                |     |                |     |                |
+----------------+     +----------------+     +----------------+
                                                     |
                                                     v
+----------------+     +----------------+     +----------------+
|                |     |                |     |                |
|  6. DEPLOYMENT |<----|  5. MODEL      |<----|  4. MODEL      |
|  & MONITORING  |     |  EVALUATION    |     |  TRAINING      |
|                |     |                |     |                |
+----------------+     +----------------+     +----------------+
```

### Stage-by-Stage Breakdown

#### Stage 1: Problem Definition
- What are you trying to predict or detect?
- What does success look like?
- Example: "Detect whether a network packet is part of a DDoS attack."

#### Stage 2: Data Collection
- Gather raw data from relevant sources.
- In security: packet captures, system logs, malware binaries, phishing emails.
- Data quality matters enormously. Garbage in, garbage out.

#### Stage 3: Data Preprocessing
- **Cleaning**: Remove duplicates, handle missing values, fix errors.
- **Feature Engineering**: Create new meaningful features from raw data (e.g., "average packet size over last 10 seconds" from raw packet data).
- **Normalization/Scaling**: Put features on comparable scales so the model does not overweight large numbers.
- **Encoding**: Convert text or categories into numbers the model can process.
- **Splitting**: Divide data into training, validation, and test sets (common split: 70/15/15 or 80/10/10).

#### Stage 4: Model Training
- Choose an algorithm (decision tree, neural network, SVM, etc.).
- Feed the training data to the algorithm.
- The model iteratively adjusts its parameters to minimize prediction error.
- Tune hyperparameters using the validation set.

#### Stage 5: Model Evaluation
- Test the model on the held-out test set.
- Compute metrics (accuracy, precision, recall, F1 -- see Section 6).
- Check for overfitting: is training performance much better than test performance?

#### Stage 6: Deployment and Monitoring
- Put the model into production (API endpoint, embedded in app, edge device, etc.).
- Monitor for **model drift**: performance degradation over time as real-world data changes.
- Retrain periodically with fresh data.

---

## 5. How ML Differs from Traditional Programming

This is a fundamental conceptual shift. Make sure you internalize this.

```
    TRADITIONAL PROGRAMMING
    ========================
    
    +-----------+
    |   DATA    |----+
    +-----------+    |     +-----------+     +------------+
                     +---->| PROGRAM   |---->|  OUTPUT    |
    +-----------+    |     | (Rules)   |     |            |
    |   RULES   |----+     +-----------+     +------------+
    +-----------+
    
    You write the rules. The computer follows them.
    Example: if packet_size > 1500 AND rate > 1000/sec THEN alert("DDoS")
    
    
    MACHINE LEARNING
    =================
    
    +-----------+
    |   DATA    |----+
    +-----------+    |     +-----------+     +------------+
                     +---->| LEARNING  |---->|  RULES     |
    +-----------+    |     | ALGORITHM |     |  (Model)   |
    |  OUTPUT   |----+     +-----------+     +------------+
    | (Labels)  |
    +-----------+
    
    You provide data and answers. The computer figures out the rules.
    Example: feed in 100,000 labeled packets --> model learns its own DDoS detection rules.
```

| Aspect | Traditional Programming | Machine Learning |
|--------|------------------------|------------------|
| **Input** | Rules + Data | Data + Labels (answers) |
| **Output** | Answers | Rules (the model) |
| **Who writes logic?** | The programmer | The algorithm discovers it |
| **Handles new patterns?** | Only if rules are updated manually | Can generalize to new patterns |
| **Maintenance** | Update rules as threats evolve | Retrain with new data |
| **Explainability** | Rules are transparent | Model may be a "black box" |
| **Scales with complexity?** | Becomes unmanageable | Handles high-dimensional data naturally |

---

## 6. Common ML Metrics

When you evaluate a model -- or when you are attacking one -- you need to understand how performance is measured.

### The Confusion Matrix

For a binary classifier (e.g., "malicious" vs. "benign"), every prediction falls into one of four boxes:

```
                          PREDICTED
                    +----------+----------+
                    | Positive | Negative |
        +-----------+----------+----------+
        | Positive  |    TP    |    FN    |
 ACTUAL +-----------+----------+----------+
        | Negative  |    FP    |    TN    |
        +-----------+----------+----------+

 TP = True Positive   -- Correctly identified as positive
 FP = False Positive  -- Incorrectly flagged as positive (false alarm)
 FN = False Negative  -- Missed a positive case (the dangerous one)
 TN = True Negative   -- Correctly identified as negative
```

### Metrics Derived from the Confusion Matrix

| Metric | Formula | What It Tells You | When It Matters Most |
|--------|---------|-------------------|---------------------|
| **Accuracy** | (TP + TN) / Total | Overall correctness | Balanced datasets where both classes matter equally |
| **Precision** | TP / (TP + FP) | Of everything flagged positive, how many actually were? | When false alarms are costly (e.g., blocking legitimate users) |
| **Recall** (Sensitivity) | TP / (TP + FN) | Of all actual positives, how many did we catch? | When missing a positive is dangerous (e.g., missing malware) |
| **F1-Score** | 2 * (Precision * Recall) / (Precision + Recall) | Harmonic mean of Precision and Recall | When you need a single balanced metric and data is imbalanced |
| **Specificity** | TN / (TN + FP) | Of all actual negatives, how many were correctly identified? | When false positives are very costly |

### A Concrete Example

Imagine a malware detection model tested on 1,000 files:
- 100 are actually malware, 900 are benign.
- The model flags 120 files as malware.

```
Confusion Matrix:
                     PREDICTED
                  Malware    Benign
 ACTUAL Malware  [  90   |   10  ]    (caught 90, missed 10)
        Benign   [  30   |  870  ]    (30 false alarms)

 Accuracy   = (90 + 870) / 1000           = 96.0%
 Precision  = 90 / (90 + 30)              = 75.0%
 Recall     = 90 / (90 + 10)              = 90.0%
 F1-Score   = 2 * (0.75 * 0.90) / (1.65)  = 81.8%
```

**Why accuracy alone can be misleading**: If the model just said "benign" for everything, it would get 900/1000 = 90% accuracy while catching zero malware. This is why precision, recall, and F1 matter -- especially in security.

---

## 7. Security Angle -- Why This Matters for Offensive Security

As an offensive security professional, you are not just building ML systems -- you are learning how to **break** them. ML systems introduce a new and expanding attack surface.

### The Attacker's View of an ML System

```
                       ATTACK SURFACE OF AN ML SYSTEM
    
    +-------------------+     +-------------------+     +-------------------+
    |   TRAINING DATA   |     |      MODEL        |     |   INFERENCE       |
    |                   |     |                   |     |   (Production)    |
    | - Data Poisoning  |     | - Model Stealing  |     | - Evasion Attacks |
    | - Label Flipping  |     | - Model Inversion |     | - Adversarial     |
    | - Backdoor        |     | - Membership      |     |   Examples        |
    |   Injection       |     |   Inference       |     | - Input Manip.    |
    +-------------------+     +-------------------+     +-------------------+
          ^                         ^                         ^
          |                         |                         |
     TRAINING PHASE           MODEL ITSELF              DEPLOYMENT PHASE
```

### Key Attack Categories

#### 1. Adversarial Examples (Evasion Attacks)
Crafting inputs that look normal to humans but fool the model.

- Add tiny, imperceptible noise to a malware sample so the detector classifies it as benign.
- Modify a phishing email just enough that the spam filter lets it through.
- The model's decision boundary can be crossed with surprisingly small perturbations.

**Why it matters**: If you know how a defensive ML model works, you can craft inputs that bypass it entirely.

#### 2. Data Poisoning
Injecting malicious data into the training set to corrupt the model.

- Compromise the data pipeline and inject mislabeled samples.
- Example: Add legitimate-looking login events labeled as "normal" that are actually attack patterns. The trained model will then ignore those attack patterns.

**Why it matters**: Attackers who can influence training data can embed persistent backdoors in models.

#### 3. Model Stealing (Model Extraction)
Recreating a proprietary model by querying it repeatedly.

- Send thousands of carefully chosen inputs to an API.
- Record the outputs (predictions, confidence scores).
- Train a "shadow model" that mimics the target's behavior.
- Now you can study the clone offline to find weaknesses.

**Why it matters**: Even "black-box" models can be reverse-engineered if they expose an API.

#### 4. Model Inversion
Extracting sensitive training data from a model's outputs.

- Query a facial recognition model to reconstruct faces from the training set.
- Infer private information (medical records, financial data) that the model learned from.

**Why it matters**: ML models can inadvertently memorize and leak sensitive training data.

#### 5. Membership Inference
Determining whether a specific data point was in the training set.

- "Was this patient's record used to train your health prediction model?"
- Exploits the fact that models behave differently on data they have seen vs. data they have not.

**Why it matters**: This is a privacy attack. It can reveal whether someone's data was used without consent.

### The Offensive Mindset

| Traditional Target | ML-Era Target |
|-------------------|---------------|
| Firewall rules | ML-based IDS/IPS |
| Signature-based AV | ML malware classifiers |
| Static spam filters | NLP-based email filters |
| Rule-based WAFs | ML anomaly detectors |
| Manual code review | AI-assisted code analysis |

**Key insight**: Every ML model is a new attack surface. The same properties that make ML powerful -- learning from data, generalizing to new inputs -- also make it vulnerable to manipulation.

---

## 8. Real-World Example -- Spam Detection Walkthrough

Let us walk through a complete (simplified) ML pipeline for building a spam email classifier. This makes every concept from earlier sections concrete.

### Step 1: Problem Definition

**Goal**: Given an email, classify it as "spam" or "not spam" (ham).

### Step 2: Data Collection

We gather 10,000 emails, each labeled by humans:
- 3,000 spam emails
- 7,000 legitimate (ham) emails

```
Sample data:

| Email Text                                    | Label |
|-----------------------------------------------|-------|
| "Congratulations! You won $1,000,000!!!"      | spam  |
| "Meeting moved to 3pm tomorrow"               | ham   |
| "Buy cheap pills now, limited offer!"         | spam  |
| "Here is the Q3 report you requested"         | ham   |
| "URGENT: Verify your account immediately"     | spam  |
```

### Step 3: Data Preprocessing

**a) Text Cleaning**
- Convert to lowercase.
- Remove punctuation, special characters, HTML tags.
- Remove stop words (the, is, at, etc.) -- or keep them, depending on the approach.

**b) Feature Engineering**
We transform raw email text into numerical features the model can work with:

```
Feature examples:

| Feature Name             | Email 1 (spam) | Email 2 (ham) |
|--------------------------|----------------|---------------|
| contains_"free"          | 1              | 0             |
| contains_"urgent"        | 1              | 0             |
| contains_"meeting"       | 0              | 1             |
| count_exclamation_marks  | 5              | 0             |
| count_uppercase_words    | 3              | 0             |
| has_attachment           | 0              | 1             |
| sender_in_contacts       | 0              | 1             |
| link_count               | 4              | 1             |
```

**c) Split the Data**

```
10,000 emails
     |
     +--- 7,000 Training Set   (70%)  -- model learns from these
     +--- 1,500 Validation Set (15%)  -- tune hyperparameters
     +--- 1,500 Test Set       (15%)  -- final evaluation only
```

### Step 4: Model Training

We pick a simple algorithm -- say, a **Naive Bayes** classifier (commonly used for text classification).

```
Training loop (simplified):

    For each epoch:
        For each batch of emails in training set:
            1. Model makes predictions  -->  "spam" or "ham"
            2. Loss function computes error (how wrong was it?)
            3. Model adjusts parameters to reduce error
        
        Check performance on validation set
        If not improving --> stop (early stopping)
```

After training, the model has learned patterns like:
- Emails with "free," "winner," multiple exclamation marks --> likely spam
- Emails with names of known contacts, normal punctuation --> likely ham

### Step 5: Model Evaluation

We run the model on the 1,500 test emails it has never seen:

```
Results on Test Set:

                      PREDICTED
                   Spam       Ham
 ACTUAL  Spam   [  420   |   30  ]
         Ham    [   15   |  1035 ]

 Total test emails: 1,500
 Accuracy  = (420 + 1035) / 1500 = 97.0%
 Precision = 420 / (420 + 15)    = 96.6%  (few false alarms)
 Recall    = 420 / (420 + 30)    = 93.3%  (missed some spam)
 F1-Score  = 2 * (0.966 * 0.933) / (1.899) = 94.9%
```

The model catches 93.3% of spam while only falsely flagging 15 legitimate emails.

### Step 6: Deployment

- Deploy as an API or integrate into the mail server.
- Each incoming email is converted to features and passed to the model.
- Model returns "spam" or "ham" with a confidence score.
- Monitor for drift: spammers constantly change tactics, so the model may need retraining.

### How an Attacker Would Target This System

| Attack | Method | Effect |
|--------|--------|--------|
| **Evasion** | Replace "free" with "fr33," use Unicode lookalike characters, insert invisible text | Spam bypasses the filter |
| **Data Poisoning** | Compromise the training pipeline, inject spam emails labeled as "ham" | Model learns to let spam through |
| **Model Stealing** | Send thousands of emails through the filter, observe which are blocked vs. allowed | Reconstruct the model's rules, then craft bypasses |

---

## 9. Key Takeaways

- **Machine learning is pattern recognition from data.** The computer learns rules from examples rather than being programmed with explicit rules.

- **Three types of ML**: Supervised (labeled data), Unsupervised (no labels, find structure), Reinforcement (learn via reward/penalty). Each has different security implications.

- **The ML pipeline** (Problem Definition --> Data --> Preprocessing --> Training --> Evaluation --> Deployment) is also the **attack surface map**. Every stage can be targeted.

- **Core tradeoffs**: Bias vs. variance, precision vs. recall, model complexity vs. interpretability. Understanding these helps you both build and break models.

- **Metrics matter**: Accuracy alone is misleading. Always consider precision, recall, and F1, especially for imbalanced datasets common in security (many benign events, few attacks).

- **Every ML model is an attack surface**. Adversarial examples, data poisoning, model stealing, model inversion, and membership inference are the major attack classes. You will study each in depth.

- **The shift from rules to models means the attack surface has shifted too.** Instead of finding logic bugs in code, attackers find statistical weaknesses in learned decision boundaries.

- **Offensive AI is not just about attacking ML -- it is about using ML as a weapon.** The same technology that powers defense can automate reconnaissance, craft phishing, generate exploits, and evade detection.

---

*Next up: Supervised Learning -- where we dive deep into learning from labeled data, covering classification, regression, and the algorithms that power most real-world ML applications.*
