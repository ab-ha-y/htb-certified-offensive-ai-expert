# Label Flipping

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Data Attacks | Section: Data Poisoning

---

## Table of Contents

1. [What is Label Flipping?](#1-what-is-label-flipping)
2. [Random Label Flipping](#2-random-label-flipping)
3. [Systematic Label Flipping](#3-systematic-label-flipping)
4. [How Flipping Rate Relates to Model Damage](#4-how-flipping-rate-relates-to-model-damage)
5. [Worked Example -- Flipping X% of a Dataset of Size N](#5-worked-example----flipping-x-of-a-dataset-of-size-n)
6. [Where Label Flipping Happens in Practice](#6-where-label-flipping-happens-in-practice)
7. [Detecting Label Flipping](#7-detecting-label-flipping)
8. [Security Angle](#8-security-angle)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What is Label Flipping?

### The Analogy

Picture a stack of 1,000 flashcards used to train a new quality-control inspector at a factory: one side shows a photo of a product, the other side says "PASS" or "FAIL." Now imagine someone secretly swaps the answer on the back of 100 of those flashcards -- a photo of a clearly defective product now says "PASS" on the back, and a perfectly good product now says "FAIL." The trainee studies all 1,000 cards trusting every answer equally. They will now confidently make the wrong call on similar-looking products in the real world, and they have no idea why -- they studied hard, they just studied corrupted material.

**Label flipping** is exactly this: taking correctly labeled training samples and changing (flipping) their labels to something incorrect, without touching the underlying data (the photo, the email text, the network packet) itself.

### The Formal Definition

**Label flipping** is a data poisoning technique where an attacker changes the ground-truth label of one or more training samples while leaving the sample's features unchanged. It is the simplest and most accessible form of data poisoning because:

- It requires **no understanding of the model architecture** or training algorithm (it's a black-box attack against the *data*, not the model).
- It requires **no ability to fabricate realistic-looking fake data** -- the attacker reuses real, legitimate samples and just corrupts the answer key.
- It can be carried out by anyone with write access (or influence) over the labeling process -- a rogue annotator, a compromised labeling API, or a manipulated crowdsourcing/feedback pipeline.

```
   BEFORE (clean labels)              AFTER (label flipped)

   +------------------+               +------------------+
   | Email: "Buy      |               | Email: "Buy      |
   |  cheap pills"    |               |  cheap pills"    |
   |                  |   ATTACKER    |                  |
   | Label: SPAM      |-------------->| Label: HAM       |
   +------------------+   flips the   +------------------+
                          label only     (features/text
                                          untouched --
                                          only the answer
                                          key changed)
```

Label flipping comes in two flavors, covered in the next two sections: **random** (broad, indiscriminate) and **systematic** (structured, often targeting a specific class or boundary).

---

## 2. Random Label Flipping

### What It Is

The attacker selects a random subset of the training samples and flips their labels, without regard for which class they belong to or how "important" they are. In a binary classification problem (two labels, e.g. spam/ham or malicious/benign), flipping means simply inverting the label. In a multi-class problem, flipping usually means reassigning to a random *other* class.

```
   RANDOM LABEL FLIPPING (binary example)

   Pick X% of samples at random from the WHOLE dataset
   (regardless of their current label)
            |
            v
   For each selected sample:
       if label == "spam": label = "ham"
       if label == "ham":  label = "spam"
```

### Why Attackers Use It

- **Simplicity**: no domain knowledge, no need to identify a specific weakness to target.
- **This is an Availability Attack** (see the parent Data Poisoning file): the goal is to make the model generally unreliable, a kind of "denial of service" against model quality, rather than to create one precise blind spot.
- Effective against automated pipelines that retrain regularly on live data without human review of labels.

### Downsides (From the Attacker's Perspective)

- It's **loud**. As flip rate increases, model accuracy on a held-out test set drops noticeably, and any monitoring on model performance metrics will surface the degradation quickly.
- It affects the model broadly and indiscriminately -- the attacker cannot control precisely *how* the model breaks, just that it breaks.

---

## 3. Systematic Label Flipping

### What It Is

Instead of flipping labels at random across the whole dataset, the attacker flips labels according to a **rule or pattern** -- usually concentrated on a specific class, a specific feature range, or a boundary region between classes.

Common systematic strategies:

| Strategy | Description | Example |
|---|---|---|
| **One-directional flipping** | Only flip labels of one class into another (never the reverse) | Flip malware samples to "benign," but never flip benign samples to "malware" |
| **Boundary flipping** | Flip only samples that sit near the model's decision boundary (where a small nudge in training data has an outsized effect on where the boundary ends up) | Flip labels on emails that already look borderline spammy, to shift the decision boundary further in the attacker's favor |
| **Confidence-based flipping** | Flip labels on samples the *current* model is *most confident* about (paradoxically, these carry the most "teaching weight" and do the most damage per flip) | Use an existing model to score training samples, then flip the highest-confidence ones |
| **Feature-correlated flipping** | Flip labels only for samples sharing a particular feature (e.g., a sender domain, a file hash prefix, a packer signature) | This shades into Targeted Label Attacks -- see the next file in this module |

### Why It Is More Dangerous Than Random Flipping

Systematic flipping gets **more damage per flipped label** than random flipping, because the attacker is spending their "flip budget" where it matters most (near decision boundaries or on high-influence samples) instead of wasting flips on samples that barely affect the model anyway.

```
   RANDOM FLIP                          SYSTEMATIC (BOUNDARY) FLIP

   .  .  x  .   .                       .  .  .   .  .
      .    x .                             .    . 
   .  x .   .  .        vs.             .  x|x .  .   <- flips concentrated
      .   x    .                           x|x .      <- right at the boundary
   .    .    x  .                        .  x|.   .   <- moves boundary a lot
                                                          per flip spent

   Flips scattered randomly.            Flips concentrated where they
   Some flips "waste" effort on        do maximum damage to the
   samples that barely matter to        decision boundary.
   the boundary.
```

---

## 4. How Flipping Rate Relates to Model Damage

The relationship between the **percentage of labels flipped** and **resulting model degradation** is not perfectly linear -- it depends on:

- **Model type**: some algorithms (e.g., k-nearest neighbors, decision trees) are more sensitive to individual mislabeled points than others (e.g., ensembles like random forests average out some noise).
- **Class balance**: flipping a fixed number of labels in a heavily imbalanced dataset (e.g., 3% malware, 97% benign) can be far more damaging to the minority class than the same number of flips in a balanced dataset.
- **Flip strategy**: as shown above, systematic flipping near the decision boundary does more damage per flip than random flipping.
- **Model capacity**: very high-capacity models (e.g., deep neural networks) can sometimes "memorize around" a modest amount of label noise better than simpler models, up to a point -- but capacity alone does not immunize a model against poisoning.

A useful mental model (illustrative, not derived from a specific published study):

```
   Illustrative relationship between flip rate and model accuracy
   (binary classifier, random flipping, balanced classes)

   Flip Rate    Approx. Accuracy Impact
   ---------    -----------------------
    1%          Small dip, often within normal noise -- hard to notice
    5%          Noticeable dip, borderline detectable via metrics
   15%          Significant, clearly detectable degradation
   30%          Severe -- model approaches random-guess territory
   50%          Model is trained on essentially the OPPOSITE of truth
                for that fraction, badly corrupted overall
```

The takeaway: **you don't need to flip a huge fraction of the dataset to cause real damage**, especially with systematic strategies -- but very high, indiscriminate flip rates make the attack easy to spot through standard evaluation.

---

## 5. Worked Example -- Flipping X% of a Dataset of Size N

Let's build a fully worked, numeric example using a malware classifier.

### Setup

- Dataset size **N = 5,000** samples.
- Class balance: 1,000 malware (20%), 4,000 benign (80%) -- a realistic imbalance for security data.
- Clean model baseline on a 1,000-sample test set: Accuracy 96%, Precision 90%, Recall 85% (illustrative).

### Scenario A: Random Flip, X = 10%

The attacker flips the label on **10% of N = 500 samples**, chosen uniformly at random across the whole training set (which is a subset of N, but for simplicity assume flips are drawn proportionally to class size, i.e. roughly 100 malware->benign flips and 400 benign->malware flips, matching the 20/80 split).

```
Training set composition after flip:
  Malware samples mislabeled as benign: ~100  (out of ~800 malware in training)
  Benign samples mislabeled as malware:  ~400  (out of ~3,200 benign in training)

Illustrative test-set result after retraining on poisoned data:
                     PREDICTED
                  Malware    Benign
 ACTUAL Malware  [  110   |   90  ]   (recall drops hard: many malware missed)
        Benign   [  140   |  660  ]   (many false alarms too)

 Accuracy   = (110 + 660) / 1000       = 77.0%   (down from 96%)
 Precision  = 110 / (110 + 140)        = 44.0%   (down from 90%)
 Recall     = 110 / (110 + 90)         = 55.0%   (down from 85%)
```

A 10% indiscriminate flip roughly **halved precision and recall**. This is a large, easily detectable degradation -- exactly the kind of loud signal a monitoring dashboard would catch before the model reaches production.

### Scenario B: Systematic Flip, X = 2% (Boundary-Targeted)

The attacker instead flips only **2% of N = 100 samples**, but chooses them specifically from malware samples that already look "borderline" (e.g., malware using heavy obfuscation that makes them resemble benign software feature-wise), flipping them all from "malware" to "benign."

```
Training set composition after flip:
  100 borderline malware samples flipped to "benign"
  (out of 1,000 total malware -- 10% of the malware class,
   but only 2% of the overall dataset)

Illustrative test-set result after retraining on poisoned data:
                     PREDICTED
                  Malware    Benign
 ACTUAL Malware  [  130   |   70  ]   (still catches "obvious" malware fine...)
        Benign   [   45   |  755  ]   (...but obfuscated malware slips through)

 Accuracy   = (130 + 755) / 1000       = 88.5%   (down from 96%, but looks "okay")
 Precision  = 130 / (130 + 45)         = 74.3%
 Recall     = 130 / (130 + 70)         = 65.0%

 BUT specifically for obfuscated/borderline malware samples in the test set:
   Before poisoning:  82% correctly caught
   After poisoning:   9% correctly caught  <-- the real damage, hidden inside
                                                the aggregate numbers above
```

Only **2% of the dataset** flipped (versus 10% in Scenario A) produced a much smaller drop in *aggregate* accuracy (88.5% vs. 77.0%) -- meaning it's more likely to slip past a reviewer glancing at the headline number -- while still devastating the model's ability to catch the specific attacker-relevant subclass (obfuscated malware). This demonstrates why **where** you flip matters as much as **how many** you flip.

---

## 6. Where Label Flipping Happens in Practice

| Scenario | How the Attacker Gets Flip Access |
|---|---|
| Crowdsourced labeling platforms | Bribe/compromise a subset of crowd workers, or register many low-reputation worker accounts |
| Community-reported labels | Abuse "report as spam / not spam" or "report as malicious / safe" community feedback that feeds a retraining pipeline |
| Compromised labeling tool or vendor | Insert a man-in-the-middle between the labeling tool and the training data store |
| Insider threat | A rogue or coerced employee with direct access to the labeled dataset |
| Public dataset tampering | Submit or modify entries in an open, collaboratively maintained dataset before it is packaged and distributed |
| Active learning feedback loops | Systems that ask humans (or automated oracles) to label the model's "most uncertain" samples are a prime target -- the samples chosen for labeling are, by definition, the most influential near the decision boundary |

---

## 7. Detecting Label Flipping

| Technique | How It Helps |
|---|---|
| **Cross-validation label consistency checks** | Train several models on different folds/subsets; if a sample's label frequently disagrees with what most models confidently predict for it, flag it for review |
| **Loss-based outlier detection** | During training, samples with unusually high loss relative to similar samples (i.e. the model "fights" to fit them) are often mislabeled |
| **Clustering / nearest-neighbor label agreement** | If a sample's label disagrees with the majority label of its k-nearest-neighbors in feature space, it is suspicious |
| **Re-labeling audits with independent annotators** | Periodically have a trusted second team re-label a random sample and compare disagreement rates against historical baselines |
| **Rate-limiting / reputation on feedback sources** | Limit how much influence any single labeling source (crowd worker, user account, API client) can have on the aggregate dataset |

---

## 8. Security Angle

Label flipping is the entry-level move in the data poisoning playbook, and it is worth mastering first because:

- **It is the lowest-effort, highest-accessibility poisoning technique.** No need to fabricate data, no need for model access -- just the ability to influence the answer key on real, unmodified samples.
- **It is a great "proof of concept" for testing whether a target pipeline validates labels at all.** If you can get even a handful of flipped labels accepted into a retraining pipeline (e.g., via a public feedback form), you have demonstrated a real vulnerability, even before attempting anything more sophisticated like clean-label or backdoor attacks.
- **Systematic flipping near decision boundaries is the natural stepping stone to Targeted Label Attacks** (next file), where the attacker goes further and deliberately engineers a specific, exploitable blind spot rather than generally degrading the model.
- When assessing a target, always check: *does this system accept community/crowdsourced feedback that eventually influences training? Is there any anomaly detection on label distributions before retraining?* Many production ML pipelines have none.

---

## 9. Key Takeaways

- **Label flipping changes only the answer, never the underlying data** -- making it the simplest data poisoning technique and requiring zero model knowledge.
- **Random flipping** is an availability attack: loud, broad, and detectable through aggregate accuracy drops.
- **Systematic flipping** (near decision boundaries, on high-confidence samples, or on a specific feature-correlated subset) does more damage per flipped label and can hide behind a healthy-looking aggregate accuracy score.
- **Flip rate and damage are not linearly related** -- class imbalance, model type, and flip strategy all change how much a given percentage of flips actually hurts the model.
- **A worked numeric example** showed that 10% random flips roughly halved precision/recall (a loud, detectable attack), while just 2% systematic flips on borderline samples caused only a modest aggregate accuracy drop but devastated performance on the specific attacker-relevant subclass.
- **Defenses rely on statistical consistency checks** (cross-validation, nearest-neighbor label agreement, loss-based outlier detection) since there is no way to "see" a flipped label just by looking at it.

*Next up: Targeted Label Attacks -- where the attacker goes beyond broad damage and deliberately engineers a precise, attacker-chosen blind spot in the model.*
