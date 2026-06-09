# Naive Bayes

## What is Naive Bayes?

Naive Bayes is a **probability-based classification** algorithm. Instead of drawing lines or building trees, it uses **Bayes' Theorem** to calculate the probability of each class given the input features, then picks the class with the highest probability.

### The Analogy

Imagine you are a doctor. A patient walks in with a cough and a fever. You do not immediately know what disease they have, but you know from experience:

- 80% of flu patients have a cough
- 70% of flu patients have a fever
- Only 10% of cold patients have a fever
- 50% of cold patients have a cough

You also know that in general (before seeing any symptoms), flu is more common this time of year (60% of cases are flu, 40% are colds).

Using all of this information together, you can calculate: "Given this patient has BOTH a cough AND a fever, what is the probability it is flu vs cold?" That calculation is Bayes' Theorem, and that is exactly what Naive Bayes does.

---

## Bayes' Theorem Explained Simply

### The Formula

```
                       P(B | A) * P(A)
    P(A | B)  =  -------------------------
                          P(B)
```

In plain English:

```
                            How often B happens given A   *   How often A happens overall
    P(A given B)  =  ---------------------------------------------------------------
                                        How often B happens overall
```

### Breaking Down Each Part

| Symbol | Name | Meaning |
|--------|------|---------|
| P(A given B) | **Posterior** | What we want: the probability of class A AFTER seeing evidence B |
| P(B given A) | **Likelihood** | How likely is evidence B if class A is true |
| P(A) | **Prior** | How common is class A in general (before seeing evidence) |
| P(B) | **Evidence** | How common is evidence B overall |

### Medical Test Analogy (Worked Through)

A disease affects 1 in 1000 people. A test for this disease is:
- 99% accurate when you HAVE the disease (true positive rate)
- 95% accurate when you DO NOT have the disease (5% false positive rate)

You test positive. What is the probability you actually have the disease?

```
    P(Disease) = 0.001           (1 in 1000 people have it)
    P(No Disease) = 0.999
    P(Positive | Disease) = 0.99 (test catches 99% of sick people)
    P(Positive | No Disease) = 0.05 (5% false positive rate)

    P(Positive) = P(Pos|Disease)*P(Disease) + P(Pos|No Disease)*P(No Disease)
                = 0.99 * 0.001 + 0.05 * 0.999
                = 0.00099 + 0.04995
                = 0.05094

    P(Disease | Positive) = P(Pos|Disease) * P(Disease) / P(Positive)
                          = 0.99 * 0.001 / 0.05094
                          = 0.00099 / 0.05094
                          = 0.0194
                          = about 1.9%
```

**Even with a positive test, there is only a 1.9% chance you have the disease!** This is because the disease is so rare that most positive results are false positives.

> This is a famous result that trips up many people. The prior probability (how rare the disease is) matters enormously.

---

## Why "Naive"? The Independence Assumption

The "naive" part comes from a simplifying assumption: **Naive Bayes assumes all features are independent of each other.**

### What Independence Means

Independent: knowing one feature tells you nothing about another.

```
    Independent:     Knowing the email has "free" tells you nothing 
                     about whether it also has "click"

    NOT independent: If an email has "free", it is MORE likely to 
                     also have "click" and "offer" (they tend to 
                     appear together in spam)
```

### Why This Assumption is "Naive"

In the real world, features are almost NEVER truly independent:
- In emails, "free" and "offer" often appear together
- In network traffic, high packet rate and high connection count are correlated
- In malware, certain API calls tend to appear in groups

### Why It Still Works

Despite this clearly wrong assumption, Naive Bayes often works surprisingly well because:
1. The ranking of probabilities is often correct even if the exact values are off
2. The simplification makes the math tractable and fast
3. With enough data, the errors from the independence assumption tend to cancel out

```
    Real world:          Naive Bayes assumes:

    Features A and B     Features A and B
    are correlated       are independent
    
    P(A,B) != P(A)*P(B)  P(A,B) = P(A) * P(B)
    
    This is wrong, but   ...and it STILL gets the
    Naive Bayes assumes  right class most of the time.
    it anyway...
```

---

## Types of Naive Bayes

| Type | Feature Type | Example Use Case |
|------|-------------|-----------------|
| **Gaussian NB** | Continuous numbers (assumes bell curve distribution) | Network traffic features (packet size, duration) |
| **Multinomial NB** | Word counts / frequencies | Text classification (spam detection, document categorization) |
| **Bernoulli NB** | Binary features (yes/no, present/absent) | Whether specific words appear in an email (1 or 0) |

### When to Use Which

```
    Is your data continuous numbers?
    |
    +-- YES --> Gaussian Naive Bayes
    |           (e.g., packet size = 1452 bytes)
    |
    +-- NO  --> Are your features word counts?
                |
                +-- YES --> Multinomial Naive Bayes
                |           (e.g., "free" appears 3 times)
                |
                +-- NO  --> Bernoulli Naive Bayes
                            (e.g., does "free" appear? yes/no)
```

---

## Worked Example: Spam Detection

### The Setup

We have a training set of 10 emails. We want to classify a new email as **Spam** or **Not Spam (Ham)** based on the presence of three words: "free", "money", and "meeting".

### Training Data Summary

| | Total Emails | Contains "free" | Contains "money" | Contains "meeting" |
|-|-------------|-----------------|-------------------|--------------------|
| **Spam** | 4 | 3 (75%) | 3 (75%) | 1 (25%) |
| **Ham** | 6 | 1 (17%) | 0 (0%)* | 5 (83%) |

*We use 0.1 instead of 0 to avoid multiplying by zero -- this is called **Laplace smoothing**.

### Prior Probabilities

```
    P(Spam) = 4/10 = 0.4
    P(Ham)  = 6/10 = 0.6
```

### A New Email Arrives

The new email contains the words "free" and "money" but NOT "meeting."

### Step 1: Calculate P(features | Spam)

```
    P("free" | Spam)       = 3/4 = 0.75
    P("money" | Spam)      = 3/4 = 0.75
    P(NOT "meeting" | Spam) = 3/4 = 0.75   (since 1/4 have "meeting")

    Naive assumption (multiply them):
    P(features | Spam) = 0.75 * 0.75 * 0.75 = 0.4219
```

### Step 2: Calculate P(features | Ham)

```
    P("free" | Ham)        = 1/6 = 0.167
    P("money" | Ham)       = 0.1/6 = 0.017  (using Laplace smoothing)
    P(NOT "meeting" | Ham) = 1/6 = 0.167    (since 5/6 have "meeting")

    P(features | Ham) = 0.167 * 0.017 * 0.167 = 0.000474
```

### Step 3: Apply Bayes' Theorem

```
    P(Spam | features) proportional to P(features | Spam) * P(Spam)
                     = 0.4219 * 0.4 = 0.1688

    P(Ham | features)  proportional to P(features | Ham) * P(Ham)
                     = 0.000474 * 0.6 = 0.000284
```

### Step 4: Normalize to Get Probabilities

```
    Total = 0.1688 + 0.000284 = 0.169084

    P(Spam | features) = 0.1688 / 0.169084 = 0.9983  (99.8%)
    P(Ham | features)  = 0.000284 / 0.169084 = 0.0017 (0.2%)
```

### Result

```
    +----------------------------------------------+
    |  New Email: contains "free" and "money"      |
    |                                              |
    |  P(Spam) = 99.8%                             |
    |  P(Ham)  =  0.2%                             |
    |                                              |
    |  Classification: SPAM                        |
    +----------------------------------------------+
```

The email containing "free" and "money" (but not "meeting") is classified as spam with 99.8% confidence. This makes intuitive sense -- those words are very common in spam and rare in legitimate emails.

---

## Security Angle: Naive Bayes in Email Security

### Where Naive Bayes is Used

**1. Email Security Gateways**
- The original and most famous application of Naive Bayes in security
- SpamAssassin, early versions of Gmail spam filters, and many commercial products use or used Naive Bayes
- Classifies emails based on word frequencies, header analysis, and metadata

**2. Malware Classification**
- Classify PE files based on byte n-gram frequencies
- Features: sequences of bytes treated like "words" in a document
- Fast enough for real-time scanning

**3. Network Protocol Classification**
- Identify which application generated network traffic
- Features: packet sizes, timing patterns, header fields
- Used for traffic analysis and policy enforcement

**4. Log Analysis**
- Classify log entries by severity or attack type
- Features: keywords, source, timestamp patterns
- Helps SIEM systems prioritize alerts

### How Attackers Craft Emails to Bypass Naive Bayes

Since Naive Bayes relies on word probabilities, attackers manipulate the words in their emails:

**1. Good Word Attacks (Bayesian Poisoning)**

The attacker adds many legitimate-sounding words to dilute the spam signal:

```
    Original phishing email:
    "URGENT: Verify your account NOW! Click here for FREE gift!"
    
    Spam score: Very high (many spam words)

    Poisoned version:
    "Dear valued customer, regarding your quarterly financial report
     and the upcoming board meeting discussion about market analysis
     and strategic planning initiatives...
     
     [Hidden in small white text or at the bottom:]
     Please verify your account: [malicious link]"
    
    Spam score: Much lower (many legitimate words dilute the spam signal)
```

The math behind this:

```
    Without good words:
    P(Spam | "free","urgent","click") = very high

    With added good words:
    P(Spam | "free","urgent","click","quarterly","financial",
             "report","meeting","strategic","planning")

    The legitimate words ("quarterly", "financial", etc.) have
    high P(word | Ham), which pulls the overall probability
    toward Ham.
```

**2. Synonym Substitution**

| Spam Word (Blocked) | Synonym (Might Pass) |
|---------------------|---------------------|
| "free" | "complimentary", "no-cost", "gratis" |
| "click here" | "follow this link", "navigate to" |
| "urgent" | "time-sensitive", "immediate attention" |
| "winner" | "selected recipient", "chosen participant" |
| "password" | "credential", "access code", "passphrase" |

**3. Obfuscation Techniques**

| Technique | Example | Purpose |
|-----------|---------|---------|
| Character substitution | "fr33" instead of "free" | Avoid exact word matching |
| Zero-width characters | "f\u200bree" (invisible char inside) | Break tokenization |
| Image-based text | Render spam text as an image | Bypass text analysis entirely |
| Base64/encoding | Encode the message body | Evade word-level scanning |
| Homoglyphs | Use Cyrillic "a" instead of Latin "a" | Visually identical, different token |

**4. Training Data Poisoning (Adversarial Training)**

```
    Attack sequence:
    1. Attacker sends many emails containing attack keywords
       but with clearly legitimate content
    2. Recipients mark these as "Not Spam"
    3. The model retrains and learns that these keywords
       are associated with legitimate email
    4. Future phishing emails using those keywords are
       now classified as legitimate
    
    This is called "adversarial retraining" -- manipulating
    the model by influencing its training data.
```

### Why Naive Bayes is Vulnerable

| Vulnerability | Explanation |
|--------------|-------------|
| Word-level analysis | The model looks at individual words, not meaning or context |
| Independence assumption | Cannot detect suspicious word COMBINATIONS (e.g., "free" + "password" together) |
| Sensitivity to word frequencies | Adding many benign words can flip the classification |
| No understanding of intent | Cannot understand that a sentence is trying to trick the reader |
| Static vocabulary | New obfuscation techniques create "unknown" words the model cannot evaluate |

---

## Key Terminology

| Term | Definition |
|------|-----------|
| **Bayes' Theorem** | Mathematical formula that updates probability based on new evidence |
| **Prior probability** | The probability of a class before seeing any evidence -- P(A) |
| **Posterior probability** | The probability of a class AFTER seeing evidence -- P(A given B) |
| **Likelihood** | The probability of the evidence given a particular class -- P(B given A) |
| **Evidence** | The observed features/data -- P(B) |
| **Independence assumption** | The "naive" part: assumes features do not influence each other |
| **Laplace smoothing** | Adding a small count (usually 1) to avoid zero probabilities |
| **Multinomial** | Based on word/feature counts (how many times does "free" appear?) |
| **Bernoulli** | Based on presence/absence (does "free" appear at all? yes/no) |
| **Gaussian** | Assumes continuous features follow a bell curve (normal distribution) |
| **Bayesian poisoning** | Injecting legitimate words into spam to fool Bayesian classifiers |
| **Tokenization** | Splitting text into individual words/tokens for analysis |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|-----------|-----------|
| Very fast to train and predict | Independence assumption is rarely true |
| Works well with small datasets | Cannot learn relationships between features |
| Handles high-dimensional data (many features) well | Probability estimates are often poorly calibrated |
| Simple to implement and understand | Sensitive to irrelevant features |
| Performs surprisingly well for text classification | Struggles with feature combinations that matter |
| Naturally handles multiclass problems | Continuous features require distributional assumptions |
| Not sensitive to missing features | Easily fooled by good-word attacks |
| Provides probability estimates | Zero-frequency problem (needs smoothing) |

---

## Key Takeaways

1. **Naive Bayes uses probability to classify.** It calculates P(Class | Features) using Bayes' Theorem and picks the class with the highest probability.

2. **The "naive" assumption is that features are independent.** This is almost always wrong in practice, but the algorithm still works well -- especially for text classification.

3. **Bayes' Theorem has four parts:** prior (baseline probability), likelihood (how likely the evidence is for each class), evidence (how common the observation is), and posterior (what we want to know).

4. **Prior probability matters a lot.** The medical test example shows that even a highly accurate test can have mostly false positives if the condition is rare.

5. **In security, Naive Bayes powers spam filters and email gateways.** It classifies emails by word probabilities -- fast and effective for basic filtering.

6. **Attackers defeat Naive Bayes with "good word" attacks** -- flooding a phishing email with legitimate-looking text to dilute the spam score. They also use synonym substitution, character obfuscation, and image-based text.

7. **For the exam:** Understand Bayes' Theorem at a conceptual level, know why the algorithm is called "naive," be able to explain the spam detection use case, and describe how attackers use Bayesian poisoning and word substitution to evade Naive Bayes classifiers.
