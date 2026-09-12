# Clean-Label Attacks

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Data Attacks | Section: Feature Attacks

---

## Table of Contents

1. [What is a Clean-Label Attack?](#1-what-is-a-clean-label-attack)
2. [Why "Clean" Labels Are So Dangerous](#2-why-clean-labels-are-so-dangerous)
3. [Where Feature Attacks Fit in the ML Pipeline](#3-where-feature-attacks-fit-in-the-ml-pipeline)
4. [How Clean-Label Poisoning Actually Works](#4-how-clean-label-poisoning-actually-works)
5. [Feature Collision -- The Core Mechanism](#5-feature-collision----the-core-mechanism)
6. [Notable Techniques](#6-notable-techniques)
7. [Worked Example -- Poisoning an Image Classifier](#7-worked-example----poisoning-an-image-classifier)
8. [Clean-Label vs. Label Flipping vs. Backdoor -- Comparison Table](#8-clean-label-vs-label-flipping-vs-backdoor----comparison-table)
9. [Detecting Clean-Label Attacks](#9-detecting-clean-label-attacks)
10. [Security Angle](#10-security-angle)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What is a Clean-Label Attack?

### The Analogy

Recall the label-flipping analogy: a coworker swapping the "PASS/FAIL" answer on the back of a flashcard while leaving the photo untouched. A **clean-label attack** is a different, sneakier move entirely: the coworker leaves the answer ("PASS" or "FAIL") **completely correct and untouched** on every card. Instead, they very subtly retouch the *photos themselves* -- adjusting lighting, adding a barely-noticeable smudge, tweaking a shadow -- in a handful of cards, in a way that is consistent with the correct label (a genuinely defective product that really does say "FAIL," but doctored so that its visual pattern quietly nudges the trainee's mental model in a specific, exploitable direction). A human quality auditor checking "does this photo match its label?" sees a perfect match every time. There is nothing to catch by checking the answer key, because the answer key was never touched.

That is the essence of a clean-label attack.

### The Formal Definition

A **clean-label attack** (also called clean-label poisoning) is a data poisoning technique in which the attacker injects poisoned samples into the training set whose **labels are entirely correct and legitimate**, but whose **features (the raw input data itself -- pixels, byte sequences, numeric values) have been subtly and deliberately perturbed**. The perturbation is crafted so that, once the model is trained on it, the model's learned decision boundary shifts in a way that benefits the attacker -- for example, causing a specific target input to be misclassified at inference time, or causing a whole class of future inputs to be misclassified.

The critical property: **the poisoned samples are correctly labeled**. Anyone auditing "does this sample's label match what a human would say the sample is?" finds nothing wrong, because the answer is genuinely yes -- the perturbation is small enough that the sample still visually/semantically belongs to its stated class.

```
   LABEL FLIPPING                        CLEAN-LABEL ATTACK

   Feature: UNCHANGED                    Feature: SUBTLY PERTURBED
   Label:   CHANGED (wrong)              Label:   UNCHANGED (correct!)

   +------------------+                  +------------------+
   | [photo of a      |                  | [photo of a cat, |
   |  real cat]       |                  |  imperceptibly   |
   |                  |                  |  perturbed]      |
   | Label: DOG  (!)  |                  | Label: CAT (OK)  |
   +------------------+                  +------------------+
     Caught by:                            Caught by:
     - label auditing                      - label auditing? NO -- label is
     - "does label match                     genuinely correct
        the content?"                      - Requires FEATURE-level
                                              inspection to catch
```

---

## 2. Why "Clean" Labels Are So Dangerous

The entire value proposition of a clean-label attack, from the attacker's point of view, is **evading exactly the defenses that catch label-based poisoning**:

- **Human label auditors are useless against it.** If a security team spot-checks "does this training image actually look like a cat?" for a poisoned cat image, the answer is genuinely yes -- it does look like a cat. There is no label discrepancy to notice.
- **Automated label-consistency checks are also blind to it.** Techniques like nearest-neighbor label agreement (mentioned in the Label Flipping file) check whether a sample's *label* matches its *neighbors' labels* -- but a clean-label poison sample's label already matches its (correct) class, so this check passes cleanly too.
- **The attack hides in exactly the place fewer defenses look**: the numeric feature values themselves, which are usually assumed to be "whatever the label says they are" once the label passes review.

This is why clean-label attacks are considered a meaningfully more advanced and dangerous technique than plain label flipping, even though both are forms of data poisoning.

---

## 3. Where Feature Attacks Fit in the ML Pipeline

Like label-based poisoning, clean-label attacks are injected during **Data Collection** or **Data Preprocessing**. The difference is *what* gets manipulated at that stage:

```
+----------------+     +----------------+     +----------------+
|  1. PROBLEM    |---->|  2. DATA       |---->|  3. DATA       |
|  DEFINITION    |     |  COLLECTION    |     |  PREPROCESSING |
+----------------+     +----------------+     +----------------+
                              ^                        ^
                              |                        |
                    [ Attacker submits/plants   [ Attacker perturbs
                      samples with genuinely      feature values during
                      correct labels but          preprocessing/feature
                      subtly perturbed             engineering, if they
                      features -- e.g. an          have write access to
                      "optimized" image             this stage of a
                      uploaded to a public          shared/compromised
                      dataset or scraped            pipeline
                      website ]
                                                        |
                                                        v
+----------------+     +----------------+     +----------------+
|  6. DEPLOYMENT |<----|  5. MODEL      |<----|  4. MODEL      |
|  & MONITORING  |     |  EVALUATION    |     |  TRAINING      |
+----------------+     +----------------+     +----------------+
                                                        ^
                                              Model learns a subtly
                                              shifted decision boundary,
                                              exploitable by the attacker
                                              at inference time.
```

Because the labels are correct, clean-label poison samples can even be **submitted publicly** -- e.g., uploaded as ordinary-looking images to social media, stock photo sites, or public datasets that will later be scraped for training data -- without needing any special access to the victim's pipeline at all. This is a major reason clean-label attacks are relevant to large models trained on scraped internet-scale data.

---

## 4. How Clean-Label Poisoning Actually Works

At a high level, an attacker executing a clean-label attack needs to solve an **optimization problem**: find a small perturbation to a correctly-labeled sample such that, after training, the model's decision boundary shifts favorably for the attacker, while the perturbation stays small enough that:

1. The sample still visually/semantically matches its label to a human.
2. The perturbation is small enough to survive normal data augmentation/preprocessing (resizing, compression, cropping) without disappearing.

```
STEP 1: DEFINE THE ATTACKER'S GOAL
   Example: "I want a specific target photo of Alice (which
   the attacker controls or knows will be seen at inference
   time) to be misclassified as Bob by a face-recognition model."

STEP 2: SELECT A BASE CLASS TO POISON
   Attacker picks a class whose samples they can inject into the
   training set with a CORRECT label -- e.g., they submit photos
   genuinely labeled "Bob" (because they are, in fact, of Bob).

STEP 3: CRAFT THE PERTURBATION
   Using optimization (e.g. gradient-based methods against a
   substitute/surrogate model that approximates the target model),
   the attacker computes a small perturbation to add to the "Bob"
   photos such that, in FEATURE SPACE, they sit unusually close to
   Alice's target photo -- while remaining visually indistinguishable
   from ordinary photos of Bob.

STEP 4: INJECT
   The perturbed-but-correctly-labeled "Bob" photos are submitted
   into the training pipeline (e.g. via a public photo-sharing
   channel that feeds the training set, or direct pipeline access).

STEP 5: MODEL TRAINS NORMALLY
   The model learns that the region of feature space near these
   "Bob" photos --- which, by the attacker's design, is also near
   Alice's target photo --- belongs to class "Bob."

STEP 6: EXPLOIT AT INFERENCE TIME
   When Alice's target photo (or the attacker wearing/producing
   something similar) is submitted at inference time, the model
   misclassifies it as "Bob," because its feature-space neighborhood
   was poisoned to belong to Bob's class.
```

---

## 5. Feature Collision -- The Core Mechanism

The most well-known technical mechanism behind clean-label attacks is called **feature collision**. The idea:

- Every input, once passed through the early/middle layers of a model (or through hand-engineered feature extraction), gets mapped to a point in a high-dimensional **feature space**.
- Two inputs that look nothing alike to a human (a photo of Bob vs. a photo of Alice) can, after feature extraction, end up mapped to points that are numerically very close together, if an attacker deliberately engineers one of them to do so.
- The attacker's crafted "Bob" sample is optimized so that its feature-space representation nearly *collides* with the target's ("Alice's") feature-space representation, while its pixel-space appearance still looks like an ordinary photo of Bob.

```
                     FEATURE SPACE (simplified 2D view)

        Before poisoning:                After poisoning:

        Bob's samples                    Bob's samples (incl. poison)
             o o o                            o o o
              o o                               o o
                                                    \
        Alice's samples                            *  <- poisoned "Bob"
              x x x                                     sample now sits
               x x                                       right next to
                                                          Alice's target
        Alice's target                    Alice's target sample
        sample:  X                        sample:  X
        (far from Bob's cluster)          (now inside/near Bob's
                                            cluster because of the
                                            feature-collision poison)

        Decision boundary cleanly         Decision boundary now
        separates Bob from Alice          misclassifies Alice's target
                                           sample as "Bob"
```

Because the poison sample's *label* ("Bob") was always correct for its *appearance* ("looks like Bob"), nothing about the sample itself is suspicious. The exploit lives entirely in the mathematical relationship between feature-space positions, which is invisible without specialized tooling.

---

## 6. Notable Techniques

You do not need to reproduce these from scratch for the exam, but you should recognize the names and general ideas, as they represent the well-documented, publicly known research on this topic:

| Technique | Core Idea |
|---|---|
| **Feature Collision** | Optimize a poison sample so its feature-space representation collides with a specific target sample's representation, while keeping the poison sample's correct label and visual appearance (described above). |
| **Convex Polytope Attack** | Instead of colliding with a single target point, craft multiple poison samples whose feature-space representations form a "polytope" (a bounding shape) that encloses the target sample, making the attack more robust to the victim using slightly different model weights/architectures than the attacker assumed. |
| **Bullseye Polytope Attack** | A refinement of the polytope idea that more efficiently centers the target inside the poison samples' feature-space hull, improving attack success rate with fewer poison samples. |
| **Witches' Brew / Gradient Matching** | Instead of targeting a specific feature-space location, craft poison samples whose *training gradients* closely match the gradient the attacker wants the model to have learned -- effectively reverse-engineering "what training data would have produced this exact bad behavior" and injecting samples that approximate it. |

The specific math is graduate-level adversarial ML research; what matters for this certification is understanding the **shared principle**: labels stay correct, features are optimized (usually via gradient-based methods against a surrogate model) to move the model's learned boundary in the attacker's favor.

---

## 7. Worked Example -- Poisoning an Image Classifier

To keep this concrete and numeric (illustrative, not from a specific published paper), consider a simplified "cat vs. dog" image classifier.

### Setup

- Training set: **N = 8,000** images (4,000 cats, 4,000 dogs), all correctly labeled.
- Clean model baseline: 97% test accuracy.
- Attacker's goal: get one specific target photo of a particular cat (which the attacker knows will be submitted at inference time, e.g. it's the attacker's own test photo) misclassified as "dog," without affecting the model's performance on anything else.

### The Attack

The attacker crafts **50 poison images (0.6% of the training set)**. Each poison image:

- Is a genuine, correctly-labeled photo of an ordinary dog (label: "dog" -- 100% correct).
- Has been subtly perturbed (small pixel-level changes, imperceptible to a casual viewer) so that, in the model's internal feature space, these 50 "dog" images sit unusually close to the attacker's target cat photo.

```
Illustrative overall test-set result after retraining on poisoned data:

 Accuracy on the full 2,000-image test set: 96.8%   (vs. 97.0% clean baseline
                                                       -- essentially unchanged)

 Specific target photo (the attacker's chosen cat image):
   Before poisoning: correctly classified "cat" with 99% confidence
   After poisoning:  misclassified as "dog" with 91% confidence

 Performance on all OTHER cat photos in the test set: unchanged (~97%)
```

Just 50 correctly-labeled, feature-perturbed images out of 8,000 (0.6%) achieved a **surgical misclassification of one specific target input**, without measurably affecting the model's accuracy on anything else -- including other cat photos. This precision is the hallmark of clean-label / feature-collision attacks, and it is what makes them attractive for high-stakes targeted scenarios (e.g., bypassing a face-recognition access control system for one specific person, or one specific attacker-controlled artifact).

---

## 8. Clean-Label vs. Label Flipping vs. Backdoor -- Comparison Table

| Property | Label Flipping | Targeted Label Attack | Clean-Label Attack | Backdoor/Trojan (next file) |
|---|---|---|---|---|
| **Labels** | Changed (incorrect) | Changed (incorrect), for a subset | Unchanged (correct) | Usually unchanged for the trigger class, or a small consistent shift |
| **Features** | Unchanged | Unchanged | Subtly perturbed | Perturbed with a specific trigger pattern |
| **Requires model/surrogate access to craft?** | No | No | Usually yes (gradient-based optimization) | Usually yes |
| **Detectable via label audit?** | Yes, if sampled | Hard (rare/small subset) | No -- labels are genuinely correct | No -- labels can be correct too |
| **Goal** | Broad damage | Precise blind spot via labels | Precise blind spot via features | Reliable attacker-controlled trigger -> output mapping |
| **Sophistication required** | Low | Low-medium | Medium-high (needs optimization/surrogate model) | Medium-high |

---

## 9. Detecting Clean-Label Attacks

Because label auditing is useless here, defenses must look at the **feature space** itself:

| Technique | How It Helps | Limitation |
|---|---|---|
| **Feature-space outlier/anomaly detection** | Flag samples whose feature-space position is unusually close to samples of a *different* class, or unusually far from the centroid of their *own* class | Sophisticated attacks (convex polytope, bullseye) deliberately try to blend in with the legitimate class distribution |
| **Activation clustering** | Cluster training samples by their internal model activations (not raw features); poisoned samples sometimes form a distinguishable sub-cluster within their labeled class | Requires access to internal model activations, and clusters can be subtle |
| **Spectral signatures** | Analyze the statistical spectrum (via techniques like singular value decomposition) of feature representations within a class; poison samples can leave a detectable statistical signature | Computationally more involved; still an active research area, not foolproof |
| **Gradient-based influence analysis** | Estimate which training samples most influence a specific prediction (similar to the technique mentioned for targeted label attacks); investigate top-influence samples for a suspicious target | Expensive at scale |
| **Robust/randomized training procedures** | Techniques like differential privacy or randomized smoothing during training can reduce sensitivity to small numbers of highly-optimized poison points | Can reduce model accuracy/utility as a tradeoff |
| **Data provenance restrictions** | Limit training to vetted, trusted sources rather than freely scraped public data | Not always feasible for large-scale internet-trained models |

---

## 10. Security Angle

Clean-label attacks matter enormously for anyone doing offensive AI work because:

- **They defeat the most intuitive defense (label auditing) by design.** Any organization whose poisoning defense strategy is "we manually check labels" is fully exposed to this class of attack.
- **They can be delivered through entirely public channels.** Because the poison samples carry correct, legitimate labels, an attacker can often submit them through ordinary public content (photos, code samples, text posts) that gets scraped into training sets, with no need to compromise any internal system.
- **They enable surgical, high-value targeting** -- as shown in the worked example, an attacker can compromise the classification of one specific target input while leaving the model's behavior on everything else statistically indistinguishable from clean.
- **They require the attacker to have (or approximate) knowledge of the model's feature space**, typically via a surrogate/substitute model -- this is a recurring theme across many advanced ML attacks (also seen in adversarial examples and model stealing) and is a skill worth building generally.
- When assessing a target's defenses, ask: *does this team's data validation go beyond checking that labels match content? Do they ever inspect the feature-space geometry of their training data?* Almost none do by default, which is exactly the gap clean-label attacks exploit.

---

## 11. Key Takeaways

- **A clean-label attack poisons the features of training samples while keeping their labels completely correct**, defeating label-based auditing by design.
- **The core mechanism is often "feature collision"**: crafting a correctly-labeled sample whose internal feature-space representation is deliberately engineered to sit near an unrelated target sample, shifting the model's learned decision boundary in the attacker's favor.
- **Notable published techniques** (Feature Collision, Convex Polytope, Bullseye Polytope, Witches' Brew/Gradient Matching) share the same principle: correct labels, optimized features, usually crafted against a surrogate model.
- **The worked example showed that 50 correctly-labeled, subtly perturbed images (0.6% of an 8,000-image dataset)** achieved a surgical misclassification of one specific target input with no measurable effect on overall accuracy.
- **Because labels are genuinely correct, this attack can often be delivered via public, unprivileged channels** (e.g., publicly posted images later scraped for training), making it relevant even against organizations with no direct pipeline compromise.
- **Defense requires feature-space analysis** (outlier detection, activation clustering, spectral signatures, influence functions) since there is nothing wrong to find by checking labels alone.

*Next up: Trojan / Backdoor Attacks -- where the attacker goes a step further, embedding a hidden trigger pattern into the model during training so that a specific input pattern reliably causes a specific attacker-chosen output at inference time, while the model behaves normally on all other inputs.*
