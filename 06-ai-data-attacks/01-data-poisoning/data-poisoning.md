# Data Poisoning

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Data Attacks | Section: Data Poisoning

---

## Table of Contents

1. [What is Data Poisoning?](#1-what-is-data-poisoning)
2. [Where Poisoning Fits in the ML Pipeline](#2-where-poisoning-fits-in-the-ml-pipeline)
3. [Threat Model -- Who Can Poison a Dataset?](#3-threat-model----who-can-poison-a-dataset)
4. [Categories of Data Poisoning](#4-categories-of-data-poisoning)
5. [Availability Attacks vs. Integrity Attacks](#5-availability-attacks-vs-integrity-attacks)
6. [Real-World Incidents and Research Context](#6-real-world-incidents-and-research-context)
7. [Worked Example -- Poisoning a Spam Filter](#7-worked-example----poisoning-a-spam-filter)
8. [Detecting and Defending Against Poisoning](#8-detecting-and-defending-against-poisoning)
9. [Security Angle](#9-security-angle)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. What is Data Poisoning?

### The Analogy

Imagine you are training a new employee by having them read through a stack of 1,000 "example decisions" made by past employees, so they learn the company's approach. Now imagine a disgruntled coworker sneaks 50 fake example decisions into that stack -- decisions that look plausible but are subtly wrong. The new employee reads all 1,000 examples, trusts every one of them equally, and quietly absorbs the bad habits along with the good ones. Nobody told them which examples were fake, so they cannot tell the difference.

That is data poisoning. A machine learning model is only as trustworthy as the data it is trained on, and it has no built-in ability to know that some of its "examples" were planted by an attacker.

### The Formal Definition

**Data poisoning** is an attack in which an adversary deliberately manipulates the training data (or the labels attached to it) used to build a machine learning model, with the goal of corrupting the model's learned behavior. The corruption might be:

- **General** -- making the model worse across the board (a denial-of-service style attack on model quality).
- **Targeted** -- making the model wrong only in specific, attacker-chosen situations, while looking perfectly normal everywhere else.

Poisoning happens **before or during training**. This distinguishes it from attacks like adversarial examples or evasion, which happen at **inference time** (i.e., against an already-trained, already-deployed model). Poisoning attacks the *nature* of the model itself, not a single prediction.

```
   NORMAL TRAINING                          POISONED TRAINING

   +-----------------+                      +-----------------+
   | Clean Dataset   |                      | Clean Dataset   |
   | (10,000 samples)|                      | (9,800 samples) |
   +-----------------+                      +-----------------+
            |                                        +
            v                                +-----------------+
   +-----------------+                       | Poisoned samples|
   | Train Model     |                       | (200 samples,   |
   +-----------------+                       |  attacker made) |
            |                                +-----------------+
            v                                        |
   +-----------------+                                v
   | Model behaves    |                      +-----------------+
   | as designed      |                      | Train Model     |
   +-----------------+                      +-----------------+
                                                       |
                                                       v
                                             +-----------------+
                                             | Model behaves    |
                                             | as ATTACKER wants|
                                             | (subtly or fully |
                                             |  broken)         |
                                             +-----------------+
```

---

## 2. Where Poisoning Fits in the ML Pipeline

Recall the ML pipeline from Module 1: Problem Definition -> Data Collection -> Data Preprocessing -> Model Training -> Model Evaluation -> Deployment. Data poisoning is injected at the **Data Collection** and **Data Preprocessing** stages -- long before the model is trained.

```
+----------------+     +----------------+     +----------------+
|  1. PROBLEM    |---->|  2. DATA       |---->|  3. DATA       |
|  DEFINITION    |     |  COLLECTION    |     |  PREPROCESSING |
+----------------+     +----------------+     +----------------+
                              ^                        ^
                              |                        |
                        [ POISONING            [ POISONING can
                          INJECTED HERE ]         also be injected
                        - Scraped web data          here if attacker
                          attacker controls          controls a
                        - Crowdsourced labels         labeling
                          (e.g. Mechanical             pipeline/vendor ]
                          Turk workers)
                        - User-submitted data
                          (e.g. spam reports,
                          telemetry, retraining
                          feedback loops)
                        - Compromised upstream
                          dataset repository
                                                        |
                                                        v
+----------------+     +----------------+     +----------------+
|  6. DEPLOYMENT |<----|  5. MODEL      |<----|  4. MODEL      |
|  & MONITORING  |     |  EVALUATION    |     |  TRAINING      |
+----------------+     +----------------+     +----------------+
                                                        ^
                                              Model faithfully
                                              learns whatever was
                                              in the (now poisoned)
                                              training set.
```

Common real-world injection points:

| Injection Point | Example |
|---|---|
| Public data scraping | A model trained on scraped web pages, forum posts, or images that an attacker seeded with malicious content ahead of time |
| Crowdsourced labeling | A labeling vendor or crowd worker is bribed/compromised to mislabel a subset of samples |
| User feedback loops | A spam filter that retrains on user "not spam" clicks -- an attacker mass-clicks "not spam" on spam campaigns |
| Federated learning | Each participant contributes local gradient updates; a malicious participant contributes poisoned updates |
| Third-party datasets | A publicly shared dataset (e.g., on a model/dataset hub) is poisoned before others download and fine-tune on it |
| Supply chain | A compromised data pipeline component silently mutates records in transit |

---

## 3. Threat Model -- Who Can Poison a Dataset?

Before diving into attack types, it helps to define **how much access and knowledge** an attacker needs. This is the same "threat model" thinking used in traditional pentesting.

| Attacker Capability | Description | Example |
|---|---|---|
| **Data injection** | Attacker can add new samples to the dataset, but cannot modify existing ones | Submitting fake reviews, fake network traffic, fake spam reports |
| **Data modification** | Attacker can alter existing samples/labels (e.g., insider access, compromised pipeline) | A rogue annotator changes labels on samples they are assigned to review |
| **Full knowledge (white-box)** | Attacker knows the model architecture, training algorithm, and can often see the whole dataset | Insider threat, or attacking an open dataset used by many downstream teams |
| **Partial/no knowledge (black-box)** | Attacker only knows the general domain (e.g., "this is a spam filter") and injects data blindly | External attacker submitting data through a public-facing feedback form |

The stronger the attacker's access and knowledge, the more precise and effective the poisoning can be -- but even weak, black-box attackers can do real damage, especially with **label flipping** (see next file), which requires no knowledge of the model at all.

---

## 4. Categories of Data Poisoning

This module covers two closely related label-based poisoning techniques in detail (in their own files), plus a feature-based technique in the next section. Here is the map:

```
                         DATA POISONING
                               |
          +--------------------+--------------------+
          |                                          |
     LABEL-BASED                              FEATURE-BASED
     (change the "answer")                    (change the "question")
          |                                          |
   +------+------+                                   |
   |             |                                    v
LABEL          TARGETED                        CLEAN-LABEL
FLIPPING       LABEL                            ATTACKS
(random/       ATTACKS                     (features perturbed,
broad          (specific                    labels stay CORRECT --
mislabeling)   blind spot)                  covered in next file)
```

| Attack Type | What Changes | Attacker Goal | Covered In |
|---|---|---|---|
| Label Flipping | Labels, broadly/randomly | Degrade overall model accuracy | This module, file 2 |
| Targeted Label Attacks | Labels, specific classes/samples | Create a precise blind spot | This module, file 3 |
| Clean-Label Attacks | Features (pixels/values), labels untouched | Poison while evading label audits | Module 2 of this section |
| Backdoor/Trojan Attacks | Features + labels, tied to a trigger | Hidden attacker-controlled behavior | Module 3 of this section |

---

## 5. Availability Attacks vs. Integrity Attacks

Data poisoning attacks are usually classified by their **objective**:

### Availability Attacks (a.k.a. Indiscriminate Poisoning)

**Goal**: Make the model broadly less useful. The attacker does not care exactly *how* the model fails, just that it fails a lot.

- Analogy: sabotaging a factory's entire assembly line so the whole batch of product comes out defective.
- Example: flip 30% of labels randomly across an entire training set so the resulting model's accuracy craters from 95% to 60%.
- Easy to detect eventually (accuracy drops are visible in evaluation metrics), but can be very disruptive if it slips through, especially in automated retraining pipelines with no human review.

### Integrity Attacks (a.k.a. Targeted Poisoning)

**Goal**: Make the model fail in one **specific, attacker-chosen** way, while performing normally on everything else -- including the model's own evaluation metrics.

- Analogy: sabotaging a single security checkpoint's badge reader so that *one specific fake badge* is always accepted, while every other badge is checked normally. Auditors testing the checkpoint with normal badges see nothing wrong.
- Example: an attacker wants malware samples signed with a particular packer to always be classified "benign," while every other malware family is still caught normally.
- Much harder to detect because overall accuracy/precision/recall on standard test sets looks completely fine.

```
   AVAILABILITY (INDISCRIMINATE)            INTEGRITY (TARGETED)
   ================================         ================================
   Accuracy on ALL test data: 60%           Accuracy on ALL test data: 94.8%
   (visibly broken, easy to catch)          (looks totally healthy!)

                                             ...but for the ONE attacker-chosen
                                             input pattern (e.g. "packer=XYZ"):
                                             Accuracy: 3%  <-- the blind spot
```

This distinction matters immensely for offensive security work: **integrity/targeted attacks are the dangerous, stealthy ones**, because standard QA processes will not catch them. Section 3 of this file (Targeted Label Attacks) focuses specifically on this category.

---

## 6. Real-World Incidents and Research Context

Data poisoning is not a purely theoretical concern -- it is a well-documented risk category that security researchers, ML researchers, and even production incidents have illustrated repeatedly. A few well-known, publicly discussed examples and research threads worth knowing for the exam:

| Case / Research Area | What Happened / What Was Shown | Lesson |
|---|---|---|
| **Chatbots learning from live user interaction** | Several publicly documented cases of conversational AI systems that retrained or adapted based on live user input ended up producing offensive or manipulated outputs after coordinated user campaigns fed them abusive/misleading input | Any system that treats live user interaction as a training signal is a poisoning target, whether the "attack" is a deliberate coordinated campaign or just adversarial trolling |
| **Spam filter research (academic)** | Early academic research on spam filtering (long before "data poisoning" was a common term) demonstrated that injecting crafted spam-like "ham" (legitimate) messages into a Bayesian spam filter's training data could measurably shift the filter's decision boundary | Established, decades-old research shows this is not a new or exotic risk -- it predates modern deep learning entirely |
| **Recommendation system manipulation** | Research and real-world abuse cases around recommendation engines show that coordinated fake engagement (clicks, "likes," reviews) can shift what a model learns to recommend, functioning as a poisoning attack via the feedback loop | Any model that treats aggregate user behavior as ground truth is exposed to poisoning via manipulated behavior at scale |
| **Federated learning poisoning research** | Academic research on federated learning (where many participants train collaboratively without sharing raw data) has repeatedly shown that a small fraction of malicious participants contributing poisoned local updates can implant backdoors or degrade the shared global model | Federated/collaborative training setups are an especially attractive target because the raw data (and therefore any poisoning) is never centrally visible for inspection |
| **Public dataset and model-hub supply chain concerns** | Security researchers have repeatedly raised concerns (and demonstrated proof-of-concept attacks) about the ease of uploading tampered datasets or backdoored pre-trained checkpoints to public sharing platforms | Ties directly into the Trojan/Backdoor and Model Artifact Exploitation attacks covered later in this module |

### Why This History Matters

The consistent theme across all of these is that **any pipeline stage where a model "learns" from data it does not fully control or verify is a poisoning surface** -- whether that's live user feedback, crowdsourced labels, federated participant updates, or a publicly shared dataset. This is true regardless of how modern or sophisticated the underlying model architecture is; poisoning is a property of the *trust model* around the data, not of the algorithm.

---

## 7. Worked Example -- Poisoning a Spam Filter

Let's make this concrete with numbers, continuing the spam-filter example from Module 1.

### Setup

- Training set: 10,000 emails (3,000 spam, 7,000 ham), same as the Module 1 example.
- Clean model (from Module 1) achieved on the test set: Accuracy 97.0%, Precision 96.6%, Recall 93.3%.

### Availability Poisoning Scenario

An attacker who has write access to the labeling queue flips the label on **15% of the training set indiscriminately** (1,500 of the 10,000 training emails get their label flipped: spam becomes ham, ham becomes spam).

```
Before poisoning:  3,000 spam (correct) | 7,000 ham (correct)
Attacker flips 15% at random (1,500 emails)

After poisoning:   ~2,700 emails that were spam now labeled "ham"  (spam mislabeled)
                    ~2,700 emails that were ham now labeled "spam" (ham mislabeled)
                    (exact split depends on random draw, roughly proportional
                     to original 30/70 class balance)
```

Illustrative effect on the resulting model's test-set performance (numbers are for teaching purposes, not derived from a real experiment):

```
                      PREDICTED (poisoned model)
                   Spam       Ham
 ACTUAL  Spam   [  280   |  170  ]   (many spam emails now missed)
         Ham    [  190   |  860  ]   (more false alarms too)

 Accuracy  = (280 + 860) / 1500  = 76.0%   (down from 97.0%)
 Precision = 280 / (280 + 190)   = 59.6%   (down from 96.6%)
 Recall    = 280 / (280 + 170)   = 62.2%   (down from 93.3%)
```

A 15% indiscriminate label flip took the model from a strong 97% accuracy classifier to a barely-usable 76% one. This is a **loud** attack -- any competent QA process reviewing model performance metrics before deployment would catch this immediately.

### Targeted Poisoning Scenario (Preview)

Compare that to a targeted attack (fully explored in the next file): the attacker flips the label on just **60 emails**, all containing the phrase "invoice attached," from "spam" to "ham." That's only 0.6% of the training set.

```
 Overall test accuracy: 96.7%  (barely moved -- looks healthy!)

 But specifically for emails containing "invoice attached":
   Before poisoning: 98% correctly flagged as spam
   After poisoning:  4% correctly flagged as spam
   (96% of "invoice attached" phishing/spam now sails through)
```

This is the power of targeted poisoning: **60 mislabeled emails out of 10,000 (0.6%)** created an almost invisible blind spot that an attacker can now exploit reliably, while a QA reviewer checking aggregate accuracy sees nothing alarming.

---

## 7. Detecting and Defending Against Poisoning

| Defense | How It Works | Limitation |
|---|---|---|
| **Data provenance / lineage tracking** | Log where every training sample came from and who/what modified it | Doesn't stop poisoning, only helps trace it after the fact |
| **Outlier / anomaly detection on features** | Flag samples that are statistical outliers vs. their labeled class (e.g., an "outlier" spam email that looks nothing like other spam) | Sophisticated attacks (clean-label, see next module) are designed to blend in |
| **Label auditing / spot-checking** | Humans manually review a random sample of labels | Random sampling can easily miss a small targeted subset (0.6% of 10,000 = only ~60 samples to hide among 10,000) |
| **Robust/differentially private training** | Use training algorithms less sensitive to small numbers of extreme/anomalous points (e.g., trimmed-mean aggregation, robust loss functions) | Adds complexity and can reduce accuracy on legitimately hard cases |
| **Influence functions** | Mathematically estimate which training samples have outsized influence on specific predictions, then inspect the top-influence samples | Computationally expensive on large datasets/models |
| **Held-out trusted validation set** | Keep a small, independently-verified "golden" dataset to sanity-check model behavior on known-good/known-bad cases | Only as good as the golden set's coverage of the attack surface |
| **Access control on data pipelines** | Limit who/what can write to training data stores; require review/approval for pipeline changes | Standard security hygiene, but doesn't fully prevent insider or supply-chain attacks |

---

## 8. Security Angle

For an offensive AI practitioner, data poisoning is one of the highest-leverage attacks in the entire ML attack surface, because:

- **It requires no access to the model or the deployment infrastructure at all** -- only to the data pipeline (or a public dataset/feedback channel the pipeline trusts).
- **It can be persistent**: once poisoned data is baked into a trained model, the vulnerability survives even if the original poisoned samples are later deleted from the dataset -- the model already "learned" from them.
- **It scales to supply-chain attacks**: many organizations fine-tune publicly available pre-trained models or use public datasets. Poisoning a widely-used public dataset or a popular pre-trained checkpoint (see the Trojan/Backdoor and Model Artifact Exploitation files later in this module) can compromise many downstream victims at once from a single upstream act.
- **Targeted variants are extremely stealthy** and can pass standard model-evaluation QA, since aggregate metrics look fine.

As you assess a target ML system, always ask: *Where does this system's training data come from, who can write to it, and how often is it retrained?* Systems that continuously retrain on live user feedback (recommendation engines, spam filters, fraud detectors) are the juiciest poisoning targets because the attacker doesn't need to breach anything -- they can just "use the product" in a way that poisons it.

---

## 9. Key Takeaways

- **Data poisoning corrupts the training data itself**, so the resulting model is broken (or backdoored) by design, not tricked at inference time like an adversarial example.
- **It is injected at the Data Collection / Preprocessing stages** of the ML pipeline -- upstream of everything else, which is exactly why it is so dangerous and hard to catch downstream.
- **Threat models range from blind data injection to full white-box insider access** -- but even the weakest attacker (blind label flipping) can meaningfully damage a model.
- **Availability attacks** (indiscriminate) are loud and easy to detect via accuracy drops; **integrity attacks** (targeted) are stealthy and can hide behind healthy-looking aggregate metrics.
- **A tiny fraction of poisoned samples** (well under 1% in the targeted example above) can create a reliable, attacker-exploitable blind spot without moving overall accuracy in any noticeable way.
- **Defenses focus on provenance, outlier detection, auditing, and robust training** -- but no single defense is complete, especially against attacks designed to look statistically normal.

*Next up: Label Flipping -- a deep dive into the simplest and most accessible form of data poisoning, where an attacker randomly or systematically mislabels training samples.*
