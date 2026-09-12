# ElasticNet Attack (EAD)

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - Sparsity Attacks | Section: ElasticNet Attack (EAD)

---

## Table of Contents

1. [The Analogy: Two Different Kinds of "Good"](#1-the-analogy-two-different-kinds-of-good)
2. [Recap: What L1 and L2 Each Do Alone](#2-recap-what-l1-and-l2-each-do-alone)
3. [Elastic Net: Combining Both](#3-elastic-net-combining-both)
4. [The EAD Attack Objective](#4-the-ead-attack-objective)
5. [A Tiny Worked Numeric Example](#5-a-tiny-worked-numeric-example)
6. [How EAD Chooses Between L1 and L2 Weight](#6-how-ead-chooses-between-l1-and-l2-weight)
7. [The Full EAD Algorithm, Step by Step](#7-the-full-ead-algorithm-step-by-step)
8. [Pseudocode](#8-pseudocode)
9. [EAD vs. Carlini-Wagner (C&W) vs. JSMA](#9-ead-vs-carlini-wagner-cw-vs-jsma)
10. [Real-World Walkthrough -- Crafting a Sparse Adversarial Digit](#10-real-world-walkthrough----crafting-a-sparse-adversarial-digit)
11. [Security Angle](#11-security-angle)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. The Analogy: Two Different Kinds of "Good"

Imagine you're editing a photo to remove a watermark, and you have two competing goals:

- **Goal 1 (sparsity):** touch as few pixels as possible, so the edit is hard to spot in a pixel-diff comparison.
- **Goal 2 (smoothness):** whatever pixels you do touch, don't change them by a huge, jarring amount -- keep the overall visual disruption low.

L1 alone (Section 2 of this module) chases Goal 1 aggressively -- it happily drives most pixels to exactly zero change, but it doesn't care much how large the *remaining* nonzero changes get, as long as the total sum of absolute values is kept in check. L2 alone (the workhorse of Module 9's dense attacks) chases Goal 2 -- it spreads change smoothly and keeps the overall magnitude small, but essentially never produces exact zeros (recall from Section 2: L2 penalties shrink coefficients asymptotically toward zero, never quite reaching it).

**ElasticNet** is simply the idea of using *both* penalties at once, added together, so the optimizer is pressured toward BOTH goals simultaneously: sparse (like L1) AND well-behaved/smooth in magnitude (like L2). The **EAD attack** (short for "Elastic-net Attacks to DNNs," introduced by Chen et al. in 2018) is the application of this combined penalty to crafting adversarial examples against neural networks.

```
   L1 ALONE                  L2 ALONE                  ELASTIC NET (L1 + L2)
   (max sparsity,            (smooth, low magnitude,    (sparse AND well-behaved)
    but nonzero entries       but never exactly zero)
    can be large/erratic)

   +---+---+---+---+        +---+---+---+---+        +---+---+---+---+
   |   |   | 9 |   |        | 1 | 1 | 2 | 1 |        |   |   | 3 |   |
   +---+---+---+---+   vs   +---+---+---+---+   =>   +---+---+---+---+
   |   |   |   |   |        | 1 | 2 | 1 | 1 |        |   |   |   |   |
   +---+---+---+---+        +---+---+---+---+        +---+---+---+---+
   Mostly zero, but the      Every pixel changes       Mostly zero, and the
   ONE change is huge        a little (dense)          nonzero change is
   (harder to keep subtle)                              moderate, not extreme
```

---

## 2. Recap: What L1 and L2 Each Do Alone

Quick refresher tying back to Section 2 of this module, framed specifically for this section's purposes:

| Penalty | Formula | Sparsity Behavior | Magnitude Behavior |
|---|---|---|---|
| **L1** | `sum(|delta_i|)` | Drives many entries to exactly zero (diamond corners touch loss contours) | Doesn't directly discourage large nonzero entries beyond their contribution to the sum |
| **L2** | `sum(delta_i^2)` | Almost never produces exact zeros (smooth ball has no corners) | Strongly discourages any single large entry, since squaring punishes big values disproportionately (an entry of 10 contributes 100 to the sum, not just 10) |

The L2 penalty's `squaring` behavior is the important detail for this section: because squaring a big number makes it much bigger relative to squaring a small number (10^2=100 is 100x bigger than 1^2=1, not just 10x), L2 penalties are very effective at preventing any *one* feature from taking on an extreme, conspicuous value. Combining this "anti-extreme-value" property with L1's "drive-to-zero" property is exactly what elastic net does.

---

## 3. Elastic Net: Combining Both

**Elastic net regularization**, originally developed in statistics (Zou & Hastie, 2005) for linear regression with many correlated features, is simply:

```
elastic_net_penalty(delta) = beta * ||delta||_1  +  ||delta||_2^2
```

Where `beta` is a weight controlling how much emphasis to put on the L1 (sparsity) term relative to the L2 (magnitude-control) term. Note we write `||delta||_2^2` (the L2 norm *squared*, i.e., simply `sum(delta_i^2)` without the square root) rather than the plain L2 norm -- this is a very common convention because the squared form is smoother and easier to differentiate, and it's what naturally falls out of many derivations. It still penalizes the same way in spirit: bigger magnitude, bigger penalty, with squaring amplifying large entries.

**Numeric sanity check**, reusing `delta = [0.01, 0.00, -0.30, 0.00, 0.02]` from Section 1:

```
||delta||_1     = 0.01 + 0 + 0.30 + 0 + 0.02              = 0.33
||delta||_2^2   = 0.01^2 + 0^2 + 0.30^2 + 0^2 + 0.02^2     = 0.0001 + 0.09 + 0.0004 = 0.0905

If beta = 1:
elastic_net_penalty = 1 * 0.33 + 0.0905 = 0.4205
```

Both terms contribute to the total penalty. If we increase `beta`, the L1 term's influence grows relative to the L2 term, pushing the optimizer harder toward exact zeros; if we decrease `beta`, the L2 term dominates, favoring smooth, spread-out (but never exactly zero) changes.

---

## 4. The EAD Attack Objective

EAD plugs this elastic-net penalty directly into the adversarial attack optimization problem, using the same overall structure we saw for L1-regularized attacks in Section 2:

```
minimize over delta:

    c * loss_attack(x + delta)   +   beta * ||delta||_1   +   ||delta||_2^2

subject to:  x + delta stays a valid input (e.g. pixel values in [0, 1])
```

Where:

- `loss_attack(x + delta)` is the same kind of attack-success loss as before (small when the model is successfully fooled toward the attacker's target).
- `c` controls how much we prioritize attack success versus keeping the perturbation small overall.
- `beta` controls the balance between the L1 (sparsity) and L2 (magnitude-smoothness) terms within the perturbation penalty itself.

This gives EAD **two knobs** (`c` and `beta`) instead of L1-only's one knob, letting an attacker independently tune "how hard to try to succeed" and "how to balance sparsity against smoothness" -- a strictly more flexible attack recipe than plain L1 or plain L2 alone.

In the original EAD paper, the attack-success loss `loss_attack` is typically the same margin-based loss popularized by the Carlini-Wagner (C&W) attack family: it measures the gap between the target class's logit (raw, pre-softmax score) and the highest logit among all other classes, pushed to be negative (meaning the target class's score exceeds every other class's score) with some margin `kappa` for confidence.

---

## 5. A Tiny Worked Numeric Example

Let's extend the 1D soft-thresholding example from Section 2 (L1-Induced Sparsity) to include the L2 term, so you can see exactly how adding L2 changes the optimal solution.

Recall the L1-only objective from Section 2:

```
objective_L1_only(delta) = (delta - 3)^2 + lambda * |delta|
```

which gave the soft-thresholding solution `delta* = sign(3) * max(|3| - lambda/2, 0)`.

Now add an L2 term with weight `mu` on top (this is a standard elastic-net-flavored objective; note the `(delta-3)^2` here plays the role of our simplified "attack loss," analogous to `loss_attack` above):

```
objective_elastic(delta) = (delta - 3)^2  +  lambda * |delta|  +  mu * delta^2
```

Taking the derivative and solving (skipping the algebra, this is a standard result) gives a **generalized soft-thresholding** solution:

```
delta* = soft_threshold(3, lambda/2) / (1 + mu)
```

Let's tabulate a few combinations to see the interaction:

| lambda | mu | soft_threshold(3, lambda/2) | delta* = .../(1+mu) | Interpretation |
|---|---|---|---|---|
| 0 | 0 | 3.0 | 3.0 | No penalty at all: full, unconstrained attack strength |
| 4 | 0 | 1.0 | 1.0 | Pure L1: shrunk toward zero, no extra magnitude control |
| 0 | 1 | 3.0 | 1.5 | Pure L2: shrunk by the `(1+mu)` denominator, but never hits exact zero unless numerator is 0 |
| 4 | 1 | 1.0 | 0.5 | Elastic net: sparsity pressure from lambda AND magnitude damping from mu, together |
| 6 | 0 | 0.0 | 0.0 | Pure L1, strong enough to hit exact zero |
| 6 | 2 | 0.0 | 0.0 | Elastic net still hits exact zero (L1's zero-producing power survives the L2 damping, since 0 divided by anything is still 0) |

**The key insight from the last two rows:** once the L1 term is strong enough to drive the numerator to exactly zero, adding an L2 term on top doesn't undo that zero (dividing zero by `(1+mu)` is still zero). This confirms that elastic net *keeps* L1's sparsity-inducing power while *also* damping the magnitude of whatever entries do stay nonzero -- exactly the "best of both worlds" behavior promised in Section 1's analogy.

---

## 6. How EAD Chooses Between L1 and L2 Weight

In practice, the original EAD algorithm uses a specific refinement worth knowing: after each gradient/shrinkage step, it keeps track of **two candidate perturbations** at every iteration:

1. The raw iterate produced by the proximal gradient step (before any special adjustment).
2. A version obtained by taking a weighted average ("momentum-like" blending) of the current and previous iterate.

It then evaluates the *actual* combined elastic-net objective (attack loss + L1 + L2) on both candidates and **keeps whichever one scores better**. This is a practical trick to squeeze out slightly better solutions from the optimization trajectory, and it foreshadows the "momentum" idea central to FISTA (Section 5), which EAD's optimizer is directly built on top of.

---

## 7. The Full EAD Algorithm, Step by Step

1. **Initialize** `delta = 0` (start with no perturbation) and pick starting values for `c` and `beta`.
2. **Iterate** (for a fixed number of steps, or until convergence):
   a. Compute the gradient of `loss_attack(x + delta)` with respect to `delta`.
   b. Take a gradient-descent step on the *combined* smooth part of the objective (the attack loss plus the L2 term -- both are differentiable everywhere, so ordinary gradient descent works fine on them).
   c. Apply the **soft-thresholding** operator (from Section 2) to the result, using threshold proportional to `beta`, to handle the *non-smooth* L1 term (recall: L1 has a sharp kink at zero that ordinary gradient descent can't handle directly, which is exactly why we need this separate "proximal" step -- this is the FISTA machinery, previewed here and detailed in Section 5).
   d. Clip so `x + delta` stays a valid input.
3. **Binary search over `c`** (repeat the whole inner loop above with different `c` values): if the attack succeeds easily, increase sparsity pressure (raise `beta` or lower `c`); if the attack fails, relax the constraints. Keep the best successful result found (measured by smallest L1 norm, i.e., fewest/least changed features, among all successful attempts).
4. **Return** the best `delta` found: the one with the smallest number of significantly-nonzero entries that still achieves misclassification.

```
    OUTER LOOP: binary search over c
      |
      +--> INNER LOOP: proximal gradient descent (FISTA-style)
             |
             +--> gradient step on (attack_loss + L2 term)
             +--> soft-threshold step on L1 term
             +--> clip to valid input range
             |
             +--> repeat for N iterations
      |
      +--> check: did the attack succeed? adjust c, repeat outer loop
      |
      +--> keep best (most sparse, still successful) result overall
```

---

## 8. Pseudocode

```python
def ead_attack(model, x, target_class, beta, c_values, inner_steps, step_size):
    """
    ElasticNet Attack (EAD), simplified.
    beta: weight on the L1 term relative to the L2 term
    c_values: a list of candidate c values to try (binary search in practice)
    """
    best_delta = None
    best_l1 = float("inf")

    for c in c_values:
        delta = [0.0] * len(x)
        prev_delta = [0.0] * len(x)

        for step in range(inner_steps):
            # --- smooth part: attack loss (weighted by c) + L2 term ---
            grad = compute_gradient(
                model, x, delta, target_class, weight=c
            )
            # add gradient contribution from the L2 term (2 * delta)
            for i in range(len(delta)):
                grad[i] += 2 * delta[i]

            candidate = [delta[i] - step_size * grad[i] for i in range(len(delta))]

            # --- non-smooth part: soft-threshold for the L1 term ---
            candidate = [soft_threshold(v, beta * step_size) for v in candidate]

            # --- keep within valid input range ---
            candidate = [
                clip(x[i] + candidate[i], 0.0, 1.0) - x[i]
                for i in range(len(x))
            ]

            prev_delta = delta
            delta = candidate

        # Did this c value produce a successful, sparse attack?
        x_adv = [x[i] + delta[i] for i in range(len(x))]
        if model.predict(x_adv) == target_class:
            this_l1 = l1_norm(delta)
            if this_l1 < best_l1:
                best_l1 = this_l1
                best_delta = delta

    return best_delta
```

---

## 9. EAD vs. Carlini-Wagner (C&W) vs. JSMA

| Attack | Primary Norm | Optimization Style | Sparsity Result | Typical Use Case |
|---|---|---|---|---|
| **C&W (L2)** | L2 | Gradient descent on margin-based loss + L2 penalty | Dense, minimal Euclidean distance | Module 9-style dense attacks; strong baseline for L2 robustness evaluation |
| **JSMA** | ~L0 (greedy) | Iterative saliency-pair selection, no explicit convex penalty | Extremely sparse (often very few features) | Sparse attacks against small/structured inputs; conceptual basis for saliency methods |
| **EAD** | L1 + L2 (elastic net) | Proximal gradient descent (FISTA-based), binary search over c | Sparse, with damped magnitude on nonzero entries -- often sparser than C&W L2 with comparable success rates | General-purpose sparse attack; strong empirical L1 results, used as an L0/L1 robustness benchmark |

The original EAD paper reports that its attacks are often *even sparser* than JSMA's while remaining reliably successful, and dramatically more efficient than exhaustively searching feature subsets -- largely because the elastic-net-regularized optimization inherits the strong convergence guarantees of proximal gradient methods (see FISTA, Section 5), rather than JSMA's more ad hoc greedy pairwise search (see Section 6).

---

## 10. Real-World Walkthrough -- Crafting a Sparse Adversarial Digit

Let's trace a simplified but complete run of the EAD algorithm (Section 7's step list) against a tiny "image" classifier, so the interplay between `c`, `beta`, and the FISTA-style inner loop is fully concrete.

**Setup.** Imagine a drastically simplified 6-pixel grayscale "image" (think of it as a tiny down-sampled digit), pixels `p1...p6`, each in `[0, 1]`. A (simplified, invented) two-class model outputs a "logit gap" `g(x) = score_target(x) - score_current(x)`; the model misclassifies (predicts the attacker's target) once `g(x) > 0`. Suppose at the original image, the gap is:

```
Original pixels: p = [0.9, 0.1, 0.8, 0.2, 0.9, 0.1]
g(p) = -4.0     (strongly favors the CURRENT, correct label)

Gradient of g with respect to each pixel at this point:
grad = [0.5, -0.3, 1.8, -0.2, 0.4, -0.1]
```

(Positive gradient entries mean "increasing this pixel helps flip toward the target"; negative entries mean "decreasing this pixel helps.")

**Round 1 -- large c, no sparsity pressure (beta = 0), to confirm the attack CAN succeed at all.** Running plain gradient ascent on `g` (ignoring sparsity entirely, purely to check feasibility) for a few steps drives all 6 pixels in their helpful direction and eventually flips the model (g becomes positive) after touching every pixel by a moderate amount -- this reproduces a dense, Module-9-style outcome and confirms the attack is achievable in principle. This "dense baseline" run is a standard sanity check before adding sparsity pressure.

**Round 2 -- introduce beta, run the FISTA-style inner loop (Section 7, step 2).** Set `beta = 0.6` (moderate sparsity pressure) and `c = 1.0`. Using the proximal gradient loop from Section 8's pseudocode:

```
Iteration 1:
  gradient step (using grad above, step_size=0.2):
    p1_half = 0.9 + 0.2*0.5  = 1.00  (clipped to 1.0, already near max)
    p2_half = 0.1 + 0.2*-0.3 = 0.04
    p3_half = 0.8 + 0.2*1.8  = 1.16  (will clip to 1.0)
    p4_half = 0.2 + 0.2*-0.2 = 0.16
    p5_half = 0.9 + 0.2*0.4  = 0.98
    p6_half = 0.1 + 0.2*-0.1 = 0.08

  soft-threshold each CHANGE (delta = p_half - p_original) with
  threshold = beta * step_size = 0.6*0.2 = 0.12:
    delta1 = soft_threshold(0.10, 0.12) = 0.0   (change was only +0.10, below threshold -> zeroed)
    delta2 = soft_threshold(-0.06, 0.12) = 0.0  (small change, zeroed)
    delta3 = soft_threshold(0.36, 0.12) = 0.24  (large change survives, shrunk a bit)
    delta4 = soft_threshold(-0.04, 0.12) = 0.0  (zeroed)
    delta5 = soft_threshold(0.08, 0.12) = 0.0   (zeroed)
    delta6 = soft_threshold(-0.02, 0.12) = 0.0  (zeroed)

  x_adv after iteration 1: [0.9, 0.1, 1.04, 0.2, 0.9, 0.1]
  (only p3 changed -- L0 = 1 after just one iteration!)
```

Notice how dramatically the soft-thresholding step (with `beta=0.6`) collapsed 5 of the 6 proposed changes to exactly zero, keeping only `p3` -- the pixel with by far the largest gradient magnitude (1.8, more than 3x any other pixel). This single iteration already demonstrates the diamond-corner geometry from Section 4 of the L1 section acting directly on real pixel updates.

**Checking the model after iteration 1:** suppose `g(x_adv)` after this one change is still `-1.2` (moved from -4.0, good progress, but not yet flipped). The algorithm continues iterating (re-computing the gradient at the new point, repeating the gradient-then-threshold steps), and after several more iterations (not traced in full detail here for brevity), suppose it converges to:

```
Final x_adv:  [0.9, 0.1, 1.0, 0.2, 0.9, 0.1]     (only p3 changed, now fully at 1.0)
Final g(x_adv) = +0.3   (success! model now favors the target class)
L1 norm of delta = 0.20   (just the change to p3: 1.0 - 0.8)
L0 norm of delta = 1     (only 1 pixel touched)
```

**Round 3 -- binary search over c (Section 7, step 3).** Suppose this run with `c=1.0, beta=0.6` succeeded with L0=1. The outer binary search would now try a *larger* `beta` (e.g., 0.8) to see if an even sparser (though here, L0=1 is already the practical floor for this toy example) or smaller `c` (weaker attack pressure) still succeeds, keeping whichever successful configuration has the smallest final L1 norm. In this toy case, L0=1 is optimal, so the search would converge here, or possibly find a configuration with an even smaller L1 magnitude touching the same single pixel.

**Comparing to Round 1's dense baseline:** the L1-regularized EAD run touched exactly 1 pixel out of 6 (versus all 6 in the naive dense run), directly illustrating why EAD's combined attack-loss-plus-elastic-net-penalty objective reliably produces far sparser results than an unregularized attack, using nothing more exotic than a gradient step immediately followed by a soft-threshold step, repeated a handful of times.

## 11. Security Angle

- **Tunable stealth-vs-reliability tradeoff.** EAD's two knobs (`c` and `beta`) give a red teamer a direct dial between "definitely fool the model" and "touch as little as possible." This is directly useful when crafting proof-of-concept evasion samples for a client engagement: you can dial up sparsity to demonstrate a minimal, highly surgical bypass (e.g., "changing just these 3 bytes in this file evades the malware classifier"), which is a much more persuasive and reproducible finding than a diffuse, dense perturbation that touches everything.

- **Benchmark for robustness evaluation.** Because EAD reliably finds sparse, magnitude-controlled adversarial examples, it (along with its L1/L0 cousins) is commonly used as a standard *evaluation* attack -- if a defended model claims robustness against dense L2/L-infinity attacks (Module 9 style) but has never been tested against a strong sparse attack like EAD, that claim is incomplete. Offensive AI practitioners should always test across multiple norm families, not just the popular L-infinity one.

- **Realistic constrained-feature domains.** Elastic net's ability to keep nonzero entries damped (not just sparse but also not extreme) matters in domains where an extreme single-feature change would itself be a red flag (e.g., a malware file where one byte suddenly jumping to an implausible value might trip a separate integrity/format check). EAD's L2 component discourages exactly that kind of conspicuous extreme value, making its sparse perturbations more plausible than a naive L0/L1-only attack that doesn't care about magnitude at all.

---

## 12. Key Takeaways

- **Elastic net regularization** combines the L1 penalty (drives sparsity, exact zeros) and the L2 penalty (damps magnitude of remaining nonzero entries, discourages extreme values) into a single weighted sum.
- **EAD (ElasticNet Attack to DNNs)** applies this combined penalty inside the adversarial attack optimization objective, alongside the usual attack-success loss.
- EAD has **two tunable knobs**: `c` (attack-success weight, tuned via binary search) and `beta` (L1-vs-L2 balance within the perturbation penalty).
- A worked 1D example shows the elastic-net solution is a scaled-down version of the pure-L1 soft-thresholding solution -- it **preserves L1's exact-zero property** while damping the magnitude of nonzero entries via the `(1+mu)` denominator.
- EAD's optimizer keeps two candidate iterates per step (raw vs. momentum-blended) and picks the better one -- foreshadowing the "momentum" mechanism central to FISTA.
- Empirically, EAD tends to produce **sparser** perturbations than L2-only attacks (like C&W) while remaining more systematically optimized than JSMA's greedy pairwise search.
- Security-wise, EAD's dual knobs let a red team dial in exactly how much stealth vs. reliability an evasion proof-of-concept needs, and its damped magnitude makes sparse changes more plausible in constrained feature domains.

---

*Next up: FISTA Optimization -- the Fast Iterative Shrinkage-Thresholding Algorithm that efficiently solves the L1/elastic-net-regularized optimization problem underlying EAD, explained through the lens of proximal gradient descent.*
