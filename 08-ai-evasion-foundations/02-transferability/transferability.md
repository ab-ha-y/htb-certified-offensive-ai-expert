# Transferability

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - Foundations | Section: Transferability

---

## Table of Contents

1. [What Is Transferability?](#1-what-is-transferability)
2. [Why This Is Surprising](#2-why-this-is-surprising)
3. [Why It Happens: Shared Decision Boundary Geometry](#3-why-it-happens-shared-decision-boundary-geometry)
4. [The Math Intuition: Similar Functions, Similar Gradients](#4-the-math-intuition-similar-functions-similar-gradients)
5. [Worked Numeric Example: Two Classifiers, One Adversarial Example](#5-worked-numeric-example-two-classifiers-one-adversarial-example)
6. [The Surrogate Attack Pipeline](#6-the-surrogate-attack-pipeline)
7. [Factors That Affect Transferability](#7-factors-that-affect-transferability)
8. [Transferability Across Model Families](#8-transferability-across-model-families)
9. [Defensive Implications](#9-defensive-implications)
10. [Security Angle](#10-security-angle)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What Is Transferability?

**Transferability** is the (initially very surprising) property that an adversarial example crafted to fool one machine learning model often *also* fools a completely different model -- one with different architecture, different training data, even trained by a different team -- as long as both models are trying to solve a similar task.

### The Analogy

Imagine two completely different security guards working at two different buildings, trained by two different companies, using two different training manuals. You discover that wearing a specific reflective vest and confidently carrying a clipboard gets you waved through the front desk at Building A without a second glance. Astonishingly, when you try the exact same vest-and-clipboard trick at Building B -- a building with different guards, a different training program, and different security policies -- it *also* works.

Why would that be? Because both guards, despite different training, ended up learning a very similar (flawed) heuristic: "people with clipboards and reflective vests are probably maintenance staff, let them through." Their decision-making processes converged on a similar shortcut, even though neither guard was trained by the same person or saw the same faces during training.

This is exactly what happens between two machine learning models trained on similar data for a similar task: they tend to learn similar decision boundaries, and a perturbation that exploits a weakness in one boundary often exploits a very similar weakness in the other.

---

## 2. Why This Is Surprising

At first glance, transferability shouldn't work at all. Consider everything that can differ between two models:

```
   MODEL A                          MODEL B
   -------                          -------
   Architecture: Random Forest      Architecture: Neural Network
   Training data: Dataset X          Training data: Dataset Y (different
                                      collection, different samples)
   Hyperparameters: depth=10,         Hyperparameters: 4 layers, Adam
   100 trees                          optimizer, dropout=0.3
   Trained by: Company 1              Trained by: Company 2
   Random seed: 42                    Random seed: 7
```

Two models built this differently should, in principle, end up as two completely unrelated mathematical functions -- with no reason for a perturbation that fools one to have any effect on the other. And yet, empirically, adversarial examples transfer at surprisingly high rates -- often 60-95%+ transfer success between models trained on the same task, even across very different architectures. This result (first widely publicized by Szegedy et al. in 2013 and studied extensively since, including cross-architecture and cross-technique studies by Papernot et al. and others) reshaped how the security community thinks about black-box ML attacks.

---

## 3. Why It Happens: Shared Decision Boundary Geometry

The explanation comes down to one central idea: **models trained on the same task, from the same or similar underlying data distribution, tend to learn similar decision boundaries**, because they are all trying to solve the same underlying problem using signals present in the same underlying data.

### 3.1 The Data Determines the Boundary More Than the Algorithm Does

If "spam" emails in the real world genuinely tend to have lots of exclamation marks, dollar signs, and urgency-language, then *any* reasonably good spam classifier -- whether it's Naive Bayes, a Random Forest, or a deep neural network -- will end up drawing its decision boundary roughly along the same lines: "more of these signals pushes toward spam." The algorithms differ, but the *statistical structure of the real world they're both trying to model* is the same. That shared structure is what produces shared boundary geometry.

```
   TWO DIFFERENT MODELS, ROUGHLY SIMILAR BOUNDARIES

   Model A's boundary (Random Forest)     Model B's boundary (Neural Net)
   feature_2                              feature_2
      ^                                       ^
      |     BENIGN                           |     BENIGN
      |    o   o                              |    o    o
      |  o    o                               |  o     o
      |________________  <- boundary A        |_____________.-  <- boundary B
      |  x   x   \                            |  x    x    /
      |    x    x \                           |    x     x/
      |       MALICIOUS                       |      MALICIOUS
      +--------------> feature_1               +---------------> feature_1

   Different algorithms, different exact shapes -- but both boundaries
   run through roughly the same region of feature space, because both
   are separating the SAME two classes based on the SAME underlying
   data signal.
```

An adversarial perturbation that nudges a malicious sample from deep inside the "malicious" region to just past boundary A will very often *also* land past boundary B, precisely because the two boundaries occupy similar territory in feature space.

### 3.2 High-Dimensional Geometry Makes This Worse (for Defenders)

In high-dimensional feature spaces (which is most real ML problems -- image pixels, byte n-grams, dozens or hundreds of extracted features), decision boundaries tend to be relatively flat/linear in the local region around any given data point, even for complex non-linear models. Research on this phenomenon (again tracing back to Goodfellow et al.'s explanation of adversarial examples via linear behavior in high dimensions) suggests that many different models, when locally approximated near a specific sample, behave like similar linear functions -- which is exactly the condition that produces similar gradients, and therefore similar "most efficient direction to cross the boundary."

---

## 4. The Math Intuition: Similar Functions, Similar Gradients

Building on the [gradient intuition from the previous file](../01-white-box-vs-black-box-attacks/white-box-vs-black-box-attacks.md#5-the-math-you-need-gradients-simply): recall that the gradient tells you which direction, for each input feature, increases a model's output fastest.

If Model A and Model B have learned *similar decision boundaries*, then near any given data point, their gradients will tend to point in **similar directions** -- even though the exact numeric values of their weights are completely different.

```
   GRADIENT DIRECTIONS FROM TWO DIFFERENT MODELS, SAME SAMPLE

           Model A's gradient          Model B's gradient
                  ^                           ^
                   \                         /
                    \                       /
                     \                     /
                      \                   /
                       \                 /
                        \               /
                         (sample point)

   The two arrows aren't identical, but they point in a broadly
   similar direction -- "increase feature_2, decrease feature_1" --
   because both models learned that feature_2 is the stronger signal
   for this task. A perturbation built from Model A's gradient will
   often still move you meaningfully in the direction Model B's
   gradient would have recommended too.
```

This is the mathematical crux of transferability: an adversarial perturbation is, at its core, "a step in the gradient direction." If two models' gradients (at the relevant point in feature space) are correlated rather than random and unrelated, a perturbation computed from one model's gradient will still make progress against the other model's boundary.

---

## 5. Worked Numeric Example: Two Classifiers, One Adversarial Example

Let's extend the toy malware detector from the previous file and show transferability numerically.

### 5.1 Two Independently "Trained" Toy Models

Same two features as before: `x1` = suspicious API calls (0-10 scale), `x2` = file entropy (0-10 scale). Imagine two different teams built two different linear classifiers on similar (but not identical) malware datasets:

```
Model A (the attacker's SURROGATE -- fully known to the attacker):
   score_A(x1, x2) = 2*x1 + 3*x2 - 20
   MALICIOUS if score_A > 0

Model B (the REAL TARGET -- attacker has no internal access, black-box only):
   score_B(x1, x2) = 2.5*x1 + 2*x2 - 21
   MALICIOUS if score_B > 0
```

Notice these are *not* the same model. Model B weighs `x1` (API calls) more heavily and `x2` (entropy) less heavily than Model A -- representing the realistic situation where two teams, using different data and slightly different feature engineering, arrive at different but related decision functions for the same underlying problem (detecting malware).

### 5.2 Starting Point

A malware sample:

```
x1 = 8, x2 = 6

score_A = 2*8 + 3*6 - 20 = 16 + 18 - 20 = 14   --> MALICIOUS (A agrees)
score_B = 2.5*8 + 2*6 - 21 = 20 + 12 - 21 = 11 --> MALICIOUS (B agrees)
```

Both models currently agree: this file is malicious.

### 5.3 Attacker Crafts an Evasion Against the Surrogate (Model A) Only

The attacker has full white-box access to Model A (their own surrogate) but zero internal access to Model B (the real target). Using the gradient of Model A, `(2, 3)`, exactly as in the previous file's worked example:

```
Step 1: x1 = 8 - 2 = 6,   x2 = 6 - 3 = 3
   score_A = 2*6 + 3*3 - 20 = 12 + 9 - 20 = 1     (still MALICIOUS on A, close)

Step 2: x1 = 6 - 1 = 5,   x2 = 3 - 1.5 = 1.5
   score_A = 2*5 + 3*1.5 - 20 = 10 + 4.5 - 20 = -5.5   --> BENIGN on A. Success against the surrogate.
```

The adversarial sample the attacker settled on: `x1 = 5, x2 = 1.5`.

### 5.4 Does It Transfer to the Real Target (Model B)?

The attacker has never queried Model B during crafting. Let's check what Model B (the real, unseen target) thinks of this same adversarial sample:

```
score_B(5, 1.5) = 2.5*5 + 2*1.5 - 21 = 12.5 + 3 - 21 = -5.5

-5.5 <= 0 --> BENIGN on Model B too!
```

**The adversarial example transferred.** Even though the attacker never had access to Model B and never queried it, the perturbation crafted purely against their own surrogate (Model A) also fooled the real target (Model B) -- because both models learned a broadly similar decision boundary for the same underlying task (both treat high API-call counts and high entropy as suspicious; they just weigh the two features slightly differently).

### 5.5 When Would Transfer Fail?

To make the contrast concrete, imagine a third, very different Model C that (unrealistically, but for illustration) learned to key almost entirely off a feature the attacker's surrogate never touched -- say it barely uses `x1`/`x2` at all and instead relies overwhelmingly on a third feature (e.g., file size) that our two-feature surrogate doesn't model. In that case, the perturbation optimized purely for `(x1, x2)` might do nothing to change Model C's score, and transfer would fail. This illustrates the core rule of thumb: **transfer succeeds to the degree that the two models rely on similar features and learned similar boundaries; it fails when the models rely on substantially different signals.**

---

## 6. The Surrogate Attack Pipeline

Transferability is what makes the following black-box strategy (previewed in the last file, and also in the Module 1 security-angle discussion of model stealing) actually work in practice:

```
   THE SURROGATE / TRANSFER ATTACK PIPELINE

   +----------------------+
   | 1. Identify the task |   e.g. "this is a malware classifier" or
   |    and likely data    |   "this is a spam filter for email"
   +-----------+----------+
               |
               v
   +----------------------+
   | 2. Gather training     |   Public datasets for the same task, OR
   |    data for a surrogate|   query the real target repeatedly and
   |                        |   record (input, output) pairs (this
   |                        |   step is itself a model-extraction /
   |                        |   model-stealing attack)
   +-----------+----------+
               |
               v
   +----------------------+
   | 3. Train your own     |   Any reasonable architecture -- it does
   |    surrogate model    |   NOT need to match the target's exact
   |                        |   architecture for transfer to work
   +-----------+----------+
               |
               v
   +----------------------+
   | 4. White-box attack   |   Full gradient access to YOUR surrogate
   |    the surrogate       |   -- use FGSM, PGD, etc. (see previous
   |                        |   file, and Module 9 for details)
   +-----------+----------+
               |
               v
   +----------------------+
   | 5. Transfer the        |   Submit the crafted adversarial example
   |    adversarial example |   to the REAL target and observe
   |    to the real target  |   whether it also fools it
   +-----------+----------+
               |
               v
   +----------------------+
   | 6. (Optional) Refine   |   If transfer partially fails, use the
   |    using ensemble       |   real target's response as limited
   |    surrogates or        |   feedback, or attack an ENSEMBLE of
   |    limited queries       |   several surrogates simultaneously to
   |                        |   improve robustness of the transfer
   +----------------------+
```

The critical insight: **step 5 requires zero (or very few) queries to the real target.** All the expensive, iterative work (steps 3-4) happens entirely offline, against a model the attacker fully controls. This is why transfer-based attacks are the preferred strategy whenever the real target has strict rate limits, charges per query, or actively monitors for suspicious query patterns.

---

## 7. Factors That Affect Transferability

Not every pair of models transfers equally well. Empirically and theoretically, several factors influence how likely an adversarial example is to transfer:

| Factor | Effect on Transferability | Why |
|---|---|---|
| **Similar training data / same task domain** | Increases transfer | Models exposed to similar statistical patterns learn similar boundaries |
| **Similar feature set** | Increases transfer | Perturbations built for features A and B won't help if the target ignores A and B entirely |
| **Similar model architecture (e.g., both CNNs)** | Increases transfer somewhat | Similar architectures tend to have similar inductive biases and learn similar internal representations, though this matters less than shared data/features |
| **Larger perturbation budget (epsilon)** | Increases transfer | A bigger "push" is more likely to clear both boundaries, even if they don't perfectly align |
| **Ensemble surrogate (attacking several surrogates at once, averaging the direction)** | Increases transfer | Reduces the chance the perturbation is overly tuned to quirks of just one surrogate; targets the *shared* direction across multiple models |
| **High confidence / large margin on the surrogate** | Increases transfer | A perturbation that pushes deep past the surrogate's boundary (not just barely across it) has more "safety margin" for a slightly different target boundary too |
| **Model regularization/robustness (e.g., adversarial training on the target)** | Decreases transfer | A target explicitly hardened against adversarial examples has a different, more defensive boundary shape that ordinary surrogates won't reproduce |
| **Very different feature engineering pipelines** | Decreases transfer | If the target computes wildly different derived features from the raw input, a surrogate built on a different feature pipeline may optimize the "wrong" representation entirely |

---

## 8. Transferability Across Model Families

A common misconception is that transferability only works between very similar models (e.g., two neural networks). In practice, evasion transfer has been repeatedly demonstrated across quite different model families, as long as the underlying task and data are related:

| Surrogate Model Type | Target Model Type | Typical Transfer Behavior |
|---|---|---|
| Neural Network | Neural Network (different architecture) | Strong transfer -- most-studied case |
| Neural Network | Random Forest / Gradient Boosted Trees | Moderate transfer -- less consistent than NN-to-NN, but documented, especially for tabular/feature-based tasks |
| Linear model (e.g., logistic regression) | Neural Network | Weaker but non-zero transfer -- works best when the true decision boundary is close to linear in the relevant region |
| Any model | Ensemble of several model types | Attacking against an ensemble surrogate tends to produce perturbations that transfer better to a broader range of unknown target types |

The practical takeaway: **when in doubt about the target's architecture, train a small ensemble of diverse surrogate model types, and craft a perturbation that fools all of them simultaneously.** This tends to find the "genuinely shared" weakness in the task rather than a quirk specific to one model type.

---

## 9. Defensive Implications

If you are building or hardening an ML-based defense, transferability tells you something uncomfortable: **you cannot assume that hiding your model is sufficient protection.** An attacker who never sees your weights can still succeed by attacking a surrogate.

Practical mitigations that specifically target the transferability weakness:

- **Adversarial training**: deliberately train the target model on adversarial examples (including ones generated against surrogates) so its decision boundary becomes more robust and less similar to an "off-the-shelf" boundary that a naive surrogate would replicate.
- **Ensemble/diverse defenses**: deploy multiple, architecturally diverse models and require agreement, making the "shared boundary geometry" assumption attackers rely on less reliable.
- **Restricting output granularity**: returning only a coarse label (not fine-grained confidence scores) makes it harder for an attacker to fine-tune a surrogate's fidelity to your real model via extraction queries, indirectly limiting how good a surrogate they can build.
- **Detecting extraction-style querying**: since surrogate-building often starts with a model-extraction phase (many systematic queries), monitoring for that query pattern can catch the attack before the transfer step ever happens.

None of these fully eliminate transferability, but they raise the cost and reduce the reliability of a transfer-based attack.

---

## 10. Security Angle

Transferability is the single biggest reason black-box ML attacks are practical at scale, and it fundamentally changes the risk calculus for any organization deploying "security through obscurity" for their ML models.

```
   THE FALSE COMFORT OF "THEY CAN'T SEE OUR MODEL"

   Company's assumption:                Reality with transferability:
   +---------------------+              +---------------------------+
   | "Our fraud model's   |              | Attacker doesn't need to  |
   |  weights are secret, |   ---X-->    | see your weights. They    |
   |  so attackers can't  |              | build a surrogate from    |
   |  craft evasions."     |              | public data + your task,  |
   +---------------------+              | attack IT, and the result |
                                         | transfers to your real    |
                                         | model anyway.              |
                                         +---------------------------+
```

**For red teamers**: transferability is your justification for treating "the client's model internals are confidential" as *not* a blocker for a realistic evasion assessment. You can build a surrogate from public malware/spam/fraud datasets covering the same task, craft evasions white-box against it, and report on the (often high) transfer success rate against the real target -- exactly reproducing what a real-world black-box adversary would do.

**For defenders/blue teams**: never treat model secrecy as your only or primary line of defense. Combine adversarial training, ensemble diversity, output restriction, and query monitoring, because a determined attacker with zero internal access can still very plausibly succeed via a surrogate.

---

## 11. Key Takeaways

- **Transferability** is the phenomenon where an adversarial example crafted against one model often also fools a different model trained on a similar task, even without any shared architecture, training data, or team.

- **The root cause is shared decision boundary geometry**: models solving the same problem on similar data tend to learn similar boundaries, because the real-world statistical structure of the data -- not the specific algorithm -- largely determines where the boundary must fall.

- **Mathematically**, similar boundaries imply correlated gradients near a given data point, so a perturbation computed from one model's gradient makes progress against the other model's boundary too.

- In the worked example, an adversarial sample `(x1=5, x2=1.5)` crafted purely against a surrogate (`score_A`) also flipped the real target (`score_B`) to BENIGN -- without a single query to the real target during crafting.

- **The surrogate attack pipeline** (gather data -> train surrogate -> white-box attack the surrogate -> transfer to the real target) is the standard practical strategy for black-box evasion when direct querying is limited, costly, or risky.

- **Transfer success depends on** shared data/features/task (helps a lot), similar architecture (helps somewhat), larger perturbation budgets and ensemble surrogates (both help), and target-side adversarial training or heavy architectural divergence (both hurt transfer).

- **Defensively**, secrecy of model internals is not sufficient protection. Adversarial training, ensemble defenses, restricting output granularity, and monitoring for extraction-style querying are the concrete countermeasures.

---

*Next up: [Feature Obfuscation (GoodWords Attack Methodology)](../03-feature-obfuscation-goodwords-attack/feature-obfuscation-goodwords-attack.md) -- a foundational, often gradient-free evasion technique for classifiers that key on the presence or absence of specific trigger features, and how injecting or removing such features can cross a decision boundary directly.*
