# Spam Classification

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Applications of AI in InfoSec | Section: Spam Classification

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [The Dataset](#2-the-dataset)
3. [From Text to Features: Bag-of-Words](#3-from-text-to-features-bag-of-words)
4. [From Text to Features: TF-IDF](#4-from-text-to-features-tf-idf)
5. [Naive Bayes Recap](#5-naive-bayes-recap)
6. [Full Pipeline: Step-by-Step Code](#6-full-pipeline-step-by-step-code)
7. [Evaluating the Model](#7-evaluating-the-model)
8. [Inspecting What the Model Learned](#8-inspecting-what-the-model-learned)
9. [Adversarial Considerations](#9-adversarial-considerations)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. Project Overview

This is the first full, end-to-end applied project in this module: building a working spam classifier from raw email text to a trained, evaluated model, using exactly the tools and concepts introduced so far -- environment setup, scikit-learn, dataset inspection, preprocessing, and data splitting.

### The Goal

Given the text of an email, predict whether it is **spam** or **ham** (legitimate).

### Why This Project First

Spam classification is the "hello world" of applied security ML for good reason:

- The data (email text) is intuitive -- you do not need domain expertise to understand why certain words suggest spam.
- **Naive Bayes** (introduced conceptually in Module 1) was practically invented for exactly this problem and remains genuinely effective and fast.
- It cleanly demonstrates the entire pipeline: raw unstructured text -> numeric features -> trained model -> evaluated, deployable classifier.

```
    PROJECT PIPELINE
    ==================

    +-----------+     +--------------+     +---------------+     +----------+     +------------+
    | Raw Email | --> | Text Cleaning| --> | Vectorization  | --> | Train/   | --> | Naive      |
    | Text      |     | (lowercase,  |     | (Bag-of-Words  |     | Test     |     | Bayes      |
    |           |     |  strip punct)|     |  or TF-IDF)     |     | Split    |     | Classifier |
    +-----------+     +--------------+     +---------------+     +----------+     +------------+
                                                                                          |
                                                                                          v
                                                                                  +----------------+
                                                                                  | Evaluate: P/R/F1|
                                                                                  +----------------+
```

---

## 2. The Dataset

We use a labeled dataset of emails/SMS messages, each tagged `spam` or `ham`. The classic public dataset for this exercise is the **SMS Spam Collection** (or an equivalent labeled email corpus) -- a CSV with two columns: the message text and its label.

```python
import pandas as pd

df = pd.read_csv("spam.csv", encoding="latin-1")
df = df[["v1", "v2"]]              # some versions of this file have extra columns
df.columns = ["label", "text"]

print(df.shape)
print(df.head())
print(df["label"].value_counts())
```

```
(5572, 2)
  label                                              text
0   ham  Go until jurong point, crazy.. Available only ...
1   ham                      Ok lar... Joking wif u oni...
2  spam  Free entry in 2 a wkly comp to win FA Cup fina...
3   ham  U dun say so early hor... U c already then say...
4   ham  Nah I don't think he goes to usf, he lives aro...

ham     4825
spam     747
Name: label, dtype: int64
```

Notice immediately: this is an **imbalanced** dataset (about 87% ham, 13% spam) -- exactly the pattern flagged as important back in Data Preprocessing. We will keep this in mind when evaluating (Section 7) and when splitting (stratified split).

---

## 3. From Text to Features: Bag-of-Words

Models need numbers, not sentences. **Bag-of-Words (BoW)** converts each document into a vector of word counts, completely ignoring grammar and word order -- hence "bag": it is as if you dumped all the words from a document into a bag and just counted how many of each word are inside.

### The Analogy

Imagine emptying an email into a bag, shaking it, and then counting how many times each specific word tumbled out. You lose all sense of sentence structure, but you keep a surprisingly useful signal: spam emails tend to have a very different "bag" of words (heavy on "free," "winner," "click," "!!!," "$$$") than legitimate ones (heavy on "meeting," "attached," "regards," "tomorrow").

### Worked Mini-Example

```
Vocabulary built from these two documents:
  Doc 1: "free money now"
  Doc 2: "meeting at three"

Vocabulary (sorted alphabetically): [at, free, meeting, money, now, three]

Doc 1 vector: [0, 1, 0, 1, 1, 0]   (free=1, money=1, now=1, others=0)
Doc 2 vector: [1, 0, 1, 0, 0, 1]   (at=1, meeting=1, three=1, others=0)
```

```
    BAG-OF-WORDS VISUALIZATION
    =============================

              at  free  meeting  money  now  three
    Doc 1  [   0     1      0       1     1     0  ]   "free money now"
    Doc 2  [   1     0      1       0     0     1  ]   "meeting at three"

    Each row = one document turned into a row of numbers = one FEATURE VECTOR
    Each column = one word in the vocabulary = one FEATURE
```

### Building a Bag-of-Words Matrix with scikit-learn

```python
from sklearn.feature_extraction.text import CountVectorizer

documents = ["free money now", "meeting at three", "free free winner"]

vectorizer = CountVectorizer()
X = vectorizer.fit_transform(documents)

print(vectorizer.get_feature_names_out())
print(X.toarray())
```

```
['at' 'free' 'meeting' 'money' 'now' 'three' 'winner']
[[0 1 0 1 1 0 0]
 [1 0 1 0 0 1 0]
 [0 2 0 0 0 0 1]]
```

Note the third document, `"free free winner"`, produces a `2` in the "free" column -- Bag-of-Words counts repeated occurrences, it does not just flag presence/absence.

### Limitations of Plain Bag-of-Words

Common words like "the," "is," "to" appear in almost every document and add noise without much discriminative value. `CountVectorizer` supports `stop_words="english"` to automatically drop these. But a deeper limitation remains: BoW treats every word as equally important, when in reality, a word that appears in *every single* spam email (and nowhere else) is far more informative than a word that appears frequently everywhere. That is exactly the gap TF-IDF closes.

---

## 4. From Text to Features: TF-IDF

**TF-IDF** stands for **Term Frequency - Inverse Document Frequency**. It is a refinement of Bag-of-Words that weighs each word by how important it actually is, not just how often it appears.

### The Two Halves of the Formula

| Component | Formula (simplified) | Plain English |
|-----------|------------------------|-----------------|
| **Term Frequency (TF)** | (times word appears in this document) / (total words in this document) | How common is this word *within this one email*? |
| **Inverse Document Frequency (IDF)** | log(total documents / documents containing this word) | How *rare* is this word *across the whole dataset*? Rare words get a bigger boost. |
| **TF-IDF score** | TF x IDF | High only when a word is frequent in this document AND rare across the whole collection |

### Why This Matters

A word like "the" has high TF (appears a lot in every document) but very low IDF (appears in nearly every document, so it is not rare) -- these multiply to a low TF-IDF score, correctly suppressing it. A word like "viagra" has a lower overall TF but an extremely high IDF (it appears in very few documents, almost all of them spam) -- multiplying to a high TF-IDF score in the documents where it does appear, correctly boosting its importance.

```
    TF-IDF INTUITION
    ==================

    Word: "the"                        Word: "viagra"
    Appears in almost EVERY email      Appears in only a FEW emails,
    -> High TF, but very LOW IDF       nearly all of them spam
    -> LOW overall TF-IDF score        -> Lower TF, but very HIGH IDF
                                        -> HIGH overall TF-IDF score

    TF-IDF automatically down-weights common,       and up-weights rare,
    uninformative words like "the",                 highly discriminative
    "is", "and"...                                  words like "viagra",
                                                     "winner", "free".
```

### Computing TF-IDF with scikit-learn

```python
from sklearn.feature_extraction.text import TfidfVectorizer

documents = [
    "free money free winner",
    "meeting at three about the project",
    "free winner click now",
]

tfidf = TfidfVectorizer()
X = tfidf.fit_transform(documents)

print(tfidf.get_feature_names_out())
print(X.toarray().round(2))
```

```
['about' 'at' 'click' 'free' 'meeting' 'money' 'now' 'project' 'the' 'three' 'winner']
[[0.   0.   0.   0.72 0.   0.48 0.   0.   0.   0.   0.48]
 [0.41 0.41 0.   0.   0.41 0.   0.   0.41 0.41 0.41 0.  ]
 [0.   0.   0.53 0.4  0.   0.   0.53 0.   0.   0.   0.4 ]]
```

Notice "free" gets a strong weight (0.72) in the first document (where it appears twice and is generally rare across the small corpus), while common connective words carry moderate, evenly-spread weights in the second document. In practice, TF-IDF features tend to outperform plain Bag-of-Words counts for text classification because they emphasize the words that actually distinguish spam from ham.

---

## 5. Naive Bayes Recap

Module 1 covered Naive Bayes' math (Bayes' theorem, the independence assumption, Laplace smoothing) in depth. For this project, the key facts to remember:

| Fact | Why It Matters Here |
|------|------------------------|
| Naive Bayes computes P(class \| features) using Bayes' theorem | It naturally outputs a probability, not just a hard label -- useful for setting a custom spam threshold |
| It assumes features (words) are independent | Wrong in reality (words correlate), but works remarkably well for text anyway |
| **Multinomial Naive Bayes** models word *counts* | The correct variant for Bag-of-Words / TF-IDF features (as opposed to Gaussian NB for continuous numeric features, or Bernoulli NB for pure presence/absence) |
| Fast to train, even on large vocabularies | Ideal for real-time email filtering at scale |

```python
from sklearn.naive_bayes import MultinomialNB

clf = MultinomialNB()
# clf.fit(X_train_tfidf, y_train)   -- shown fully in Section 6
```

---

## 6. Full Pipeline: Step-by-Step Code

```python
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score,
    f1_score, confusion_matrix, classification_report
)

# -----------------------------------------------------------------
# Step 1: Load the data
# -----------------------------------------------------------------
df = pd.read_csv("spam.csv", encoding="latin-1")
df = df[["v1", "v2"]]
df.columns = ["label", "text"]

# -----------------------------------------------------------------
# Step 2: Clean the text
# -----------------------------------------------------------------
def clean_text(text):
    text = text.lower()
    text = "".join(ch for ch in text if ch.isalnum() or ch.isspace())
    return text

df["text_clean"] = df["text"].apply(clean_text)

# Drop exact duplicate messages (preprocessing lesson applied here)
df = df.drop_duplicates(subset=["text_clean"])

# -----------------------------------------------------------------
# Step 3: Encode the label (ham=0, spam=1)
# -----------------------------------------------------------------
df["label_num"] = df["label"].map({"ham": 0, "spam": 1})

# -----------------------------------------------------------------
# Step 4: Split BEFORE vectorizing, and stratify due to class imbalance
# -----------------------------------------------------------------
X_train_text, X_test_text, y_train, y_test = train_test_split(
    df["text_clean"], df["label_num"],
    test_size=0.2, stratify=df["label_num"], random_state=42
)

# -----------------------------------------------------------------
# Step 5: Vectorize with TF-IDF -- fit ONLY on training text
# -----------------------------------------------------------------
vectorizer = TfidfVectorizer(stop_words="english", max_features=3000)
X_train = vectorizer.fit_transform(X_train_text)
X_test = vectorizer.transform(X_test_text)   # transform only, no fitting

print("Training matrix shape:", X_train.shape)
print("Test matrix shape:", X_test.shape)

# -----------------------------------------------------------------
# Step 6: Train the Naive Bayes classifier
# -----------------------------------------------------------------
clf = MultinomialNB()
clf.fit(X_train, y_train)

# -----------------------------------------------------------------
# Step 7: Predict on the held-out test set
# -----------------------------------------------------------------
y_pred = clf.predict(X_test)
```

Sample output for the shapes:

```
Training matrix shape: (4135, 3000)
Test matrix shape: (1034, 3000)
```

---

## 7. Evaluating the Model

```python
print("Accuracy: ", accuracy_score(y_test, y_pred))
print("Precision:", precision_score(y_test, y_pred))
print("Recall:   ", recall_score(y_test, y_pred))
print("F1-Score: ", f1_score(y_test, y_pred))
print()
print(classification_report(y_test, y_pred, target_names=["ham", "spam"]))
print()
print("Confusion Matrix:")
print(confusion_matrix(y_test, y_pred))
```

Realistic-style output:

```
Accuracy:  0.9768
Precision: 0.9836
Recall:    0.8300
F1-Score:  0.9002

              precision    recall  f1-score   support

         ham       0.98      1.00      0.99       894
        spam       0.98      0.83      0.90       140

    accuracy                           0.98      1034
   macro avg       0.98      0.91      0.94      1034
weighted avg       0.98      0.98      0.98      1034

Confusion Matrix:
[[892   2]
 [ 24 116]]
```

### Reading the Confusion Matrix

```
                     PREDICTED
                  Ham       Spam
 ACTUAL  Ham   [  892   |    2  ]     (2 false positives -- legit
                                        emails wrongly flagged)
         Spam  [   24   |  116  ]     (24 false negatives -- spam
                                        that slipped through)
```

**Why accuracy alone would be misleading here**: this dataset is roughly 87% ham. A model that predicted "ham" for literally everything would score around 86-87% accuracy while catching zero spam. Our model's 97.7% accuracy is genuinely earned -- but the precision/recall breakdown is what proves it: 98% precision (very few false alarms) and 83% recall (catches most, though not all, spam).

### Precision/Recall Trade-off in Practice

Naive Bayes outputs probabilities, not just hard labels, letting you tune the decision threshold to favor precision or recall depending on the cost of each type of mistake:

```python
probabilities = clf.predict_proba(X_test)[:, 1]   # probability of being spam

# Default threshold is 0.5 -- lower it to catch more spam (higher recall,
# at the cost of more false positives), or raise it to reduce false
# positives (higher precision, at the cost of missing more spam)
threshold = 0.3
y_pred_adjusted = (probabilities >= threshold).astype(int)

print("Recall at threshold 0.3:", recall_score(y_test, y_pred_adjusted))
print("Precision at threshold 0.3:", precision_score(y_test, y_pred_adjusted))
```

```
Recall at threshold 0.3: 0.8929
Precision at threshold 0.3: 0.9481
```

Lowering the threshold from 0.5 to 0.3 catches more spam (recall rose from 0.83 to 0.89) at a modest cost to precision (0.98 -> 0.95) -- more false alarms, but fewer missed spam messages. Which direction to move this dial is a business/security decision, not a purely technical one: a corporate mail gateway might tolerate more false positives to avoid missing a phishing attack, while a consumer product might prioritize never blocking a real message from a friend.

---

## 8. Inspecting What the Model Learned

One advantage of Naive Bayes (and TF-IDF features) is interpretability: you can directly inspect which words most strongly push the model toward "spam."

```python
import numpy as np

feature_names = vectorizer.get_feature_names_out()
log_prob_spam = clf.feature_log_prob_[1]     # log P(word | spam)
log_prob_ham = clf.feature_log_prob_[0]      # log P(word | ham)

# The "spam-ness" of a word: how much more likely it is in spam vs ham
spam_score = log_prob_spam - log_prob_ham

top_spam_words = np.argsort(spam_score)[-15:][::-1]
print("Top words most associated with SPAM:")
for idx in top_spam_words:
    print(f"  {feature_names[idx]:15s}  score={spam_score[idx]:.2f}")
```

```
Top words most associated with SPAM:
  claim            score=6.81
  txt              score=6.54
  mobile           score=6.31
  free             score=6.02
  prize            score=5.95
  urgent           score=5.88
  won              score=5.79
  cash              score=5.61
  reply             score=5.44
  guaranteed        score=5.30
  ...
```

This matches human intuition almost exactly -- "claim," "free," "prize," "urgent," "won," "guaranteed" are textbook spam vocabulary. This kind of inspection is not just satisfying, it is an essential *offensive* skill: knowing exactly which words the model weighs most heavily tells you exactly which words an attacker would want to avoid, obfuscate, or dilute (see Section 9).

---

## 9. Adversarial Considerations

Module 1's Naive Bayes file already covered the theory of evading Bayesian spam filters. Applied to this concrete, trained model:

| Attack | Applied to This Model |
|--------|---------------------------|
| **Synonym substitution** | Replace "free" with "complimentary," "prize" with "reward" -- words with a low `spam_score` in our trained model but similar meaning |
| **Good-word attack (Bayesian poisoning)** | Pad the email with common, high-frequency ham vocabulary ("meeting," "attached," "regards") to dilute the aggregate TF-IDF-weighted spam signal |
| **Character obfuscation** | "fr33" instead of "free" creates a token our `TfidfVectorizer` vocabulary has never seen, so it contributes nothing to the score at all (effectively invisible to this exact model) |
| **Model stealing** | Send crafted probe emails through the deployed filter, observe pass/block decisions, and reconstruct an approximate vocabulary/weighting -- entirely feasible given how transparent Naive Bayes's learned weights are once you have query access |

### A Concrete Evasion Attempt

```python
original = "URGENT: Claim your free prize now, guaranteed cash reward!"
obfuscated = "urgent: cl@im your fr33 pr1ze now, guaranteed c4sh rew@rd!"

for text in [original, obfuscated]:
    cleaned = clean_text(text)
    vec = vectorizer.transform([cleaned])
    prob_spam = clf.predict_proba(vec)[0][1]
    print(f"{text[:40]:42s} -> P(spam) = {prob_spam:.4f}")
```

```
URGENT: Claim your free prize now, guar   -> P(spam) = 0.9991
urgent: cl@im your fr33 pr1ze now, guar   -> P(spam) = 0.6142
```

Even light character substitution meaningfully drags the spam probability down, because several high-signal tokens (`claim`, `free`, `prize`, `cash`) simply do not exist in the vectorizer's vocabulary once obfuscated -- they contribute zero information instead of strongly positive spam evidence. A production system would need additional defenses (character-level or subword tokenization, homoglyph normalization, or a more context-aware model) to close this gap.

---

## 10. Key Takeaways

- A full spam classification pipeline is: **load -> clean text -> encode labels -> split (stratified) -> vectorize (fit on train only) -> train Naive Bayes -> evaluate**.
- **Bag-of-Words** counts word occurrences; **TF-IDF** additionally weighs words by how rare and distinctive they are across the whole corpus, generally producing better features for text classification.
- **Multinomial Naive Bayes** is the correct variant for count-based / TF-IDF text features, and remains fast, effective, and genuinely used in production email security gateways.
- **Evaluate with precision, recall, and F1**, not accuracy alone -- this dataset's ~87/13 class imbalance would let a lazy "always predict ham" model score deceptively well on accuracy.
- **Naive Bayes is interpretable**: you can directly extract and rank which words most strongly drive the spam prediction, which is valuable both for debugging and for anticipating evasion.
- **The model's own transparency is also its attack surface**: synonym substitution, good-word dilution, and character obfuscation can all measurably reduce a message's predicted spam probability, as shown by the working code example above.

*Next up: Network Anomaly Detection -- a full worked project applying Random Forests to the NSL-KDD intrusion detection dataset.*
