# Saliency-Based Feature Selection

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - Sparsity Attacks | Section: Saliency-Based Feature Selection

---

## Table of Contents

1. [The Analogy: Which Wire Do You Cut?](#1-the-analogy-which-wire-do-you-cut)
2. [What Is a Gradient? (No Calculus Background Assumed)](#2-what-is-a-gradient-no-calculus-background-assumed)
3. [From Gradients to Saliency](#3-from-gradients-to-saliency)
4. [A Tiny Worked Numeric Example](#4-a-tiny-worked-numeric-example)
5. [The General Saliency Formula](#5-the-general-saliency-formula)
6. [Building a Saliency Map](#6-building-a-saliency-map)
7. [Greedy Feature Selection Using Saliency](#7-greedy-feature-selection-using-saliency)
8. [Pseudocode: Rank-and-Perturb Loop](#8-pseudocode-rank-and-perturb-loop)
9. [Why This Beats Random or Exhaustive Search](#9-why-this-beats-random-or-exhaustive-search)
10. [Limitations of Simple Saliency](#10-limitations-of-simple-saliency)
11. [Real-World Walkthrough -- Ranking Features on a Network Intrusion Detector](#11-real-world-walkthrough----ranking-features-on-a-network-intrusion-detector)
12. [Security Angle](#12-security-angle)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. The Analogy: Which Wire Do You Cut?

Imagine a control panel with 1,000 wires, and you need to disable a machine by cutting exactly one wire (or as few as possible). You don't have time to cut each wire one at a time and check if the machine stopped -- that's the "brute force" search from Section 1 of this module, and it doesn't scale.

Instead, imagine you had a special sensor that could touch each wire (without cutting it) and tell you, "if you tugged this wire by 1mm, the machine's behavior would change by *this much*." You could scan all 1,000 wires with this sensor in one pass, rank them by "how much tugging this wire matters," and cut only the top-ranked wire(s).

**That sensor is exactly what a gradient gives you in machine learning.** A **saliency map** is the result of running that sensor over every feature (pixel, byte, API-call flag) and recording how much each one matters to the model's decision. "Saliency," in plain English, just means "noticeable" or "standing out" -- a saliency map highlights which features stand out as most influential to the prediction.

This section is the practical answer to the problem posed at the end of Section 1: instead of searching over `C(n, k)` combinatorial subsets, use one gradient computation to *rank* all `n` features by importance, then just take the top `k`. One pass, not trillions.

```
BRUTE FORCE (Section 1):                    SALIENCY-BASED (this section):

Try {1,2,3} -> optimize -> check            Compute gradient ONCE (fast)
Try {1,2,4} -> optimize -> check                     |
Try {1,2,5} -> optimize -> check                     v
   ... (trillions of tries) ...             Rank ALL features by "impact"
                                                      |
                                                      v
                                             Perturb only the TOP-ranked
                                             feature(s). Done.
```

---

## 2. What Is a Gradient? (No Calculus Background Assumed)

If you've never seen a gradient before, here is the plain-English version, built from the ground up.

**Step 1: the derivative (1 input, 1 output).** Imagine you're standing on a hillside, and your height depends only on your east-west position. The **derivative** at your current spot answers: "if I take one small step east, how much does my height change?" A positive derivative means stepping east takes you *up*; a negative derivative means stepping east takes you *down*; a derivative near zero means you're on flat ground (or at a peak/valley).

**Numeric example.** Suppose your height (in meters) as a function of your east position `x` (in meters) is `height(x) = x^2`. At `x = 3`, the derivative (computed with the standard calculus rule "derivative of x^2 is 2x") is `2 * 3 = 6`. This means: near x=3, moving 1 meter east increases your height by roughly 6 meters. Let's sanity check: `height(3) = 9`, `height(4) = 16`, difference = 7 (close to 6 -- it's not exact because the hill is curving, but it's the right ballpark, and the approximation gets better the smaller the step).

**Step 2: the gradient (many inputs, 1 output).** Now imagine your height depends on BOTH your east-west position (`x`) AND your north-south position (`y`): `height(x, y)`. The **gradient** is simply the *list* of derivatives, one per input direction: "if I nudge x, how much does height change?" and separately "if I nudge y, how much does height change?" We write it as a vector: `gradient = [d(height)/dx, d(height)/dy]`.

```
    Gradient = "which direction should I step to change the
                output the FASTEST, and by roughly how much
                per direction?"

         North (y)
            ^
            |     gradient vector points in the
            |     direction of STEEPEST INCREASE
            |    /
            |   /
            |  /
            | /
    --------o------------------> East (x)
           you are here

    Each ENTRY of the gradient vector tells you the
    sensitivity to nudging JUST that one input.
```

**Step 3: for a neural network.** A neural network is just a (very complicated) mathematical function that takes an input vector (pixel values, byte features, etc.) and produces an output (e.g., a confidence score for "this is a cat"). The gradient of that output with respect to the *input* tells you: "if I nudge input feature i by a tiny amount, how much does the cat-confidence score change?" Modern deep learning frameworks compute this gradient automatically via an algorithm called **backpropagation** -- you don't need to derive it by hand, but you should understand what it represents: a list of per-feature sensitivities.

---

## 3. From Gradients to Saliency

A **saliency map** (in the context of adversarial ML) is essentially the gradient of the model's output with respect to the input, often reshaped back into the input's original shape (e.g., back into a 2D grid if the input was an image) so a human can visualize which regions matter most.

There's an important subtlety worth flagging early, because it becomes central once we reach JSMA (Section 6): a saliency value can be computed with respect to the score of the model's **current predicted class**, or with respect to the score of the **attacker's target class**, and these give different (often opposite-signed) information:

- Gradient w.r.t. current class score: "if I increase this feature, does the model become MORE or LESS confident in the CURRENT (correct) label?"
- Gradient w.r.t. target class score: "if I increase this feature, does the model become MORE or LESS confident in the ATTACKER'S DESIRED (wrong) label?"

An attacker generally wants features that simultaneously **increase** the target-class score and **decrease** the current-class score -- the biggest "double win" per feature touched. JSMA (Section 6) formalizes exactly this combined saliency score.

---

## 4. A Tiny Worked Numeric Example

Let's reuse the toy 4-feature classifier from Section 1 (`A, B, C, D`, each in `[0, 1]`), with the same linear decision rule:

```
score(x) = 2*A + 0.1*B + 0.1*C + 3*D
predict "CAT" if score > 2.5, else "DOG"
```

Because this scoring function is linear, its gradient with respect to each input is simply the **coefficient** of that input -- this is the simplest possible case, and it makes the "sensitivity" intuition completely transparent:

```
d(score)/dA = 2.0
d(score)/dB = 0.1
d(score)/dC = 0.1
d(score)/dD = 3.0
```

The **saliency map** is just this list: `[2.0, 0.1, 0.1, 3.0]`. Ranking by absolute value (magnitude of impact, regardless of direction): `D (3.0) > A (2.0) > B (0.1) ≈ C (0.1)`.

This instantly tells us feature `D` gives the most "bang per unit of perturbation," which is exactly the reasoning used to justify the choice in Section 1's worked example -- we now have the formal machinery (gradients) behind that earlier intuitive choice.

For a real neural network the scoring function is nonlinear (it's a stack of many layers with nonlinear activation functions like ReLU or sigmoid in between), so the gradient is not a fixed set of coefficients -- it changes depending on the current input values. But the interpretation stays identical: **the gradient at the current point tells you the local sensitivity of the output to each input feature.**

---

## 5. The General Saliency Formula

For a model `f` that outputs a score `f_c(x)` for class `c` given input `x` (a vector of `n` features `x_1, ..., x_n`), the **saliency of feature i with respect to class c** is:

```
S_c(x, i) = d(f_c(x)) / d(x_i)
```

In plain English: "how much does the model's confidence in class c change if I nudge only feature `x_i`, holding all other features fixed?"

Stacking this across all `i` from 1 to `n` gives the **saliency map** (a vector, same length as the input):

```
saliency_map(x, c) = [ S_c(x, 1), S_c(x, 2), ..., S_c(x, n) ]
```

The full object of all `d(f_c(x))/d(x_i)` values for every output class `c` and every input feature `i` simultaneously is called the **Jacobian matrix** of the model at point `x` -- it's literally a table with one row per output class and one column per input feature, where each cell is a partial derivative. This is exactly where "Jacobian-based Saliency Map Attack" (JSMA, Section 6) gets its name: JSMA uses this full Jacobian matrix, not just the saliency w.r.t. one class, to decide which pairs of features to perturb.

```
                    JACOBIAN MATRIX (rows = classes, cols = features)

               feature_A   feature_B   feature_C   feature_D
  class "CAT"     2.0         0.1         0.1         3.0
  class "DOG"    -2.0        -0.1        -0.1        -3.0
  class "BIRD"    0.3         0.4        -0.2         0.1
     ...

  Row for the TARGET class -> tells attacker which features to
                               INCREASE to boost target confidence
  Row for the CURRENT class -> tells attacker which features to
                               DECREASE to reduce current confidence
```

---

## 6. Building a Saliency Map

Here's a minimal pseudocode sketch of computing a saliency map for a given target class, using automatic differentiation (the thing that makes backpropagation possible; you call one function and the framework hands you the full gradient vector).

```python
def compute_saliency_map(model, x, target_class):
    """
    Returns a vector the same length as x, where entry i tells us
    how much increasing x[i] would increase the model's confidence
    in target_class.
    """
    # In a real framework (PyTorch/TensorFlow) this line triggers
    # automatic differentiation / backpropagation under the hood.
    gradient = autodiff_gradient(
        output=model.class_score(x, target_class),
        wrt=x
    )
    return gradient   # one entry per input feature

# Worked example values from Section 4:
# gradient = [2.0, 0.1, 0.1, 3.0]  for target_class = "CAT"
```

To visualize a saliency map for an actual image, you'd reshape this flat vector back into the image's height x width grid and often display it as a heatmap (bright = high saliency = high impact, dark = low saliency = low impact):

```
Original image (a "7" digit)      Saliency map heatmap
+--------------------+            +--------------------+
|                    |            |    ..              |
|      _____         |            |   .##.             |
|     |     |        |            |   .##.             |
|         /          |            |     ##             |
|        /           |            |    ##.             |
|       /            |            |   ##.              |
|      /             |            |  ##.               |
|                    |            |                    |
+--------------------+            +--------------------+
                                    (bright/# = pixels that
                                     most affect the "7"
                                     vs "1" decision boundary,
                                     likely along the diagonal
                                     stroke and the top bar)
```

---

## 7. Greedy Feature Selection Using Saliency

Once we have a saliency map, a **greedy** selection strategy (greedy = at each step, take the locally-best option without reconsidering past choices) is straightforward:

1. Compute the saliency map for the current input and target class.
2. Sort features by saliency magnitude, highest first.
3. Take the top `k` features (your L0 budget) and perturb them toward the target class direction (increase if saliency is positive, decrease if negative -- or in a simplified attack, just push toward the boundary of the valid range: e.g., 0 or 1 for pixels).
4. Re-check whether the model's prediction has flipped. If not, and there's budget left, optionally re-compute the saliency map on the *updated* input (since after changing some features, the sensitivities of the *remaining* features may have shifted -- remember, saliency is a *local* measurement, valid only "at the current point") and repeat.

This iterative "recompute-then-select" loop, applied one (or two) feature(s) per iteration, is precisely the structure of JSMA (Section 6). Saliency-based selection is the ranking mechanism; JSMA is the full attack algorithm built around it.

---

## 8. Pseudocode: Rank-and-Perturb Loop

```python
def saliency_greedy_attack(model, x, target_class, k, max_iters):
    """
    Greedily perturb the top-k most salient features toward the
    target class, re-ranking after each change.
    """
    x_adv = copy(x)
    changed_features = set()

    for iteration in range(max_iters):
        if model.predict(x_adv) == target_class:
            return x_adv, changed_features   # success!

        if len(changed_features) >= k:
            break   # L0 budget exhausted

        saliency = compute_saliency_map(model, x_adv, target_class)

        # Don't re-touch features we've already maxed out
        for i in changed_features:
            saliency[i] = 0

        # Pick the single most influential remaining feature
        best_feature = argmax(abs(saliency))

        # Push it toward the extreme in the direction that helps
        if saliency[best_feature] > 0:
            x_adv[best_feature] = MAX_VALID_VALUE   # e.g. 1.0 or 255
        else:
            x_adv[best_feature] = MIN_VALID_VALUE   # e.g. 0.0 or 0

        changed_features.add(best_feature)

    return x_adv, changed_features   # may or may not have succeeded
```

**Worked trace, reusing our toy classifier** (`A,B,C,D`; boundary at `score > 2.5`; k=1 budget; original `A=0.4,B=0.9,C=0.9,D=0.2`, currently DOG, target CAT):

```
Iteration 1:
  Predict(x_adv) = DOG (score 1.58) -- not yet CAT
  changed_features = {} -- budget not exhausted
  saliency = [2.0, 0.1, 0.1, 3.0]  (all positive, since increasing
                                     any of A,B,C,D increases "CAT" score)
  best_feature = D (largest |saliency| = 3.0)
  Push D to MAX_VALID_VALUE = 1.0
  changed_features = {D}

Iteration 2:
  Predict(x_adv) = CAT (score 3.98, computed in Section 1) -- SUCCESS
  Return x_adv, {D}
```

One feature changed, attack succeeds, exactly matching the manual reasoning from Section 1 -- but now derived systematically from the saliency map instead of by inspection.

---

## 9. Why This Beats Random or Exhaustive Search

| Strategy | Features Checked per Iteration | Uses Model Structure? | Scales to Large Inputs? |
|---|---|---|---|
| **Exhaustive subset search (Section 1)** | `C(n, k)` combinations | No (blind search) | No -- combinatorial explosion |
| **Random feature selection** | 1 (but no guarantee it's a good one) | No | Yes, but often needs far more of the L0 budget to succeed |
| **Saliency-based greedy selection** | 1 gradient computation ranks all `n` at once | Yes -- directly uses the model's own sensitivity information | Yes -- cost is roughly one backward pass per iteration, same as ordinary training |

The key efficiency win: **one gradient computation via backpropagation gives information about all `n` features simultaneously**, because backpropagation is specifically designed to compute an entire gradient vector in roughly the same cost as one forward pass through the network (not `n` separate forward passes). This is why saliency-based methods scale to realistic image sizes (thousands to millions of pixels) where brute-force L0 search (Section 1) simply cannot.

---

## 10. Limitations of Simple Saliency

It's important to be honest about where this technique falls short, both for accuracy in your understanding and because these limitations directly motivate later refinements:

- **Locality.** The gradient only tells you the sensitivity *at the current point*. For a highly nonlinear model, a feature that looks unimportant right now might become very important after other features have changed. This is why greedy methods re-compute the saliency map after each change (Section 7), rather than ranking once and perturbing blindly.

- **Interacting features.** Saliency treats each feature's impact independently (one partial derivative per feature), but real model decisions often depend on *combinations* of features. JSMA's specific innovation (Section 6) is to explicitly consider **pairs** of features together, partially addressing this.

- **Saturation and vanishing gradients.** If a feature is already pushed to an extreme value (e.g., a pixel at pure black or pure white, past which a ReLU or sigmoid activation "flattens out"), its local gradient can shrink toward zero even though that feature might still matter for the overall decision. This is a well-known general challenge with gradient-based methods in deep learning, not unique to attacks.

- **Gradient masking as a defense.** Because saliency-based attacks fundamentally rely on gradient information, defenders can deploy defenses that deliberately obscure or randomize gradients (e.g., adding noise layers, non-differentiable preprocessing) to make saliency maps unreliable -- forcing attackers toward gradient-free (black-box) alternatives.

---

## 11. Real-World Walkthrough -- Ranking Features on a Network Intrusion Detector

Let's do a full, concrete walkthrough of the greedy rank-and-perturb loop (Section 8) against a slightly richer, multi-feature model, so you can see the iterative "recompute saliency after each change" behavior actually matter (unlike the single-step toy example in Section 4, where one feature was enough).

**Setup.** A network intrusion detection system (NIDS) scores outbound connections using 6 features extracted from a flow record, flagging "ATTACK" if the score exceeds 5.0. This is a nonlinear model (unlike our earlier linear toy classifiers), so the gradient is NOT a fixed set of coefficients -- it changes depending on the current feature values. For teaching purposes, imagine the model behaves like this simplified nonlinear scoring function:

```
score(x) = 2*sqrt(x1) + 1.5*x2 + 0.8*x3^2 + x4 + 0.3*x5 + 0.1*x6

x1 = bytes_sent_thousands      (currently 9,   i.e. 9000 bytes)
x2 = unique_ports_contacted     (currently 3)
x3 = connection_duration_mins    (currently 1.5)
x4 = packets_per_second           (currently 2)
x5 = payload_entropy                (currently 1)
x6 = time_of_day_score               (currently 1)
```

Current score: `2*sqrt(9) + 1.5*3 + 0.8*(1.5^2) + 2 + 0.3*1 + 0.1*1 = 2*3 + 4.5 + 0.8*2.25 + 2 + 0.3 + 0.1 = 6 + 4.5 + 1.8 + 2 + 0.3 + 0.1 = 14.7`. Flagged **ATTACK** (14.7 > 5.0).

**Computing the gradient at the current point** (using calculus rules for each term: derivative of `2*sqrt(x1)` is `1/sqrt(x1)`, derivative of `0.8*x3^2` is `1.6*x3`, everything else is linear so its derivative is just its coefficient):

```
d(score)/dx1 = 1/sqrt(9) = 0.333
d(score)/dx2 = 1.5
d(score)/dx3 = 1.6 * 1.5 = 2.4
d(score)/dx4 = 1.0
d(score)/dx5 = 0.3
d(score)/dx6 = 0.1

saliency_map = [0.333, 1.5, 2.4, 1.0, 0.3, 0.1]
```

**Iteration 1.** Ranking by magnitude: `x3 (2.4) > x2 (1.5) > x4 (1.0) > x1 (0.333) > x5 (0.3) > x6 (0.1)`. Feature `x3` (connection duration) has the highest saliency. Suppose the valid range for x3 is `[0, 5]` minutes and we push it toward its minimum (since decreasing it reduces the score, given the positive gradient sign means "increasing x3 increases score," so decreasing helps the attacker): set `x3 = 0`.

New score: `2*sqrt(9) + 1.5*3 + 0.8*(0^2) + 2 + 0.3 + 0.1 = 6 + 4.5 + 0 + 2 + 0.3 + 0.1 = 12.9`. Still ATTACK (12.9 > 5.0). Not done yet.

**Iteration 2 -- re-computing the saliency map at the NEW point** (x3 is now 0, everything else unchanged): the derivative of `0.8*x3^2` at `x3=0` is `1.6*0 = 0`. This is the critical "locality" lesson from Section 10's limitations discussion made concrete: **x3's saliency has now collapsed to zero**, exactly because we pushed it to a boundary point where its local sensitivity vanishes (this is the "saturation" phenomenon).

```
new saliency_map = [0.333, 1.5, 0.0, 1.0, 0.3, 0.1]
```

Now `x2` (unique ports contacted, saliency 1.5) is the new top-ranked feature. Push it toward its minimum, say `x2 = 0` (contact zero distinct ports -- consolidate traffic to one port).

New score: `2*sqrt(9) + 1.5*0 + 0 + 2 + 0.3 + 0.1 = 6 + 0 + 0 + 2 + 0.3 + 0.1 = 8.4`. Still ATTACK. Continue.

**Iteration 3 -- recompute again.** x2's saliency contribution is now moot (it's a linear term, so its gradient stays 1.5 regardless of value, but we've already used up this feature -- mark it "maximized" per the pseudocode in Section 8). Remaining candidates: `x1 (0.333), x4 (1.0), x5 (0.3), x6 (0.1)`. Top is `x4` (packets_per_second, saliency 1.0). Push toward minimum: `x4 = 0`.

New score: `6 + 0 + 0 + 0 + 0.3 + 0.1 = 6.4`. Still ATTACK, but close.

**Iteration 4.** Remaining candidates: `x1 (0.333), x5 (0.3), x6 (0.1)`. Top is `x1` (bytes sent, saliency 0.333). This one is harder to push to a hard minimum realistically (an attacker still needs to exfiltrate *some* data), but suppose we reduce it from 9 (9000 bytes) down to 1 (1000 bytes, spread across more connections instead -- a realistic evasion tactic): 

New score: `2*sqrt(1) + 0 + 0 + 0 + 0.3 + 0.1 = 2 + 0.4 = 2.4`. Below 5.0 -- **ATTACK avoided, now classified BENIGN.**

**Summary of the trace:**

```
Iteration:  1        2        3        4
Feature:    x3       x2       x4       x1
Score:      14.7 --> 12.9 --> 8.4 --> 6.4 --> 2.4
                (still ATTACK the whole way, until iteration 4)
Features touched (L0): 4 out of 6
```

This walkthrough demonstrates two lessons that the single-step toy example in Section 4 was too simple to show: **(1)** saliency ranking genuinely changes between iterations as features saturate (x3's contribution vanished entirely after being maxed out), and **(2)** a greedy method can require touching several features in sequence even when it starts from the single highest-ranked one -- exactly the iterative loop structure formalized in the pseudocode of Section 8, and the direct conceptual ancestor of JSMA's pairwise version (Section 6).

## 12. Security Angle

- **Single-pixel/few-feature evasion.** Saliency ranking is the core building block behind demonstrations that a handful of pixel changes -- sometimes just one -- can flip an image classifier's prediction (explored fully in Section 7 of this module). Knowing *which* pixel to target is what makes those attacks practical instead of a random lottery.

- **Malware/spam feature-cost targeting.** In a malware classifier with hundreds of binary features ("has this API call," "contains this string," "packed with tool X"), a saliency map tells an attacker precisely which single feature flip gives the biggest swing toward "benign" classification -- letting them make the *cheapest* possible modification (e.g., adding one harmless-looking API call reference) rather than restructuring the entire file.

- **Explainability tools double as attack tools.** Saliency maps were originally popularized as an **interpretability** technique -- helping defenders understand *why* a model made a decision. The exact same computation, pointed at an adversarial objective instead of an explanatory one, becomes an attack primitive. This dual-use nature is a recurring theme in offensive AI: tools built for trust and transparency often hand attackers a precise map of where a model is weakest.

- **Query-efficient attacks against black-box APIs.** When an attacker doesn't have direct access to gradients (e.g., a remote ML API), techniques exist to *estimate* saliency-like sensitivity information using only input/output queries (e.g., finite-difference approximations: nudge one feature slightly, observe the score change, repeat). This is slower and noisier than true gradients but follows the same underlying "rank by sensitivity" logic.

---

## 13. Key Takeaways

- A **gradient** measures local sensitivity: how much a function's output changes per tiny nudge to one input. For a neural network, backpropagation computes this efficiently for every input feature at once.
- A **saliency map** is the gradient of a target class's score with respect to each input feature, reshaped for interpretation -- it ranks features by how much each one matters to the model's decision.
- The full table of gradients (all output classes x all input features) is called the **Jacobian matrix** -- the namesake of JSMA (Section 6).
- **Greedy saliency-based selection** replaces brute-force subset search: rank all features in one pass, perturb the top-ranked one(s), re-check, and (optionally) re-rank -- turning an NP-hard search into a fast, iterative loop.
- Saliency is a **local** measurement (valid at the current input), which is why iterative attacks re-compute it after each change rather than ranking once upfront.
- Limitations -- locality, ignoring feature interactions, gradient saturation, and vulnerability to gradient-masking defenses -- motivate JSMA's pairwise refinement and the black-box query-based alternatives.
- Security-wise, saliency maps let attackers find the cheapest, stealthiest possible feature changes, and the same computation used for model interpretability (explaining a decision) doubles as an attack primitive (breaking that decision).

---

*Next up: ElasticNet Attack (EAD) -- combining the L1 sparsity penalty from Section 2 with an L2 penalty in a single optimization objective, giving attackers a tunable balance between sparse perturbations and overall attack reliability.*
