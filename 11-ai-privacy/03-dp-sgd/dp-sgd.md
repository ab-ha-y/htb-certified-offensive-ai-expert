# DP-SGD (Differentially Private Stochastic Gradient Descent)

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Privacy | Section: DP-SGD

---

## Table of Contents

1. [Recap: What Is Vanilla SGD?](#1-recap-what-is-vanilla-sgd)
2. [Why Vanilla SGD Leaks Privacy](#2-why-vanilla-sgd-leaks-privacy)
3. [What DP-SGD Changes](#3-what-dp-sgd-changes)
4. [Step 1: Per-Example Gradient Clipping](#4-step-1-per-example-gradient-clipping)
5. [Step 2: Calibrated Noise Addition](#5-step-2-calibrated-noise-addition)
6. [The Full DP-SGD Algorithm](#6-the-full-dp-sgd-algorithm)
7. [Worked Numeric Example](#7-worked-numeric-example)
8. [DP-SGD vs. Vanilla SGD](#8-dp-sgd-vs-vanilla-sgd)
9. [Why This Protects Against Membership Inference and Model Inversion](#9-why-this-protects-against-membership-inference-and-model-inversion)
10. [Practical Considerations and Limitations](#10-practical-considerations-and-limitations)
11. [Privacy Angle -- Why This Matters](#11-privacy-angle----why-this-matters)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. Recap: What Is Vanilla SGD?

Before we can understand what makes DP-SGD "differentially private," we need to be crystal clear on ordinary (non-private) **Stochastic Gradient Descent (SGD)**, the workhorse training algorithm behind most neural networks.

### The Analogy

Recall the "rolling a ball downhill" analogy for gradient descent from Module 1: the **loss function** is a landscape of hills and valleys, where height represents "how wrong the model currently is." Training means repeatedly nudging the model's parameters (weights) a little bit in the downhill direction, so the loss gets smaller and smaller.

**Stochastic** means "based on randomness/samples" -- instead of looking at the *entire* training set before taking each step (which would be extremely slow for large datasets), SGD looks at just a small random **batch** of examples at a time, computes the average downhill direction for that batch, and takes a step. Repeat this over and over across the whole dataset (each full pass being one **epoch**), and the model gradually improves.

### The Vanilla SGD Loop

```
    VANILLA SGD (no privacy)
    ==========================

    Initialize model weights W randomly

    For each training step:
        1. Sample a random batch B of examples from the training set
        2. For each example x_i in B:
               compute gradient g_i = derivative of loss w.r.t. W, for x_i
               ("gradient" = the direction/steepness of the slope at
                this example's exact position on the loss landscape)
        3. Average the gradients across the batch:
               g_avg = (1/|B|) * sum(g_i for all i in B)
        4. Update weights:
               W = W - learning_rate * g_avg
        5. Repeat until loss stops improving (or a fixed number of steps)
```

**Key term -- Gradient**: a vector (a list of numbers, one per model parameter) that tells the optimizer "which direction, and how steeply, to adjust each parameter to reduce the loss for this specific example." Bigger gradient magnitude = the model is currently very wrong about this example and needs a bigger correction.

---

## 2. Why Vanilla SGD Leaks Privacy

Here is the crucial connection back to Section 1 (Membership Inference) of this module: **the gradient computed for each individual training example directly reflects that example's specific characteristics.**

- If an example is unusual, mislabeled, or rare, its gradient can be very large (the model needs a big correction to fit it).
- If an example is a duplicate of something the model has already learned well, its gradient can be tiny (the model is already right, so little correction is needed).
- Over many training steps, the model's final weights end up encoding the *cumulative influence* of every single gradient computed along the way -- including outlier and unusual examples whose gradients had an outsized effect.

```
    THE PRIVACY LEAK IN VANILLA SGD

    Training Example "Bob's rare medical record"
                    |
                    v
         Large, distinctive gradient
         (because the model was very
          wrong about this unusual case)
                    |
                    v
         Weight update is disproportionately
         influenced by Bob's specific record
                    |
                    v
         Final model weights partially "encode"
         information distinctive to Bob's record
                    |
                    v
         An attacker probing the model (via membership
         inference, model inversion, or even directly
         inspecting gradients in federated learning
         settings) can pick up on this signature
```

This is exactly the mechanism behind overfitting-driven membership inference from Section 1: unusual or memorable training examples leave an outsized "fingerprint" on the trained model, purely because vanilla SGD has **no limit** on how much any single example's gradient can influence the update.

**DP-SGD's entire purpose is to put a hard mathematical cap on that influence**, converting the informal observation "gradients can leak information about individual examples" into the formal differential privacy guarantee from Section 2 of this module.

---

## 3. What DP-SGD Changes

DP-SGD (introduced by Abadi et al., 2016, "Deep Learning with Differential Privacy") modifies the vanilla SGD loop in exactly **two** places, both happening *before* the gradients get averaged and applied:

```
    VANILLA SGD STEP                       DP-SGD STEP
    ===================                    =============

    1. Sample batch                        1. Sample batch
    2. Compute per-example gradients       2. Compute per-example gradients
                                            2a. >>> CLIP each gradient <<<
                                                (cap its magnitude/length)
                                            2b. >>> ADD NOISE <<< to the
                                                sum/average of clipped
                                                gradients
    3. Average gradients                   3. Average the CLIPPED+NOISED
                                                gradients
    4. Update weights                      4. Update weights
```

The two new steps are:
1. **Per-example gradient clipping** -- covered in Section 4 below.
2. **Calibrated noise addition** -- covered in Section 5 below.

Together, these are exactly the "Gaussian Mechanism" pattern introduced conceptually in Section 2 of this module: **calibrate sensitivity (via clipping), then add proportional noise.**

---

## 4. Step 1: Per-Example Gradient Clipping

### The Problem Clipping Solves

Differential privacy's mathematical guarantee requires being able to bound (put a maximum limit on) exactly how much any single training example could possibly change the outcome. But an unbounded, arbitrarily large gradient from one weird outlier example could, in principle, change the model's weights by an unlimited amount. You cannot add "enough" noise to mask an unbounded signal -- so the signal must first be bounded.

### The Analogy

Imagine a group project where each team member's opinion is supposed to count toward a group decision, but one very loud team member is allowed to shout as forcefully as they want, effectively overriding everyone else's input. **Clipping is the rule "everyone gets exactly the same maximum volume, no matter how strongly they feel."** A team member who mildly disagrees still speaks at their natural (smaller) volume, but nobody is allowed to shout louder than the agreed cap.

### The Mechanism

For each individual example `x_i` in the batch, after computing its gradient `g_i`:

1. Compute the **norm** of the gradient (its overall "length" or magnitude, treating the gradient as a vector of numbers). The most common choice is the L2 norm (Euclidean length): `||g_i||`.
2. Compare `||g_i||` to a chosen **clipping threshold**, `C`.
3. If `||g_i||` is already less than or equal to `C`, leave it unchanged.
4. If `||g_i||` exceeds `C`, **rescale** the gradient down so its norm becomes exactly `C` (same direction, smaller magnitude).

```
    Clipped gradient:

        g_i_clipped = g_i / max(1,  ||g_i|| / C )

    In words:
      - If the gradient's length is already <= C: divide by 1 -> unchanged.
      - If the gradient's length is > C: divide by (||g_i|| / C), which
        shrinks it down until its length is exactly C.
```

```
        GRADIENT CLIPPING VISUALIZED (2D simplification)

                    ^
                    |          * g_3 (huge, unusual example)
                    |         /
                    |        /
        - - - - - - - - - - X  <-- clipped to length C
                    |      /
                    |     * g_2 (already within C, untouched)
                    |    /
        ------------+---/------------------->
                    |  /
                    | * g_1 (already within C, untouched)
                    |

        Clipping circle of radius C
        Every gradient vector is squeezed to fit inside (or on)
        this circle before being used further.
```

### Why This Matters for the Overall Guarantee

After clipping, **no single example's gradient can have a norm larger than `C`**, no matter how unusual or extreme that example is. This gives DP-SGD a known, fixed **sensitivity** (the maximum possible change one example could cause) -- exactly the quantity the Gaussian noise mechanism (Section 5) needs in order to calibrate how much noise is "enough."

---

## 5. Step 2: Calibrated Noise Addition

### The Analogy

Continuing the "loud team member" analogy: even after everyone is capped at the same maximum volume, if the final group decision is announced as "we heard exactly these words from exactly these people," you could still, in principle, work backward and identify individual contributions if you know everyone's positions precisely. **Adding noise is like playing a bit of static over the loudspeaker before the final decision is announced** -- just enough that no individual voice can be reliably picked out of the mix, but not so much that the overall group consensus becomes unintelligible.

### The Mechanism

After clipping every per-example gradient in the batch:

1. **Sum** the clipped gradients across the batch.
2. **Add random noise**, sampled from a **Gaussian (normal) distribution** -- the classic "bell curve," where values near zero are most likely and larger deviations become progressively rarer -- to that sum. The amount of noise (its standard deviation, i.e., how "wide" the bell curve is) is calibrated based on:
   - The clipping threshold `C` (the sensitivity established in Step 1).
   - The desired final epsilon (privacy budget from Section 2).
   - The batch size and total number of training steps (since privacy loss accumulates across steps, via the composability property from Section 2).
3. **Average** this noised sum (divide by the batch size) to get the final gradient used to update the weights.

```
    Noised, averaged gradient for this training step:

        g_noised = ( sum(g_i_clipped for all i in batch)  +  N(0, sigma^2 * C^2) )  /  batch_size

    Where:
      N(0, sigma^2 * C^2) = a random value drawn from a Gaussian
                             (bell-curve) distribution centered at 0,
                             with "spread" (standard deviation) equal
                             to sigma * C.
      sigma                = the noise multiplier -- a hyperparameter
                             you choose; bigger sigma = more noise =
                             stronger privacy, smaller final epsilon.
```

**Why noise is added to the *sum* (not each individual gradient separately)**: the mathematical proof of the privacy guarantee (via the Gaussian mechanism) is calibrated around the *aggregate* sensitivity of the sum, which is capped at `C` per example thanks to clipping. Adding noise once to the aggregate is what the formal DP proof for this mechanism relies on.

---

## 6. The Full DP-SGD Algorithm

Putting Steps 1 and 2 together into the complete loop:

```
    DP-SGD ALGORITHM
    ==================

    Inputs: training set, clipping threshold C, noise multiplier sigma,
            batch size B, learning rate, number of steps T

    Initialize model weights W randomly

    For each of T training steps:

        1. Sample a random batch of B examples

        2. For each example x_i in the batch:
               a. Compute per-example gradient:      g_i
               b. Clip it:                            g_i_clipped =
                                                        g_i / max(1, ||g_i|| / C)

        3. Sum the clipped gradients:
               g_sum = sum(g_i_clipped for all i in batch)

        4. Add Gaussian noise:
               g_noised_sum = g_sum + N(0, sigma^2 * C^2)

        5. Average:
               g_final = g_noised_sum / B

        6. Update weights:
               W = W - learning_rate * g_final

    After all T steps, use a "privacy accountant" (a formal bookkeeping
    tool, e.g., the Moments Accountant or Renyi DP Accountant) to compute
    the TOTAL epsilon spent across all T steps, accounting for composition.

    Output: trained weights W, and a provable (epsilon, delta)-DP guarantee
```

**Why a "privacy accountant" is needed**: recall from Section 2 (Composability) that privacy loss accumulates across multiple queries/steps. Naively adding up epsilon per step across potentially tens of thousands of training steps would produce an enormous, useless total epsilon. Specialized accounting techniques (beyond the scope of this intro module, but good to recognize by name) track the *tighter*, more favorable cumulative epsilon that Gaussian noise composition actually achieves, which is what makes DP-SGD practical at all for deep learning.

---

## 7. Worked Numeric Example

Let's trace through one single (simplified) DP-SGD training step with concrete numbers.

### Setup

- Clipping threshold: `C = 1.0`
- Noise multiplier: `sigma = 1.0`
- Batch size: `B = 4` (tiny, for illustration -- real batches are often hundreds or thousands)
- We are training a simple model with just **2 parameters** for this example, so gradients are 2-dimensional vectors, making norms and clipping easy to compute by hand.

### Step 1: Per-Example Gradients (before clipping)

| Example | Raw Gradient (g_i) | Norm \|\|g_i\|\| | Exceeds C=1.0? |
|---------|----------------------|-------------------|-----------------|
| x_1 | (0.3, 0.4) | sqrt(0.3² + 0.4²) = 0.50 | No |
| x_2 | (0.6, 0.8) | sqrt(0.6² + 0.8²) = 1.00 | No (exactly at C) |
| x_3 | (2.4, 3.2) | sqrt(2.4² + 3.2²) = 4.00 | **Yes** (unusual/outlier example) |
| x_4 | (0.1, 0.1) | sqrt(0.1² + 0.1²) ≈ 0.14 | No |

### Step 2: Clip Each Gradient

For `x_3`, which exceeds `C`: scale factor = `max(1, 4.00 / 1.0) = 4.0`, so:

```
  g_3_clipped = (2.4, 3.2) / 4.0 = (0.6, 0.8)   -->  new norm = 1.0 (exactly C)
```

All others were already within `C`, so they stay unchanged.

| Example | Clipped Gradient |
|---------|--------------------|
| x_1 | (0.3, 0.4) |
| x_2 | (0.6, 0.8) |
| x_3 | (0.6, 0.8) -- rescaled down from (2.4, 3.2) |
| x_4 | (0.1, 0.1) |

**Notice**: the outlier example `x_3`, which originally had 8x the influence of `x_1`, now has exactly the same maximum possible influence as `x_2` and no more than any other example. This is clipping doing its job.

### Step 3: Sum the Clipped Gradients

```
  g_sum = (0.3+0.6+0.6+0.1, 0.4+0.8+0.8+0.1) = (1.6, 2.1)
```

### Step 4: Add Gaussian Noise

Suppose (for this worked example) the random noise draw from `N(0, sigma^2 * C^2) = N(0, 1.0)` happens to produce the vector `(-0.2, 0.3)` in each dimension (in reality each dimension gets its own independent random draw):

```
  g_noised_sum = (1.6 - 0.2, 2.1 + 0.3) = (1.4, 2.4)
```

### Step 5: Average and Update

```
  g_final = (1.4, 2.4) / 4  =  (0.35, 0.6)

  If learning_rate = 0.1:
  W_new = W_old - 0.1 * (0.35, 0.6) = W_old - (0.035, 0.06)
```

**Compare to what vanilla SGD would have done** (no clipping, no noise): summing the *raw* gradients `(0.3+0.6+2.4+0.1, 0.4+0.8+3.2+0.1) = (3.4, 4.5)`, averaged to `(0.85, 1.125)` -- notice this is **dominated by the outlier `x_3`**, which alone contributed more than half the total. DP-SGD's clipping neutralized that outsized influence, and the added noise further obscures exactly how much any one example (including `x_3`) actually contributed to this specific step.

---

## 8. DP-SGD vs. Vanilla SGD

| Aspect | Vanilla SGD | DP-SGD |
|--------|-------------|--------|
| **Per-example gradient handling** | Used as-is, no limit on magnitude | Clipped to a fixed maximum norm `C` |
| **Noise** | None | Gaussian noise added to the summed gradient each step |
| **Outlier influence** | Unbounded -- rare/unusual examples can dominate updates | Bounded -- capped at the same maximum as every other example |
| **Formal privacy guarantee** | None | Provable (epsilon, delta)-differential privacy |
| **Computational cost** | Standard | Higher -- must compute *per-example* gradients (not just batch-averaged gradients), which is more memory/compute intensive |
| **Accuracy (typical)** | Baseline | Usually somewhat lower for the same architecture/training budget, due to noise and clipping -- the privacy-utility tradeoff from Section 2 in action |
| **Key hyperparameters** | Learning rate, batch size | Learning rate, batch size, **plus** clipping threshold `C` and noise multiplier `sigma` |
| **Resistance to membership inference** | Weak -- overfitting-driven confidence gaps remain large | Strong -- formally bounded generalization gap directly shrinks the confidence-gap signal from Section 1 |
| **Resistance to model/gradient inversion** | Weak -- individual gradients can sometimes be used to reconstruct input data (notably dangerous in federated learning) | Strong -- clipped, noised gradients carry much less example-specific information to reconstruct from |

```
    VANILLA SGD                          DP-SGD
    =============                        ========

    per-example gradients                per-example gradients
         |                                     |
         v                                     v
    (no limit)                            CLIP to norm <= C
         |                                     |
         v                                     v
      average                                  sum
         |                                     |
         v                                     v
   update weights                        ADD GAUSSIAN NOISE
                                                |
                                                v
                                              average
                                                |
                                                v
                                          update weights

                                     +-----------------------------+
                                     | Provable epsilon, delta-DP  |
                                     | guarantee attached to output|
                                     +-----------------------------+
```

---

## 9. Why This Protects Against Membership Inference and Model Inversion

Tying directly back to Sections 1 and 2 of this module:

1. **Against membership inference**: recall that the attack relies on the confidence-gap signal -- models are abnormally confident/low-loss on examples they memorized. DP-SGD's clipping and noise **formally bound** how much any single training example can shift the final weights. If no example can uniquely leave a large "fingerprint," the model cannot develop the sharp, example-specific overconfidence that made shadow-model attacks succeed in Section 1. The differential privacy guarantee directly caps the best-possible membership inference attack accuracy as a mathematical function of epsilon -- smaller epsilon provably shrinks the attacker's maximum possible advantage over random guessing.

2. **Against model inversion**: model inversion attacks try to reconstruct actual feature values (e.g., an approximate face image, or specific medical attribute values) of training examples by exploiting how strongly the model's outputs/gradients depend on those exact values. Because DP-SGD's clipped-and-noised gradients carry deliberately obscured, capped information about any individual example, there is fundamentally less example-specific signal available to invert back into the original data -- especially relevant in **federated learning** setups, where raw gradients are sometimes transmitted between devices and a server, creating a direct opportunity for gradient-based reconstruction attacks if left undefended.

---

## 10. Practical Considerations and Limitations

| Consideration | Why It Matters |
|----------------|------------------|
| **Choosing `C` (clipping threshold)** | Too small: clips away useful signal from almost every example, hurting accuracy. Too large: barely limits outliers, weakening the privacy benefit. Usually tuned empirically. |
| **Choosing `sigma` (noise multiplier)** | Directly trades off final epsilon against accuracy -- higher sigma means smaller epsilon (more private) but more training noise (less accurate). |
| **Per-example gradient computation cost** | Standard deep learning frameworks are optimized to compute *batch-averaged* gradients efficiently; computing and clipping *per-example* gradients requires specialized libraries (e.g., Opacus for PyTorch, TensorFlow Privacy) and more memory/compute. |
| **Large batch sizes help** | Averaging over a larger batch dilutes the relative impact of the added noise per example, often improving the privacy-utility tradeoff -- one reason DP-SGD papers often use unusually large batch sizes. |
| **Hyperparameter tuning itself can leak privacy** | If you tune `C`, `sigma`, learning rate, etc. by repeatedly training and checking results on the *same* sensitive dataset, that tuning process itself consumes privacy budget that is easy to forget to account for. |
| **Total epsilon grows with more training steps** | More epochs/steps generally give better accuracy but consume more of the privacy budget (composability) -- another facet of the tradeoff explored more broadly in Section 5. |

---

## 11. Privacy Angle -- Why This Matters

> **Privacy Angle**: DP-SGD is, in practice, the single most widely deployed technique for bringing formal differential privacy guarantees into real deep learning systems -- it is used (in various forms) by organizations including Google and Apple for training models on sensitive user data. As an offensive security professional, DP-SGD matters in two directions:
>
> 1. **Auditing claims**: many organizations claim to use "privacy-preserving AI" or "differential privacy" without disclosing their actual epsilon, clipping threshold, or noise multiplier. A model trained with an enormous epsilon (say, epsilon = 100) using DP-SGD provides only a token, largely meaningless privacy guarantee, even though the vendor can technically claim "we used differential privacy." Knowing the mechanics lets you ask the right diagnostic questions: *what epsilon, over how many steps, with what clipping norm?*
> 2. **Attack surface awareness**: DP-SGD specifically defends the *training* phase against gradient-level leakage. It does **not** automatically protect other parts of an ML pipeline -- e.g., a poorly secured training data store, an API that returns overly detailed outputs, or a model architecture that is separately vulnerable to adversarial examples (Module 1) or data poisoning (Module 6). DP-SGD is one layer of defense, not a silver bullet, and recognizing its precise scope is what separates a superficial understanding from real red-team-level insight.

---

## 12. Key Takeaways

- **DP-SGD modifies vanilla SGD in exactly two places**: per-example gradient clipping, and calibrated Gaussian noise addition to the summed gradients, applied at every training step.

- **Clipping bounds sensitivity**: capping every per-example gradient's norm at a threshold `C` ensures no single training example (however unusual) can disproportionately influence the model's weight updates.

- **Noise obscures individual contribution**: adding Gaussian noise, calibrated to the clipping threshold and desired epsilon, makes it mathematically hard to determine any one example's exact contribution to a given training step.

- **Privacy loss composes across training steps**, which is why a formal "privacy accountant" is used to track the true cumulative epsilon over potentially thousands of steps -- naive summation would overestimate the true (worse) cost.

- **DP-SGD directly weakens membership inference attacks** (Section 1) by formally bounding the confidence-gap signal, and directly weakens **model/gradient inversion attacks** by limiting example-specific information carried in gradients.

- **There is a real accuracy cost**: DP-SGD models typically perform somewhat worse than their non-private counterparts, embodying the privacy-utility tradeoff introduced in Section 2 and explored fully in Section 5.

- **Auditing real-world "DP" claims requires checking the actual epsilon, clipping norm, and noise multiplier used** -- the mere phrase "differential privacy" says nothing about how strong the guarantee actually is.

*Next up: PATE (Private Aggregation of Teacher Ensembles) -- an alternative architecture for training privacy-preserving models, where an ensemble of "teacher" models trained on disjoint data partitions votes (with noise) to train a "student" model that never directly touches the sensitive raw data.*
