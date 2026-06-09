# Logistic Regression

## What is Logistic Regression?

Despite its name, logistic regression is a **classification** algorithm, not a regression algorithm. It predicts which **category** something belongs to, not a continuous number.

### The Key Difference from Linear Regression

| Aspect | Linear Regression | Logistic Regression |
|--------|------------------|-------------------|
| Output | Any number (e.g., $225,000) | Probability between 0 and 1 |
| Task | Regression (predict a number) | Classification (predict a category) |
| Decision | No threshold needed | Uses a threshold (usually 0.5) |
| Example | "This house costs $225K" | "This email is 92% likely to be spam" |

### The Analogy

Think of logistic regression as a bouncer at a club. The bouncer looks at several features of each person (age, dress code, ID validity) and makes a **yes/no decision**: let them in or turn them away. But internally, the bouncer has a confidence level -- "I am 95% sure this person meets the criteria" or "I am only 30% sure." If the confidence is above 50%, they get in. Below 50%, they are turned away.

---

## The Sigmoid Function

The secret sauce of logistic regression is the **sigmoid function** (also called the logistic function). It takes any number and squashes it into a value between 0 and 1 -- perfect for representing probability.

### The Formula

```
    sigmoid(z) = 1 / (1 + e^(-z))

    Where:
    - z = w1*x1 + w2*x2 + ... + b  (the linear combination, same as linear regression)
    - e = Euler's number (approximately 2.718)
```

### What the Sigmoid Curve Looks Like

```
    Output (Probability)
    1.0 |                          _______________
        |                        /
    0.9 |                      /
        |                    /
    0.7 |                  /
        |                /
    0.5 |..............x..............  <-- Decision boundary
        |            /
    0.3 |          /
        |        /
    0.1 |      /
        |    /
    0.0 |___/
        +----+----+----+----+----+----+---
        -6   -4   -2    0    2    4    6
                         z
```

### How to Read the Sigmoid

| z value | sigmoid(z) | Interpretation |
|---------|-----------|----------------|
| -6 | ~0.002 | Almost certainly Class 0 (e.g., NOT spam) |
| -2 | ~0.12 | Probably Class 0 |
| 0 | 0.50 | Completely uncertain -- right on the boundary |
| +2 | ~0.88 | Probably Class 1 (e.g., spam) |
| +6 | ~0.998 | Almost certainly Class 1 |

The sigmoid converts the raw score (z) into a probability. Large positive z values push toward 1.0 (Class 1). Large negative z values push toward 0.0 (Class 0).

---

## Decision Boundary

The **decision boundary** is the threshold where the model switches from predicting one class to the other. Usually, this is set at 0.5.

```
    If sigmoid(z) >= 0.5  -->  Predict Class 1 (e.g., Spam, Malicious)
    If sigmoid(z) <  0.5  -->  Predict Class 0 (e.g., Not Spam, Benign)
```

### Visualizing the Decision Boundary (2 features)

```
    Feature 2
    |
    |  o o o o          Decision
    |    o o o o        Boundary
    |      o o o  |  x x x
    |        o o  |  x x x x
    |          o  |  x x x
    |             |  x x x x
    |             |  x x
    +-------------+-----------
                              Feature 1

    o = Class 0 (Benign)
    x = Class 1 (Malicious)
    | = Decision boundary line
```

### Adjusting the Threshold

You do not HAVE to use 0.5. In security, you might adjust the threshold:

| Threshold | Effect | When to Use |
|-----------|--------|-------------|
| 0.3 | More sensitive, catches more attacks, more false alarms | When missing an attack is very costly |
| 0.5 | Balanced | General purpose |
| 0.7 | More conservative, fewer false alarms, might miss attacks | When false alarms are very expensive |

---

## Binary vs Multiclass Classification

### Binary Classification (Two Classes)

Standard logistic regression handles **two classes**:
- Spam vs Not Spam
- Malware vs Benign
- Attack vs Normal

Output: a single probability. If P(Class 1) = 0.8, then P(Class 0) = 0.2.

### Multiclass Classification (Three or More Classes)

When you have more than two classes, you extend logistic regression using:

**One-vs-Rest (OvR):** Build a separate classifier for each class. "Is it Class A vs everything else?" "Is it Class B vs everything else?" Pick the class with the highest probability.

```
    Input: Network packet features

    Classifier 1: Normal vs [All Others]    --> P(Normal)  = 0.1
    Classifier 2: SQL Injection vs [All]    --> P(SQLi)    = 0.7  <-- Winner
    Classifier 3: DDoS vs [All Others]      --> P(DDoS)    = 0.2

    Prediction: SQL Injection (highest probability)
```

**Softmax Regression:** A generalized version that outputs probabilities across all classes at once, and they sum to 1.0.

---

## Worked Example: Phishing Email Classification

### The Setup

We want to classify emails as **phishing (1)** or **legitimate (0)** based on two features:
- x1 = Number of suspicious words ("urgent", "verify", "click now")
- x2 = Number of links in the email

### Training Data

| Email | Suspicious Words (x1) | Links (x2) | Label (y) |
|-------|----------------------|-------------|-----------|
| A | 0 | 1 | 0 (Legit) |
| B | 1 | 2 | 0 (Legit) |
| C | 3 | 4 | 1 (Phishing) |
| D | 5 | 6 | 1 (Phishing) |
| E | 4 | 5 | 1 (Phishing) |

### Step 1: The Model Learns Weights

After training (using gradient descent), suppose the model learns:
```
    w1 = 0.8   (weight for suspicious words)
    w2 = 0.5   (weight for links)
    b  = -3.0  (bias)
```

### Step 2: Classify a New Email

A new email arrives with 3 suspicious words and 3 links.

```
    Step 1: Compute z
    z = w1*x1 + w2*x2 + b
    z = 0.8(3) + 0.5(3) + (-3.0)
    z = 2.4 + 1.5 - 3.0
    z = 0.9

    Step 2: Apply sigmoid
    sigmoid(0.9) = 1 / (1 + e^(-0.9))
                 = 1 / (1 + 0.407)
                 = 1 / 1.407
                 = 0.711

    Step 3: Apply threshold (0.5)
    0.711 >= 0.5  -->  Predict: PHISHING
```

The model is 71.1% confident this is a phishing email.

### Step 3: Try Another Email

An email with 1 suspicious word and 1 link:

```
    z = 0.8(1) + 0.5(1) - 3.0 = -1.7
    sigmoid(-1.7) = 1 / (1 + e^(1.7)) = 1 / (1 + 5.47) = 0.154

    0.154 < 0.5  -->  Predict: LEGITIMATE
```

Only 15.4% chance of being phishing. Classified as legitimate.

```
    Visualization:
    
    Links (x2)
    7 |
    6 |                    P
    5 |                 P      P = Phishing
    4 |              P         L = Legitimate
    3 |           ?  <-- New email (Phishing, 71.1%)
    2 |        L
    1 |  L  ?  <-- New email (Legit, 15.4%)
    0 +--+--+--+--+--+--+--
      0  1  2  3  4  5  6
         Suspicious Words (x1)
```

---

## Security Angle: Logistic Regression in Security Tools

### Where Logistic Regression is Used

**1. Spam Filters**
- Email gateways use logistic regression to score incoming messages
- Features: sender reputation, word frequency, header anomalies, attachment types
- Output: probability of spam

**2. Intrusion Detection Systems (IDS)**
- Network-based IDS classifies traffic flows as normal or malicious
- Features: packet rate, byte ratio, protocol flags, connection duration
- Output: attack probability per flow

**3. Web Application Firewalls (WAFs)**
- Classifies HTTP requests as legitimate or malicious
- Features: URL length, special character count, SQL keywords present, encoding type
- Output: probability of attack (SQLi, XSS, etc.)

**4. Phishing URL Detection**
- Browser extensions and email gateways classify URLs
- Features: domain age, URL length, use of IP address, number of subdomains
- Output: probability of phishing

### How Attackers Evade Logistic Regression Classifiers

Understanding the model allows attackers to craft inputs that fall on the "benign" side of the decision boundary.

**1. Feature Manipulation**

```
    Original phishing email:
    - Suspicious words: 5  -->  z = 0.8(5) + 0.5(3) - 3.0 = 2.5  -->  sigmoid = 0.924
    - Links: 3                  DETECTED AS PHISHING

    Attacker's modified email:
    - Uses synonyms to reduce suspicious word count
    - Hides links using URL shorteners or image-based links
    - Suspicious words: 1  -->  z = 0.8(1) + 0.5(1) - 3.0 = -1.7  -->  sigmoid = 0.154
    - Links: 1                  CLASSIFIED AS LEGITIMATE
```

**2. Adversarial Crafting Techniques**

| Technique | Description | Example |
|-----------|-------------|---------|
| **Word substitution** | Replace flagged words with synonyms | "urgent" becomes "time-sensitive" |
| **Homoglyph attacks** | Replace characters with similar-looking Unicode | "a" replaced with Cyrillic "a" |
| **Content padding** | Add legitimate-looking text to dilute suspicious features | Appending a real news article to a phishing email |
| **Link obfuscation** | Hide malicious URLs behind URL shorteners or redirects | bit.ly/xyz instead of suspicious-domain.com |
| **HTML tricks** | Use CSS to hide text visible only to the classifier | White text on white background adds benign words |

**3. Threshold Exploitation**

If the attacker knows the decision threshold, they can aim for a score just below it:
```
    Threshold = 0.5

    Attacker crafts input so:
    sigmoid(z) = 0.49  -->  Classified as LEGITIMATE

    The email is still very suspicious (49% confidence)
    but it slips past the binary threshold.
```

**4. Model Stealing**

Logistic regression models are particularly easy to steal because:
- They have relatively few parameters (one weight per feature + bias)
- An attacker can query the model with chosen inputs
- With enough query-response pairs, they can solve for the weights mathematically
- Once they have the weights, they know the exact decision boundary

---

## Key Terminology

| Term | Definition |
|------|-----------|
| **Sigmoid function** | Mathematical function that squashes any number into the range (0, 1) |
| **Decision boundary** | The threshold (usually 0.5) that separates class predictions |
| **Binary classification** | Classifying into exactly two categories |
| **Multiclass classification** | Classifying into three or more categories |
| **Softmax** | Generalization of sigmoid for multiple classes; outputs probabilities that sum to 1 |
| **Log loss (cross-entropy)** | The cost function used in logistic regression (not MSE) |
| **One-vs-Rest (OvR)** | Strategy to handle multiclass by training one binary classifier per class |
| **Threshold** | The probability cutoff for making a class prediction |
| **True Positive (TP)** | Model correctly predicts the positive class (e.g., correctly flags phishing) |
| **False Positive (FP)** | Model incorrectly predicts positive (e.g., flags a legit email as phishing) |
| **False Negative (FN)** | Model misses a positive case (e.g., phishing email gets through) |
| **Precision** | Of all predicted positives, how many were actually positive: TP / (TP + FP) |
| **Recall** | Of all actual positives, how many did we catch: TP / (TP + FN) |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|-----------|-----------|
| Outputs calibrated probabilities (not just class labels) | Assumes a linear decision boundary |
| Simple, fast, and easy to interpret | Cannot capture complex non-linear relationships |
| Works well when classes are linearly separable | Struggles when features are highly correlated |
| Low risk of overfitting (especially with regularization) | Not ideal for very high-dimensional data without regularization |
| Each weight shows feature importance and direction | Requires feature engineering for non-linear problems |
| Widely used and well-understood | Performance ceiling is lower than complex models |

---

## Key Takeaways

1. **Logistic regression is for classification, not regression.** The name is misleading. It predicts the probability of belonging to a class.

2. **The sigmoid function is the core.** It transforms a raw score into a probability between 0 and 1. Know what the S-curve looks like and how to interpret it.

3. **The decision boundary is adjustable.** In security, you often lower the threshold (e.g., 0.3) to catch more attacks, accepting more false positives.

4. **Attackers evade logistic regression classifiers** by manipulating input features to push their score below the decision threshold. Techniques include word substitution, homoglyph attacks, and content padding.

5. **Logistic regression models are easy to steal** because they have few parameters. An attacker with query access can reconstruct the model.

6. **Precision vs Recall tradeoff is critical in security.** High recall (catch every attack) comes at the cost of precision (more false alarms). The threshold controls this tradeoff.

7. **For the exam:** Understand the sigmoid function, how the decision boundary works, the difference between binary and multiclass classification, and how attackers manipulate features to evade logistic regression-based spam filters and IDS.
