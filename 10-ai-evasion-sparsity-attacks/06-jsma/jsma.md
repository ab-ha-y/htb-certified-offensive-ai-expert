# JSMA (Jacobian-based Saliency Map Attack)

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - Sparsity Attacks | Section: JSMA (Jacobian-based Saliency Map Attack)

---

## Table of Contents

1. [The Analogy: A Surgeon, Not a Sledgehammer](#1-the-analogy-a-surgeon-not-a-sledgehammer)
2. [Where JSMA Sits in This Module](#2-where-jsma-sits-in-this-module)
3. [Recap: The Jacobian Matrix](#3-recap-the-jacobian-matrix)
4. [Why Pairs, Not Single Features?](#4-why-pairs-not-single-features)
5. [The JSMA Saliency Pair Score](#5-the-jsma-saliency-pair-score)
6. [A Full Worked Numeric Example](#6-a-full-worked-numeric-example)
7. [The Full JSMA Algorithm, Step by Step](#7-the-full-jsma-algorithm-step-by-step)
8. [Pseudocode](#8-pseudocode)
9. [JSMA Variants: Increase-Only vs. Increase-and-Decrease](#9-jsma-variants-increase-only-vs-increase-and-decrease)
10. [JSMA's Strengths and Weaknesses](#10-jsmas-strengths-and-weaknesses)
11. [JSMA vs. EAD vs. Single-Pixel Attacks](#11-jsma-vs-ead-vs-single-pixel-attacks)
12. [Real-World Walkthrough -- A Full Multi-Iteration JSMA Trace](#12-real-world-walkthrough----a-full-multi-iteration-jsma-trace)
13. [Security Angle](#13-security-angle)
14. [Key Takeaways](#14-key-takeaways)

---

## 1. The Analogy: A Surgeon, Not a Sledgehammer

If Module 9's dense attacks (FGSM, PGD) are like painting an entire canvas with a very thin, even coat of a different color, and EAD (Section 4) is like a careful sculptor using a mathematically optimal chisel, **JSMA** is like a surgeon: it looks at the "patient" (the input), identifies the *single most critical spot* (or, specifically for JSMA, the single most critical *pair* of spots) to operate on, makes the most impactful possible cut there, checks the result, and repeats -- one precise, greedy incision at a time, using x-ray vision (the gradient/Jacobian) to see exactly where to cut before ever touching the scalpel.

JSMA (introduced by Papernot et al. in 2016, "The Limitations of Deep Learning in Adversarial Settings") was one of the very first algorithms to explicitly target the L0 norm for crafting adversarial examples against neural networks, predating both EAD and modern black-box single-pixel search methods (Section 7). It is the direct, concrete embodiment of the saliency-based greedy selection idea introduced in Section 3 of this module, extended with one key refinement: **it considers pairs of features together, not just one at a time.**

```
   DENSE ATTACK               EAD (elastic net,               JSMA (greedy,
   (paint the whole            optimization-based             one/two features
   canvas thinly)               sculpting)                     at a time)

   +---+---+---+---+          +---+---+---+---+           +---+---+---+---+
   | . | . | . | . |          |   |   | 3 |   |           |   |   | X |   |
   +---+---+---+---+          +---+---+---+---+           +---+---+---+---+
   | . | . | . | . |    vs    |   |   |   |   |    vs     |   |   |   | X |
   +---+---+---+---+          +---+---+---+---+           +---+---+---+---+
   Every pixel, tiny           Solved via a convex          Picked ONE pair at
   nudge (Module 9)            optimization objective        a time via direct
                                (FISTA/proximal grad)         saliency ranking,
                                                               no optimization
                                                               solver needed
```

---

## 2. Where JSMA Sits in This Module

Historically, JSMA actually predates EAD and FISTA-based attacks -- it's one of the foundational sparsity-attack papers, and it's what first popularized the idea of using a model's **Jacobian matrix** (introduced in Section 3 of this module) directly as an attack tool. We've covered it after EAD/FISTA in this module's ordering because JSMA is conceptually simpler (no convex optimization theory required -- it's a direct greedy search), but from a learning-the-field perspective, it's worth knowing JSMA is the *older*, more intuitive ancestor, and EAD/FISTA represent a later, more mathematically rigorous evolution of the same underlying goal (small L0 perturbations).

---

## 3. Recap: The Jacobian Matrix

From Section 3 of this module: for a model `f` with `m` output classes and an input `x` with `n` features, the **Jacobian matrix** `J_f(x)` is the `m x n` table of all partial derivatives:

```
J_f(x)[c][i] = d(f_c(x)) / d(x_i)

           feature_1   feature_2   ...   feature_n
class_1      value       value      ...    value
class_2      value       value      ...    value
  ...
class_m      value       value      ...    value
```

Each row is the saliency map for one specific output class (exactly what we computed in Section 3). JSMA's name comes directly from the fact that it uses this **full** matrix -- specifically, the row for the attacker's **target class** and, in some variants, information about the **current predicted class** too -- to decide exactly which features to perturb.

---

## 4. Why Pairs, Not Single Features?

Section 3's simple greedy method perturbed one top-saliency feature at a time. JSMA's specific innovation is to consider **pairs** of features jointly, for a well-motivated reason: real-world constraints (especially for images, where pixel values are bounded, e.g., to `[0, 1]` or `[0, 255]`) mean a single feature's saliency can be "used up" quickly -- once a pixel is pushed all the way to its maximum value, you can't push it any further, even if its saliency score says it's still highly influential.

By pairing two features together, JSMA can:
- Spend one "step" of the algorithm getting the combined benefit of two features at once (potentially reaching the misclassification goal in fewer iterations than perturbing one feature per iteration).
- Partially capture feature *interactions*: two features that individually have moderate saliency might, together, have an outsized combined effect if the model's decision boundary depends on their combination (a scenario individual/per-feature saliency ranking, as in Section 3, cannot detect at all).

The original JSMA paper specifically evaluated on small, structured inputs (28x28 MNIST handwritten digit images, 784 pixels total) where this pairwise consideration was computationally feasible; for larger images the `O(n^2)` cost of checking all pairs (explained in Section 7) becomes a real practical bottleneck, which is part of why later work (EAD, single-pixel search) explored other ways to scale sparsity attacks.

---

## 5. The JSMA Saliency Pair Score

For a target class `t`, define two saliency quantities for each feature `i`:

```
alpha_i = d(f_t(x)) / d(x_i)      <- how much increasing x_i helps the TARGET class
beta_i  = sum over all OTHER classes c != t of  d(f_c(x)) / d(x_i)
                                    <- how much increasing x_i helps everything ELSE
```

JSMA's original "increase-only" formulation (assuming we're allowed to only increase pixel values, e.g. brightening pixels, which was the original paper's simplifying assumption for MNIST digits on a black background) looks for **pairs** of features `(i, j)` that maximize:

```
pair_score(i, j) = alpha_i * alpha_j    if  (alpha_i > 0  and  alpha_j > 0
                                              and  alpha_i * beta_j + alpha_j * beta_i < 0)
                  = -infinity (invalid pair)   otherwise
```

That condition looks intimidating, but the plain-English version is simple: **we want both features to help the target class (positive alpha), and we want the combined effect on all the other classes to be net negative** (i.e., increasing these two features should be pushing confidence AWAY from every competing class, not just toward the target). The pair that satisfies this and has the largest product `alpha_i * alpha_j` is the winner for this iteration -- it represents the single best "double win" available at the current step.

---

## 6. A Full Worked Numeric Example

Let's build a slightly richer toy example than our running 4-feature classifier, because we need at least 2 output classes with distinguishable per-class gradients to show the pair logic meaningfully.

Imagine a tiny 3-feature input (`P, Q, R`, each in `[0, 1]`) and 3 possible output classes (`CAT, DOG, BIRD`), with these (invented, for teaching) gradients at the current input, forming the Jacobian:

```
                feature_P   feature_Q   feature_R
class CAT          0.5         0.6         -0.1
class DOG          0.3        -0.2          0.4
class BIRD        -0.8        -0.4         -0.3
```

Suppose the model currently predicts **DOG**, and our target class is **CAT**.

**Step 1: compute alpha (target = CAT row):**
```
alpha_P = 0.5
alpha_Q = 0.6
alpha_R = -0.1
```

**Step 2: compute beta (sum of the OTHER rows, DOG + BIRD):**
```
beta_P = 0.3 + (-0.8) = -0.5
beta_Q = -0.2 + (-0.4) = -0.6
beta_R = 0.4 + (-0.3) = 0.1
```

**Step 3: check candidate pairs.** We only consider features with positive alpha (helps CAT): that's `P` (0.5) and `Q` (0.6). `R` has negative alpha, so it's excluded from being increased (it would hurt the target class). With only 2 valid candidates, there's exactly one pair to check: `(P, Q)`.

**Step 4: check the validity condition for pair (P, Q):**
```
alpha_P * beta_Q + alpha_Q * beta_P
  = 0.5 * (-0.6) + 0.6 * (-0.5)
  = -0.3 + (-0.3)
  = -0.6   <- negative, so the pair IS valid (satisfies "< 0" condition)
```

**Step 5: compute the pair score:**
```
pair_score(P, Q) = alpha_P * alpha_Q = 0.5 * 0.6 = 0.30
```

Since this is the only valid pair, JSMA selects `(P, Q)` for this iteration and pushes both `P` and `Q` toward their maximum allowed value (e.g., 1.0), then re-checks the model's prediction. If it hasn't yet flipped to CAT, JSMA recomputes the entire Jacobian at the new input values (since P and Q have changed, all the gradients may have shifted -- remember, saliency is a *local* measurement) and repeats the whole process, now searching among the remaining unperturbed feature (`R`) plus any features not yet pushed to their maximum.

---

## 7. The Full JSMA Algorithm, Step by Step

1. **Initialize** `x_adv = x` (start from the original input), and set of "already maximized" features to empty.
2. **Repeat**, until the model predicts the target class OR the L0 budget (max number of features allowed to change) is exhausted:
   a. Compute the full Jacobian `J_f(x_adv)` (one gradient computation per output class, or in efficient implementations, computed together in one backward-pass-friendly way).
   b. Compute `alpha_i` and `beta_i` for every feature `i` not yet maximized.
   c. Search over all valid candidate pairs `(i, j)` (features with the right alpha sign and satisfying the beta condition), and find the pair maximizing `pair_score(i, j)`.
   d. Perturb both `x_i` and `x_j` toward their extreme valid value (e.g., set to 1.0 if increasing, or 0.0 if the "increase-and-decrease" variant calls for decreasing -- see Section 9) by a fixed step amount (or all the way to the extreme, depending on the specific implementation).
   e. Mark `i` and `j` as "maximized" if they've hit their extreme value (so future iterations don't waste time reconsidering them).
   f. Check the model's new prediction on `x_adv`. If it now matches the target class, stop and return success.
3. **If the L0 budget runs out before success**, the attack has failed for this input (some inputs are more robust than others; a real implementation may then relax the budget and retry, or report failure).

```
    START: x_adv = x, maximized_features = {}
      |
      v
    +---------------------------------------------+
    | LOOP:                                        |
    |   1. Compute Jacobian at x_adv                |
    |   2. Compute alpha, beta for all features      |
    |   3. Find best valid pair (i, j)                |
    |   4. Push x_i, x_j toward extreme values          |
    |   5. Mark maximized features                       |
    |   6. model.predict(x_adv) == target? --> STOP, success
    |      else if budget exhausted?      --> STOP, failure
    |      else                            --> repeat loop
    +---------------------------------------------+
```

---

## 8. Pseudocode

```python
def jsma_attack(model, x, target_class, max_features_to_change,
                 perturb_step=1.0):
    """
    Simplified Jacobian-based Saliency Map Attack.
    perturb_step: how far to push a chosen feature per iteration
                  (1.0 means "jump straight to the extreme value")
    """
    x_adv = copy(x)
    maximized = set()

    while len(maximized) < max_features_to_change:
        if model.predict(x_adv) == target_class:
            return x_adv, True   # success

        jacobian = compute_full_jacobian(model, x_adv)  # rows = classes
        alpha = jacobian[target_class]          # target row
        beta = [
            sum(jacobian[c][i] for c in all_classes if c != target_class)
            for i in range(len(x_adv))
        ]

        best_score = -infinity
        best_pair = None

        candidates = [i for i in range(len(x_adv))
                      if i not in maximized and alpha[i] > 0]

        for i in candidates:
            for j in candidates:
                if i == j:
                    continue
                validity = alpha[i] * beta[j] + alpha[j] * beta[i]
                if validity < 0:
                    score = alpha[i] * alpha[j]
                    if score > best_score:
                        best_score = score
                        best_pair = (i, j)

        if best_pair is None:
            break   # no valid pair found -- attack cannot proceed further

        i, j = best_pair
        x_adv[i] = clip(x_adv[i] + perturb_step, 0.0, 1.0)
        x_adv[j] = clip(x_adv[j] + perturb_step, 0.0, 1.0)

        if x_adv[i] in (0.0, 1.0):
            maximized.add(i)
        if x_adv[j] in (0.0, 1.0):
            maximized.add(j)

    return x_adv, model.predict(x_adv) == target_class
```

Note the `O(n^2)` cost hiding in the nested `for i in candidates: for j in candidates:` loop -- this is the practical scalability concern flagged in Section 4. For `n = 784` (a small MNIST image), checking all pairs means up to ~307,000 comparisons per iteration; for a modern high-resolution image with `n` in the hundreds of thousands, this becomes prohibitively slow, which is exactly why later sparse-attack research moved toward EAD's convex-optimization approach (which scales linearly in `n` per iteration) or the query-based single-pixel search methods of Section 7.

---

## 9. JSMA Variants: Increase-Only vs. Increase-and-Decrease

The original JSMA paper's simplifying assumption (only ever *increasing* pixel values) worked reasonably well for MNIST digits, which are drawn as bright strokes on a dark background -- increasing pixel brightness in the right spots is a natural way to "add strokes" that shift the digit's appearance toward another digit class.

For more general inputs, a more flexible variant allows each feature to be pushed toward *either* extreme (increase toward max, or decrease toward min), based on the actual sign of its saliency (positive alpha and appropriate beta condition -> increase; negative alpha and appropriate beta condition -> decrease). This generalization is necessary for domains where "increasing everything" isn't a meaningful attack strategy at all -- e.g., malware feature vectors where some features benefit from being turned "on" and others from being turned "off."

| Variant | Direction Allowed | Best Suited For | Downside |
|---|---|---|---|
| **Increase-only (original)** | Push toward max value only | MNIST-style bright-on-dark images | Fails for inputs where decreasing a feature would be more effective |
| **Increase-and-decrease** | Push toward either extreme, based on sign | General images, general feature vectors | Roughly doubles the candidate search space per iteration |

---

## 10. JSMA's Strengths and Weaknesses

**Strengths:**
- Conceptually simple and directly interpretable: at every step, you can literally point to "these two features, because of this saliency reasoning."
- Historically important: one of the first algorithms to demonstrate that neural networks can be fooled by changing a *very* small number of input features (in the original paper's MNIST experiments, often under 4% of pixels).
- Naturally produces extremely sparse perturbations, since it explicitly targets an L0-style budget from the start, unlike L1/elastic-net relaxations which only approximately encourage sparsity.

**Weaknesses:**
- `O(n^2)` per-iteration cost from checking all feature pairs makes it computationally expensive for large inputs (high-resolution images, long feature vectors).
- Purely greedy: it never reconsiders earlier choices, so it can get stuck making locally good but globally suboptimal choices (a feature pair perturbed early might turn out, several iterations later, not to have been the ideal choice, but JSMA has no mechanism to "undo" it).
- Like all gradient/Jacobian-based methods, it's vulnerable to gradient masking defenses (Section 3's limitations discussion applies here directly).
- The original "pushed to the extreme value" step can sometimes make perturbations more perceptible than necessary (an optimization-based method like EAD might find a smaller, still-effective change), even though the *number* of changed features stays low.

---

## 11. JSMA vs. EAD vs. Single-Pixel Attacks

| Attack | Selection Mechanism | Per-Iteration Cost | Typical Sparsity | Optimization-Based? |
|---|---|---|---|---|
| **JSMA** | Greedy pairwise saliency ranking | `O(n^2)` (pair search) | Very sparse, but perturbation magnitude per feature is often extreme (pushed to bounds) | No -- pure greedy heuristic |
| **EAD** | Convex elastic-net objective, solved via FISTA | `O(n)` per iteration (one gradient computation) | Sparse, with damped, non-extreme magnitudes | Yes -- formal convex optimization |
| **Single-pixel (Section 7)** | Black-box search (e.g., differential evolution), no gradients needed at all | Depends on search budget (many black-box queries) | Extreme sparsity (as low as k=1) | No -- gradient-free heuristic search |

JSMA sits conceptually between EAD (fully optimization-driven) and single-pixel attacks (fully gradient-free search): it uses gradient information (like EAD) but applies it via a greedy heuristic rather than a formal convex solver (like single-pixel search).

---

## 12. Real-World Walkthrough -- A Full Multi-Iteration JSMA Trace

Section 6 walked through a single iteration of JSMA in detail. Let's now run it for a full *second* iteration on the same example, to show how the "maximized features" bookkeeping and the recomputed Jacobian change the outcome across iterations -- the part that a one-shot example can't demonstrate.

**Recap of where we left off (end of Section 6):** starting classes `CAT/DOG/BIRD`, features `P, Q, R`, target class `CAT`, current prediction `DOG`. We selected the pair `(P, Q)` (the only valid pair, score 0.30) and pushed both to their maximum value of 1.0. `R` was excluded (negative alpha) and left untouched.

**After iteration 1:** suppose the model, now evaluated at the updated input `(P=1.0, Q=1.0, R=r_original)`, still predicts DOG (not yet flipped -- this is common; one pairwise step is often not enough). Both `P` and `Q` are marked as "maximized" (they're at their upper bound of 1.0), so the algorithm will exclude them from consideration in future iterations, per the pseudocode's `if x_adv[i] in (0.0, 1.0): maximized.add(i)` logic.

**Iteration 2: recompute the Jacobian at the new point.** Because the input has changed substantially (P and Q both jumped from their original values to 1.0), the model's local gradients at this new point are generally different from the original Jacobian used in iteration 1 -- this is the direct consequence of gradients being a *local* measurement, the same lesson demonstrated numerically in Section 11 of the Saliency section. Suppose the new Jacobian, evaluated at `(P=1.0, Q=1.0, R=r_original)`, is:

```
                feature_P   feature_Q   feature_R
class CAT          0.1         0.05        0.7      <- R's saliency for CAT has grown!
class DOG           0.2        0.1        -0.3
class BIRD         -0.9        -0.5        -0.2
```

Notice `R`'s saliency for the target class CAT has jumped from `-0.1` (iteration 1) to `+0.7` (iteration 2) -- entirely plausible in a nonlinear model, where the sensitivity to one feature can depend heavily on the current values of other features (this is exactly the "feature interaction" phenomenon flagged as a general saliency limitation back in Section 10 of the previous section, and it's precisely why JSMA insists on recomputing the full Jacobian at every iteration rather than using a stale saliency ranking).

**Step-by-step iteration 2:**

```
Candidates (not yet maximized, alpha > 0): only R (since P, Q are excluded
  as "maximized," even though their alpha values might still look favorable)

alpha_R = 0.7
beta_R = 0.2 (DOG's row) + (-0.9)... wait, beta sums over OTHER classes for R:
       = jacobian[DOG][R] + jacobian[BIRD][R] = -0.3 + -0.2 = -0.5

With only ONE candidate feature remaining (R), there is no valid PAIR to form
(JSMA's pairwise logic requires two distinct candidates). In this situation, a
real implementation falls back to a single-feature saliency step for R alone
(a documented practical fallback when the candidate pool shrinks to size 1).

Since alpha_R > 0 and beta_R < 0 (R helps the target and hurts the competitors,
satisfying the same underlying spirit as the pairwise validity condition),
R is pushed to its maximum value: R = 1.0 (assuming valid range [0,1]).
```

**Checking the model after iteration 2:** new input `(P=1.0, Q=1.0, R=1.0)` -- all three features now maxed out. Suppose the model now predicts **CAT**. Success, after 2 iterations, having touched all 3 available features (L0 = 3 out of 3, the maximum possible for this tiny toy example -- in a real image with hundreds or thousands of pixels, this would represent an extremely small fraction of the total).

**What this two-iteration trace demonstrates that the single-iteration example in Section 6 could not:**

1. **The "maximized" bookkeeping matters.** Without excluding `P` and `Q` from iteration 2's candidate pool, the algorithm might waste computation re-evaluating pairs that can no longer be usefully changed (they're already at their bound).
2. **Saliency genuinely shifts between iterations** -- R's contribution to the target class changed sign-relevant magnitude entirely because of the changes made to P and Q in the previous step, directly illustrating why JSMA's loop structure (recompute, don't just rank once) is not just a defensive-sounding caveat but an observable, load-bearing part of correctness.
3. **The pairwise mechanism degrades gracefully to single-feature saliency** when the candidate pool shrinks below 2, which is worth knowing when reading or implementing real JSMA code -- the "clean" pairwise formula in Section 5 is not always literally applicable at every step of every run.

## 13. Security Angle

- **Minimal, surgical evasion demonstrations.** JSMA's directly interpretable "these exact 2 features, at this exact step" trace is excellent for producing clear, explainable proof-of-concept reports for clients: you can show precisely which features were changed and why (citing the saliency scores), which is often more convincing in a security engagement write-up than an opaque optimization result.

- **Historical basis for modern sparse-attack tooling.** JSMA is implemented in most major adversarial ML toolkits (e.g., IBM's Adversarial Robustness Toolbox, CleverHans) and is frequently used as a standard baseline for evaluating a model's robustness to sparse/L0-style perturbations. Understanding it is necessary to correctly interpret robustness benchmarks that reference it.

- **Small structured inputs are especially vulnerable.** JSMA's `O(n^2)` cost makes it most practical against smaller, structured feature vectors -- which describes many real-world security-relevant classifiers well: tabular malware feature vectors, network flow feature vectors, and small fixed-size sensor inputs are often far smaller than a megapixel image, making JSMA (or its pairwise logic) directly applicable, not just a toy demonstration limited to MNIST.

- **Feature-interaction insight for defenders.** JSMA's pairwise consideration is itself informative for defenders: if a JSMA-style search consistently finds that certain feature *pairs* are highly exploitable together (even when neither feature alone has high individual saliency), that's a signal the model has learned a fragile decision boundary depending on specific feature co-occurrence -- useful information for hardening feature engineering or adding interaction-aware regularization during training.

---

## 14. Key Takeaways

- **JSMA** greedily and iteratively selects the *pair* of features with the best combined saliency for the attacker's target class, pushes both toward their extreme valid values, and repeats until the model misclassifies or the L0 budget runs out.
- It directly uses the **Jacobian matrix** (Section 3): `alpha` measures how much a feature helps the target class, `beta` measures how much it helps everything else; the best pair maximizes `alpha_i * alpha_j` subject to a validity condition ensuring the pair doesn't inadvertently help a competing class.
- Pairs (not single features) are used because real feature bounds limit how much benefit a single feature's saliency can deliver, and pairs partially capture feature *interactions* that single-feature saliency ranking misses.
- JSMA's Achilles' heel is its **O(n^2) per-iteration cost** from exhaustively checking feature pairs, which limits practical scalability to smaller inputs -- a key motivation for later, more scalable approaches like EAD.
- JSMA is a **pure greedy heuristic** (no formal optimization guarantees), in contrast to EAD's convex, FISTA-solved objective -- it's historically important and highly interpretable, but not provably optimal.
- Security-wise, JSMA's step-by-step interpretability makes it excellent for clear proof-of-concept evasion reporting, and its pairwise logic can reveal exploitable feature *interactions* useful for both attack and defense.

---

*Next up: Single-Pixel and Pairwise Variants -- the extreme case of flipping a classifier's prediction by changing just one (or two) pixels, why high-dimensional models are so fragile to this, and what black-box, gradient-free search methods make it possible without ever touching the model's gradients.*
