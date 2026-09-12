# Membership Inference Attacks

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Privacy | Section: Membership Inference Attacks

---

## Table of Contents

1. [What Is Membership Inference?](#1-what-is-membership-inference)
2. [Why Models Leak Membership: Overfitting as the Root Cause](#2-why-models-leak-membership-overfitting-as-the-root-cause)
3. [The Confidence-Gap Signal](#3-the-confidence-gap-signal)
4. [The Shadow Model Methodology](#4-the-shadow-model-methodology)
5. [Worked Numeric Example](#5-worked-numeric-example)
6. [Signal Types Used in Membership Inference](#6-signal-types-used-in-membership-inference)
7. [Simplified Attack Walkthrough](#7-simplified-attack-walkthrough)
8. [Defenses](#8-defenses)
9. [Privacy Angle -- Why This Matters](#9-privacy-angle----why-this-matters)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. What Is Membership Inference?

**Membership Inference Attack (MIA)**: an attack where someone tries to determine whether a *specific, known data record* was used to train a machine learning model, just by observing (or querying) that model.

### The Analogy

Imagine a teacher who graded thousands of practice exams during the school year, then also graded the final exam. If you showed the teacher any single exam sheet today, could they tell you whether they had graded it before, months ago, versus a completely brand-new sheet they are seeing for the first time?

A good teacher who genuinely learned the *subject* (rather than the specific exam sheets) probably could not tell the difference -- both a memorized old exam and a fresh new one would feel equally familiar because the teacher understands the material generally.

But a teacher who **crammed and memorized the specific practice exams** (rather than learning the underlying subject) would recognize the practice sheets instantly -- the handwriting, the exact wording, the specific mistakes -- while a new exam would feel unfamiliar and harder to grade confidently.

Machine learning models behave the same way. A model that has **memorized** parts of its training data reacts differently -- more "confidently," more "smoothly," with lower error -- to data it was trained on than to new data it has never encountered. Membership inference exploits that behavioral difference to answer one yes/no question:

> "Was *this exact record* part of the training set, or not?"

### Formal Definition

Given:
- A **target model** `f` trained on some training set `D_train` (unknown to the attacker).
- A **candidate record** `x` (a specific data point the attacker is curious about, e.g., "Alice's medical record").
- Access to the target model, typically only via its outputs (predictions, confidence scores) -- this is called **black-box access**.

The attacker's goal is to build a function `A(f, x)` that outputs:
- `IN`  -- "x was probably in `D_train`"
- `OUT` -- "x was probably not in `D_train`"

This is a **binary classification problem** where the attacker is classifying *the model's relationship to a data point*, not classifying the data point itself.

```
   +------------------+          +---------------------+
   |  Candidate       |          |                      |
   |  Record  x       | -------> |   TARGET MODEL  f    |
   |  ("Alice's data")|          |  (already trained)   |
   +------------------+          +----------+-----------+
                                             |
                                   confidence scores /
                                   predictions on x
                                             |
                                             v
                                  +-----------------------+
                                  |   MEMBERSHIP          |
                                  |   INFERENCE ATTACK     |
                                  |   A(f, x) --> IN / OUT|
                                  +-----------------------+
```

---

## 2. Why Models Leak Membership: Overfitting as the Root Cause

Recall from Module 1 (Fundamentals of AI): **overfitting** happens when a model memorizes quirks of its training data instead of learning general patterns. A model minimizes a **loss function** (a number that measures how wrong its predictions are) during training. Training pushes the loss on *training* examples down, down, down -- sometimes all the way to nearly zero.

If a model is overfit:
- On **training data**: predictions are highly confident, loss is very low, the model has essentially "seen the answer key" and memorized it.
- On **unseen (test) data**: predictions are less confident, loss is higher, because the model never adjusted its parameters specifically to fit that exact point.

This gap between "how well the model does on data it trained on" versus "how well it does on data it never saw" is called the **generalization gap**. Membership inference is fundamentally an attack that **measures the generalization gap for a single data point** and uses it as a leak of information about training set membership.

```
                    LOSS / CONFIDENCE BEHAVIOR

    Training Data (member)          Unseen Data (non-member)
    -----------------------          -------------------------
    Loss:        very low            Loss:        higher
    Confidence:  very high (~0.99)   Confidence:  moderate (~0.55-0.75)
    Prediction:  "certain"           Prediction:  "hesitant"

         |                                  |
         |         THE GAP BETWEEN THESE TWO IS
         |         THE MEMBERSHIP INFERENCE SIGNAL
         v                                  v
```

**Key insight**: The *more* a model overfits, the *bigger* this gap, and the *easier* membership inference becomes. A perfectly generalizing model (one that behaves identically on training and test data) would, in theory, leak zero membership information. In practice almost all real models overfit to some degree, which is exactly why this attack class exists and works.

---

## 3. The Confidence-Gap Signal

The simplest and most intuitive membership inference signal is the **confidence score** a classifier outputs for its predicted class.

Most classifiers do not just output a label ("cat" or "dog") -- they output a **probability distribution** over all possible classes, e.g., `[cat: 0.97, dog: 0.02, bird: 0.01]`. The maximum value in that distribution is often called the model's **confidence**.

### Plain-English Statistics Refresher

- **Probability**: a number between 0 and 1 (or 0% and 100%) representing how likely something is. 0 means "never," 1 means "certain."
- **Probability distribution**: a full list of probabilities for every possible outcome, which together add up to 1 (100%). For a 3-class model, something like `[0.97, 0.02, 0.01]`.
- **Confidence score**: the probability the model assigns to its top-choice answer. High confidence = the model is "sure." Low confidence = the model is "unsure" and the probability mass is spread across multiple classes.

### Why This Leaks Membership

Because training pushes the model to be *very sure and very correct* about the exact examples it trained on, records from the training set tend to get **abnormally high confidence scores** compared to records the model has never seen. An attacker who can see confidence scores can often just threshold them:

```
  IF confidence(f(x)) > threshold  -->  predict "IN" (member)
  ELSE                              -->  predict "OUT" (non-member)
```

This is called a **threshold attack**, and it is the simplest possible membership inference attack. It is weak (it ignores a lot of nuance) but it illustrates the core idea perfectly, and it is often the first thing an attacker tries before building anything more sophisticated (like shadow models, covered next).

---

## 4. The Shadow Model Methodology

The threshold attack above has a problem: what threshold do you pick? "Confidence > 0.9" might work great for one model and terribly for another, depending on how that specific model was trained, what data it saw, and how it tends to behave. The attacker needs a way to **learn** the right signal rather than guess it. This is exactly what the **shadow model methodology** (introduced by Shokri et al., 2017, in the original membership inference paper) does.

### The Core Idea, in Plain English

If I don't know exactly how the target model behaves on members vs. non-members, I can **build my own copies** ("shadow models") that behave similarly, where *I control* which data points are members and which are not. By training many shadow models and watching exactly how their outputs differ for known members vs. known non-members, I can learn the *pattern* of what "membership" looks like in a model's output -- and then apply that learned pattern to the real target model.

### Step-by-Step Pipeline

```
                    THE SHADOW MODEL ATTACK PIPELINE
    ============================================================

    STEP 1: Gather "Shadow Data"
    -----------------------------
    The attacker collects a dataset that resembles the target
    model's training distribution (does NOT need to be the exact
    same data -- just data from a similar domain/distribution).

        +--------------------------------------------------+
        |           SHADOW DATASET (attacker-owned)         |
        |    (e.g., similar medical records, similar images)|
        +--------------------------------------------------+


    STEP 2: Train Many Shadow Models
    ----------------------------------
    Split the shadow dataset repeatedly into different
    IN / OUT partitions. Train one shadow model per split.
    Because the attacker did the splitting, they KNOW exactly
    which records were "IN" (used to train that shadow model)
    and which were "OUT" (held back).

        Shadow Dataset
             |
       +-----+------+------+------+
       |            |             |
       v            v             v
    Split 1      Split 2       Split 3    ... (many splits)
    IN: {a,b,c}  IN: {d,e,f}   IN: {g,h,i}
    OUT:{d,...}  OUT: {a,...}  OUT: {b,...}
       |            |             |
       v            v             v
    Shadow       Shadow        Shadow
    Model 1      Model 2       Model 3
    (trained     (trained      (trained
    on Split 1   on Split 2    on Split 3
    "IN" data)   "IN" data)    "IN" data)


    STEP 3: Query Each Shadow Model on ITS OWN Known IN/OUT Data
    ---------------------------------------------------------------
    For every shadow model, run its known "IN" records and known
    "OUT" records through it, and record the output vectors
    (confidence scores / full probability distributions).

        Shadow Model 1 + record "a" (known IN)  --> [0.97, 0.02, 0.01] --> label: IN
        Shadow Model 1 + record "x" (known OUT) --> [0.55, 0.30, 0.15] --> label: OUT
        Shadow Model 2 + record "d" (known IN)  --> [0.95, 0.03, 0.02] --> label: IN
        Shadow Model 2 + record "y" (known OUT) --> [0.48, 0.31, 0.21] --> label: OUT
        ... (thousands of these rows) ...

    This produces a brand-new labeled TRAINING SET for the attack
    itself, where:
        FEATURES = the output vector (confidence scores) from a
                   shadow model on a record
        LABEL    = IN or OUT (which the attacker knows for certain,
                   because they controlled the splits)


    STEP 4: Train the "Attack Model"
    -----------------------------------
    Train a separate classifier -- the ATTACK MODEL -- on this
    new IN/OUT-labeled dataset. This attack model learns the
    general SIGNATURE of "what a member's output vector looks
    like" vs. "what a non-member's output vector looks like."

        Attack Model Training Data:
        +---------------------------+----------+
        | Output vector (features)  |  Label   |
        +---------------------------+----------+
        | [0.97, 0.02, 0.01]        |   IN     |
        | [0.55, 0.30, 0.15]        |   OUT    |
        | [0.95, 0.03, 0.02]        |   IN     |
        | [0.48, 0.31, 0.21]        |   OUT    |
        +---------------------------+----------+
                    |
                    v
            +-------------------+
            |   ATTACK MODEL    |
            | (e.g., a simple   |
            |  binary classifier)|
            +-------------------+


    STEP 5: Attack the Real Target Model
    ----------------------------------------
    Now query the ACTUAL target model with the candidate record
    "Alice's data" the attacker actually cares about. Feed the
    target model's output vector into the trained attack model.

        Candidate record "Alice" --> TARGET MODEL --> [0.94, 0.04, 0.02]
                                                              |
                                                              v
                                                     ATTACK MODEL
                                                              |
                                                              v
                                                     Prediction: IN
                                          ("Alice's record was probably
                                           in the target's training set")
```

### Why Shadow Models Work Even Without the Real Training Data

The attacker never needs to see the target model's actual training data. They only need:
1. Data that comes from a **similar distribution** (e.g., if the target is a hospital's diagnosis model, any reasonably representative medical dataset works as shadow data).
2. **Query access** to the target model (black-box access: send an input, get a confidence score back).
3. The ability to train their own models locally (shadow models + attack model), which is cheap and fully under their control.

This is what makes the attack so dangerous in practice -- it does not require insider access to the victim's infrastructure, only the ability to query a deployed model's API and knowledge of the general kind of data it was trained on.

### Shadow-in-Shadow: Multiple Shadow Models Improve Robustness

A single shadow model might learn quirks specific to *that* model rather than the general phenomenon of overfitting. Using **many** shadow models (often dozens), each trained on a different random IN/OUT split, produces a much more robust attack model -- one that has learned the *general* statistical signature of memorization rather than a single model's idiosyncrasies. This mirrors the same "many weak signals combine into one strong signal" idea you will see again in Section 4 of this module (PATE), just used offensively here instead of defensively.

---

## 5. Worked Numeric Example

Let's make the confidence-gap signal concrete with small, easy-to-follow numbers.

### Setup

A hospital trains a binary classifier to predict "has condition X" (`positive` / `negative`) from patient records. The model was trained on 1,000 patient records. It **overfits somewhat**, as most real-world models do.

An attacker wants to know: "Was patient Bob's record used to train this model?" The attacker has a copy of Bob's record (perhaps leaked, or reconstructed from public information) and can query the deployed model's API, which returns a confidence score.

### Step 1: Attacker Trains Shadow Models

The attacker collects 5,000 similar (but not identical) patient records from public health datasets, splits them into 10 different IN/OUT partitions of 1,000 records each, and trains 10 shadow models with the same architecture as they assume the target uses (e.g., a similar-sized neural network).

### Step 2: Attacker Observes the Confidence Gap on Shadow Models

Averaging across all 10 shadow models, the attacker observes:

| Group | Average Confidence on Predicted Class | Average Loss |
|-------|---------------------------------------|--------------|
| Known members (IN) | 0.94 | 0.06 |
| Known non-members (OUT) | 0.62 | 0.41 |

The **confidence gap** is `0.94 - 0.62 = 0.32`. This 0.32 gap is the signal the attack model learns to detect.

### Step 3: Attack Model Learns a Decision Rule

From patterns like the table above (across thousands of individual records, not just the two averages), the attack model learns something like:

```
IF confidence >= 0.85  -->  predict IN   (member)
IF confidence <  0.85  -->  predict OUT  (non-member)
```

(In reality the attack model is more nuanced than a single threshold -- it may consider the full output vector, the entropy of the distribution, and even the ranking of classes -- but a single threshold is a good simplification to build intuition.)

### Step 4: Attacker Queries the Real Target Model with Bob's Record

```
Query target model with Bob's record -->  confidence = 0.91
```

`0.91 >= 0.85` -> the attack model predicts **IN**: Bob's record was likely part of the hospital's training set.

### Step 5: Interpreting Attack Accuracy

Suppose the attacker validates their attack model on a held-out shadow-model test set (records the shadow models never saw during attack-model training) and gets this confusion matrix:

```
                          PREDICTED
                    Member    Non-Member
        Member    [  850   |   150   ]   (out of 1,000 known members)
 ACTUAL Non-Member[  200   |   800   ]   (out of 1,000 known non-members)

 Attack Accuracy = (850 + 800) / 2000 = 82.5%
```

An accuracy of 82.5% is far above the 50% you would expect from random guessing on a balanced IN/OUT problem -- meaning the attack is working substantially better than chance, and the target model is leaking real membership information. In academic papers, even attack accuracies in the 55-65% range (well above the 50% baseline) are considered meaningful privacy leaks, because they demonstrate the model is *not* generalizing perfectly.

---

## 6. Signal Types Used in Membership Inference

Confidence score thresholding is the simplest signal, but real-world attacks combine several signal types for higher accuracy.

| Signal Type | What It Measures | Why It Leaks Membership | Attack Difficulty |
|-------------|-------------------|--------------------------|--------------------|
| **Prediction confidence** | The probability assigned to the top predicted class | Training pushes confidence on training points toward 1.0; unseen points get lower, more "honest" confidence | Low -- simplest to compute |
| **Loss value** | How wrong the model's prediction was, per the loss function | Loss on training data is driven toward zero during optimization; loss on unseen data stays higher | Low-Medium -- requires access to true labels |
| **Full prediction vector / entropy** | The complete probability distribution across all classes, or how "spread out" it is | Members tend to produce low-entropy (peaked, confident) distributions; non-members produce higher-entropy (spread out) ones | Medium -- more informative than a single confidence number |
| **Gradient-based signals (white-box)** | Magnitude/direction of gradients if the attacker has model access (weights) | Training data points tend to sit near local minima of the loss surface, producing small gradients; unseen points produce larger gradients | High -- requires white-box (full model) access |
| **Label-only signals** | Whether the model's *hard label* prediction is correct, without any confidence score at all | Models are more likely to correctly classify training points than test points; correctness alone is a weaker but still usable signal when confidence scores are hidden | Medium-High -- used when the API only returns labels, not probabilities |

```
    ACCESS LEVEL VS. SIGNAL RICHNESS

    BLACK-BOX               GRAY-BOX                  WHITE-BOX
    (label only)         (label + confidence)       (full model access)
        |                        |                          |
        v                        v                          v
    Weakest signal        Medium signal              Strongest signal
    (correctness only)    (confidence/entropy)       (gradients, loss
                                                        landscape, weights)
```

**Terminology check**:
- **Black-box access**: the attacker can only send inputs and see the final output label (e.g., "spam"). No numbers, no internals.
- **Gray-box access**: the attacker sees richer outputs, like full confidence scores or probability vectors, but still cannot see the model's internal weights.
- **White-box access**: the attacker has full access to the model itself -- weights, architecture, gradients, everything (e.g., a stolen or open-sourced model file).

---

## 7. Simplified Attack Walkthrough

Here is a condensed, end-to-end mental walkthrough tying everything together, in the order an attacker would actually execute it:

```
 1. IDENTIFY TARGET
    "There's a facial-recognition API used by Company X. I want to know
     if a specific photo of a real person was in its training set."

 2. COLLECT SHADOW DATA
    Gather a public dataset of similar face images (different individuals,
    similar domain: face photos).

 3. TRAIN SHADOW MODELS
    Train ~20 shadow models on different random splits of the shadow
    dataset, each with known IN/OUT labels per split.

 4. BUILD ATTACK TRAINING SET
    Query each shadow model with its own known IN and OUT images,
    recording confidence vectors + IN/OUT ground truth labels.

 5. TRAIN ATTACK MODEL
    Train a binary classifier (the "attack model") on the confidence
    vectors -> IN/OUT labels.

 6. QUERY TARGET MODEL
    Send the candidate photo to Company X's real API, record the
    confidence vector it returns.

 7. RUN ATTACK MODEL
    Feed that confidence vector into the trained attack model.
    Get back: IN or OUT.

 8. INTERPRET RESULT
    If IN: strong evidence the person's photo was used to train the
    model without (potentially) their consent -- a privacy violation.
```

---

## 8. Defenses

Briefly (this module explores each defense in depth in later sections):

| Defense | How It Helps | Covered In |
|---------|-------------|------------|
| **Regularization** (dropout, weight decay, early stopping) | Reduces overfitting directly, shrinking the confidence gap between members and non-members | Module 1 |
| **Differential Privacy (DP)** | Formally bounds how much any single training record can influence the model's output, capping the maximum possible membership-inference signal | Section 2 of this module |
| **DP-SGD** | A concrete training algorithm that achieves differential privacy by clipping and noising gradients | Section 3 of this module |
| **PATE** | Trains a "student" model that never directly touches sensitive raw records, limiting what can be inferred about any one record | Section 4 of this module |
| **Confidence score rounding/hiding** | Returning only top-1 labels instead of full probability vectors reduces (but does not eliminate) signal richness | Deployment-level mitigation |

---

## 9. Privacy Angle -- Why This Matters

> **Privacy Angle**: Membership inference is not just an academic curiosity -- it is a real, demonstrated privacy violation with legal and ethical weight. If a model was trained on sensitive data (medical records, financial history, private photos, genomic data) and an attacker can reliably determine "yes, this specific person's data was used," that alone can:
>
> - **Reveal sensitive facts by association.** If a model predicts "likelihood of disease X" and an attacker confirms a specific person's record was used to train it, they may be able to infer that person actually *has* condition X (especially if the training set was curated from confirmed-diagnosis patients).
> - **Violate consent and regulatory requirements.** Regulations like GDPR and HIPAA restrict how personal data can be used. Successful membership inference can prove a data subject's information was used without proper authorization, exposing the model owner to legal liability.
> - **Undermine anonymization claims.** Organizations often claim a trained model is a "safe," "anonymized," or "aggregated" summary of data that cannot leak individual records. Membership inference is direct proof that this claim can be false.
> - **Serve as a stepping stone to worse attacks.** Once membership is established, more powerful **model inversion attacks** (reconstructing actual feature values, not just yes/no membership) become easier to target and validate.
>
> This is precisely why formal, mathematical privacy guarantees -- not just "we hope overfitting is low" -- are needed. That is the motivation for **Differential Privacy**, covered next.

---

## 10. Key Takeaways

- **Membership inference attacks (MIA)** determine whether a specific record was part of a model's training set, by observing how the model behaves (confidence, loss, correctness) on that record.

- **Overfitting is the root cause.** Models trained to minimize loss on their training set tend to become abnormally confident and low-loss on that exact data, creating a measurable gap versus unseen data -- the **generalization gap**.

- **The confidence-gap signal** is the simplest exploit: threshold the model's confidence score to guess "member" vs. "non-member."

- **The shadow model methodology** is the gold-standard technique: train many shadow models on known IN/OUT splits of similar data, use their outputs to build a labeled dataset of "what membership looks like," train an attack model on that, then apply the attack model to the real target's outputs.

- **Signal richness scales with access level**: black-box (label only) attacks are weakest, gray-box (confidence scores) are stronger, white-box (gradients, weights) are strongest.

- **Attack accuracy above 50% (random guessing) is a meaningful privacy leak**, even if it's not close to 100% -- it proves the model is not perfectly generalizing and is retaining record-specific information.

- **Defenses trace directly back to reducing the generalization gap**: regularization, and more rigorously, mathematically-guaranteed approaches like Differential Privacy, DP-SGD, and PATE -- all covered in the rest of this module.

*Next up: Differential Privacy Fundamentals -- the mathematical framework that formally bounds how much a model's output can depend on any single individual's data, directly limiting how much membership inference (and other privacy attacks) can ever succeed.*
