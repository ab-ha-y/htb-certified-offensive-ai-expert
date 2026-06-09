# Supervised Learning Algorithms

## What is Supervised Learning?

### The Teacher-Student Analogy

Imagine a teacher showing a student flashcards. On the front of each card is a picture of an animal. On the back is the answer -- "cat," "dog," or "bird." The teacher shows the student hundreds of these cards. Over time, the student learns to recognize patterns: pointy ears and whiskers probably mean "cat," floppy ears and a snout probably mean "dog."

After enough practice, the teacher gives the student a NEW picture -- one they have never seen before -- and asks: "What animal is this?" The student uses everything they learned from the flashcards to make a prediction.

That is supervised learning in a nutshell.

- The **flashcards** are the training data
- The **pictures** are the input features
- The **answers on the back** are the labels
- The **student** is the model
- The **teacher** is the training process
- The **new picture** is unseen/test data

```
    SUPERVISED LEARNING FLOW
    ========================

    Training Phase:
    +-----------+     +-----------+     +--------+
    | Labeled   | --> | Learning  | --> | Trained|
    | Data      |     | Algorithm |     | Model  |
    | (X, y)    |     |           |     |        |
    +-----------+     +-----------+     +--------+

    Prediction Phase:
    +-----------+     +--------+     +------------+
    | New Data  | --> | Trained| --> | Prediction |
    | (X only)  |     | Model  |     | (y_hat)    |
    +-----------+     +--------+     +------------+
```

---

## Labeled Data Explained

In supervised learning, every piece of training data comes with a **label** -- the correct answer. This is what makes it "supervised." Someone (a human, usually) has already gone through the data and tagged each example.

### Examples of Labeled Data

| Domain | Input Features (X) | Label (y) |
|--------|-------------------|-----------|
| Email Security | Word count, sender domain, link count | Spam / Not Spam |
| Malware Detection | File size, API calls, entropy | Malicious / Benign |
| House Pricing | Square footage, bedrooms, location | Price in dollars |
| Network Traffic | Packet size, port, protocol, frequency | Normal / Attack |

### How Labels Are Created

Labels come from:
1. **Manual annotation** -- Human experts review and tag data (expensive but accurate)
2. **Existing records** -- Historical data that already has outcomes (e.g., past transactions marked as fraud)
3. **Automated labeling** -- Using rules or other systems to generate labels (faster but noisier)
4. **Crowdsourcing** -- Distributing labeling tasks to many workers

> **Key insight:** The quality of your labels determines the ceiling of your model's performance. Garbage labels in = garbage predictions out.

---

## Classification vs Regression

Supervised learning problems fall into two main categories:

### Classification: Predicting a Category

The model outputs a **discrete class label** -- one of a fixed set of categories.

```
    Input: Email features
                              +---> "Spam"
    Classification Model -----|
                              +---> "Not Spam"
```

**Examples:**
- Is this file malware or benign? (Binary classification -- 2 classes)
- What type of attack is this? SQL injection, XSS, or DDoS? (Multiclass -- 3+ classes)
- Which vulnerabilities apply to this code? (Multilabel -- multiple labels can be true)

### Regression: Predicting a Number

The model outputs a **continuous numerical value**.

```
    Input: Network features
                              +---> 87.3 (risk score)
    Regression Model ---------|
                              +---> 12.1 (risk score)
```

**Examples:**
- What is the estimated damage cost of this breach?
- How many minutes until this server is overloaded?
- What is the probability score (0.0 to 1.0) of this being an attack?

### Side-by-Side Comparison

| Aspect | Classification | Regression |
|--------|---------------|------------|
| Output type | Category/class label | Continuous number |
| Example output | "Malware" or "Benign" | 0.87 risk score |
| Evaluation metrics | Accuracy, Precision, Recall, F1 | MSE, RMSE, MAE, R-squared |
| Algorithms | Logistic Regression, Decision Trees, SVM, Naive Bayes | Linear Regression, Polynomial Regression |
| Security use case | Classify traffic as attack/normal | Predict severity score of incident |

---

## Train/Test Split Concept

You never want to test a student on the exact same questions they studied. That would only measure memorization, not understanding. The same idea applies to ML models.

### How It Works

```
    Full Dataset (1000 samples)
    ============================================
    |  Training Set (800 samples - 80%)  | Test |
    |  Used to LEARN patterns            | Set  |
    |                                    | 200  |
    |                                    | 20%  |
    |                                    | Used |
    |                                    | to   |
    |                                    | TEST |
    ============================================
         Model trains here          Evaluate here
         (sees labels)              (hides labels,
                                     then checks)
```

### Why Split the Data?

1. **Prevents overfitting** -- The model might memorize training data instead of learning general patterns
2. **Estimates real-world performance** -- Test data simulates data the model has never seen
3. **Builds trust** -- You can report honest accuracy numbers

### Common Split Ratios

| Split | Training | Testing | When to Use |
|-------|----------|---------|-------------|
| 80/20 | 80% | 20% | Most common, good default |
| 70/30 | 70% | 30% | When you want more test confidence |
| 90/10 | 90% | 10% | When data is scarce |

### Validation Set (Bonus Concept)

Sometimes you split into THREE parts:

```
    |--- Training (60-70%) ---|--- Validation (15-20%) ---|--- Test (10-20%) ---|
         Learn patterns            Tune hyperparameters        Final evaluation
```

The **validation set** is used to tune the model's settings (hyperparameters) without touching the test set. The test set stays locked away until the very end.

---

## Common Supervised Learning Algorithms

| Algorithm | Type | How It Works (One Sentence) | Best For | Complexity |
|-----------|------|---------------------------|----------|------------|
| Linear Regression | Regression | Fits a straight line through data points | Predicting continuous values with linear relationships | Low |
| Logistic Regression | Classification | Uses a sigmoid curve to output probabilities for classes | Binary classification problems | Low |
| Decision Trees | Both | Asks a series of yes/no questions to reach a decision | Interpretable models, mixed data types | Medium |
| Random Forest | Both | Combines many decision trees and takes a vote | High accuracy with less overfitting | Medium |
| Naive Bayes | Classification | Uses probability theory (Bayes' theorem) to classify | Text classification, spam detection | Low |
| Support Vector Machine (SVM) | Both | Finds the widest separating boundary between classes | High-dimensional data, clear margins | Medium-High |
| k-Nearest Neighbors (KNN) | Both | Looks at the k closest data points and follows the majority | Simple problems, small datasets | Low |
| Neural Networks | Both | Layers of interconnected nodes that learn complex patterns | Complex patterns, large datasets | High |
| Gradient Boosting (XGBoost) | Both | Sequentially builds trees that fix previous trees' errors | Competitions, tabular data | Medium-High |

---

## Security Angle: Supervised Learning in Cybersecurity

### How Defenders Use Supervised Learning

Supervised learning is the backbone of many security tools:

**1. Malware Detection**
- Train on labeled samples of known malware and known clean files
- Features: API calls, file entropy, section sizes, imported libraries
- Model learns to classify new files as malicious or benign

**2. Intrusion Detection Systems (IDS)**
- Train on labeled network traffic (normal vs. various attack types)
- Features: packet size, protocol, port numbers, flow duration, byte counts
- Model flags suspicious traffic in real time

**3. Phishing Detection**
- Train on labeled URLs and emails (phishing vs. legitimate)
- Features: URL length, domain age, number of special characters, SSL status
- Model scores incoming emails and URLs

**4. User Behavior Analytics (UBA)**
- Train on labeled user activity (normal vs. compromised accounts)
- Features: login times, accessed resources, data transfer volumes
- Model detects insider threats and account takeover

### How Attackers Evade Supervised Models

Understanding how these models work is essential for offensive security:

**1. Adversarial Examples**
- Attackers craft inputs that are deliberately designed to fool the model
- Small, carefully calculated perturbations that change the model's prediction
- Example: Adding specific bytes to malware so the classifier thinks it is benign

**2. Feature Manipulation**
- If the attacker knows what features the model uses, they can manipulate them
- Example: Padding malware with benign API calls to shift feature distributions
- Example: Slowing attack traffic to look like normal browsing patterns

**3. Model Evasion**
- Training a substitute model to understand the defender's decision boundary
- Then crafting inputs that sit just on the other side of that boundary
- This works even without direct access to the target model (black-box attacks)

**4. Data Poisoning**
- Injecting malicious samples into the training data
- If the attacker can influence what data the model trains on, they can create blind spots
- Example: Submitting benign-looking malware samples to VirusTotal to pollute datasets

**5. Concept Drift Exploitation**
- Attack patterns change over time, but the model was trained on old data
- Attackers create novel attack patterns that the model has never seen
- The model's accuracy degrades as the real world drifts from the training data

```
    ATTACKER vs DEFENDER MODEL
    ==========================

    Defender trains model:
    [Malware Samples] + [Benign Samples] --> [Classifier]

    Attacker evades model:
    [Malware] + [Carefully Added Noise] --> [Classifier] --> "Benign" (wrong!)

    The classifier is fooled because the noise shifts the
    input across the decision boundary.
```

---

## Key Terminology

| Term | Definition |
|------|-----------|
| **Features (X)** | The input variables the model uses to make predictions (e.g., file size, packet count) |
| **Label (y)** | The correct answer/output associated with each training example |
| **Training** | The process of feeding labeled data to an algorithm so it learns patterns |
| **Inference** | Using a trained model to make predictions on new, unseen data |
| **Overfitting** | When a model memorizes training data and performs poorly on new data |
| **Underfitting** | When a model is too simple to capture the underlying patterns |
| **Hyperparameters** | Settings you choose before training (learning rate, tree depth, etc.) |
| **Ground Truth** | The actual correct label -- what really happened |
| **Generalization** | A model's ability to perform well on data it has never seen |
| **Epoch** | One complete pass through the entire training dataset |
| **Bias** | Systematic error from wrong assumptions (leads to underfitting) |
| **Variance** | Sensitivity to small fluctuations in training data (leads to overfitting) |

---

## Strengths and Weaknesses of Supervised Learning

| Strengths | Weaknesses |
|-----------|-----------|
| Well-understood and widely studied | Requires large amounts of labeled data |
| Clear evaluation metrics (accuracy, F1, etc.) | Labeling data is expensive and time-consuming |
| Works well when labeled data is available | Cannot discover patterns not represented in labels |
| Many mature algorithms and tools available | Vulnerable to adversarial attacks |
| Can achieve high accuracy on well-defined problems | Performance degrades with concept drift |
| Easy to explain results to stakeholders | May learn biases present in training data |

---

## Key Takeaways

1. **Supervised learning = learning from labeled examples.** The model sees input-output pairs during training and learns to predict outputs for new inputs.

2. **Classification predicts categories; regression predicts numbers.** Know which one you need before choosing an algorithm.

3. **Always split your data into training and test sets.** Never evaluate a model on the same data it trained on -- that measures memorization, not learning.

4. **Label quality matters more than quantity.** A smaller dataset with accurate labels often beats a huge dataset with noisy labels.

5. **In security, supervised models are both sword and shield.** Defenders use them for detection; attackers study them to find evasion strategies.

6. **Models are not static.** Attack patterns evolve, and models must be retrained regularly to stay effective. An outdated model is a vulnerable model.

7. **For the exam:** Understand the difference between classification and regression, know what labeled data means, and be able to explain how adversarial examples can fool supervised classifiers.
