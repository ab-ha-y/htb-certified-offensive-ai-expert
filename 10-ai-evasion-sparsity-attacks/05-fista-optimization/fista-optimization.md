# FISTA Optimization

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - Sparsity Attacks | Section: FISTA Optimization

---

## Table of Contents

1. [The Analogy: Rolling Downhill, But the Hill Has a Crack in It](#1-the-analogy-rolling-downhill-but-the-hill-has-a-crack-in-it)
2. [Recap: Smooth vs. Non-Smooth Parts of the Objective](#2-recap-smooth-vs-non-smooth-parts-of-the-objective)
3. [Proximal Gradient Descent, Explained Simply](#3-proximal-gradient-descent-explained-simply)
4. [A Tiny Worked Numeric Example](#4-a-tiny-worked-numeric-example)
5. [ISTA: The Non-Accelerated Baseline](#5-ista-the-non-accelerated-baseline)
6. [Adding Momentum: From ISTA to FISTA](#6-adding-momentum-from-ista-to-fista)
7. [The Full FISTA Algorithm](#7-the-full-fista-algorithm)
8. [Pseudocode](#8-pseudocode)
9. [Why FISTA Converges Faster -- Intuition](#9-why-fista-converges-faster----intuition)
10. [FISTA's Role Inside EAD](#10-fistas-role-inside-ead)
11. [Real-World Walkthrough -- Comparing ISTA vs. FISTA Convergence Speed](#11-real-world-walkthrough----comparing-ista-vs-fista-convergence-speed)
12. [Security Angle](#12-security-angle)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. The Analogy: Rolling Downhill, But the Hill Has a Crack in It

Picture a ball rolling downhill to find the lowest point of a valley -- that's the standard "gradient descent" picture from Module 1. Ordinary gradient descent works great when the entire hill is smooth: at every point, you can compute a well-defined slope and roll a little further downhill.

Now imagine the valley has a **sharp crack or ridge** running through it -- a spot where the slope is undefined or jumps abruptly (this is exactly what the L1 penalty's `|delta|` does at `delta = 0`: it has a sharp kink, no single well-defined slope right at that point). A ball rolling by ordinary physics doesn't know how to smoothly roll across that crack; it needs a special rule for what to do exactly at (or near) the crack.

**FISTA** (Fast Iterative Shrinkage-Thresholding Algorithm) is a recipe for handling exactly this situation: an optimization objective that is the *sum* of one smooth part (that ordinary gradient descent handles fine) and one non-smooth part (the L1 penalty's kink, that needs a special rule). It alternates between "roll downhill on the smooth part" and "apply the special crack-crossing rule (soft-thresholding) for the non-smooth part," and it adds a clever "momentum" trick borrowed from physics (think of a ball with actual weight and inertia, not a weightless point) to reach the bottom of the valley dramatically faster than doing this alternation naively.

```
     Smooth hill (no crack):              Hill with a crack (L1 kink):

          \                                    \        |
           \                                     \       |  <- sharp kink
            \___                                  \___  /   at delta=0
                \___                                   \/
                    \___.                          ordinary gradient
                        (minimum)                  descent doesn't know
                                                     what to do exactly here

     Ordinary gradient descent           FISTA: gradient step on the
     works fine everywhere.              smooth part, THEN a special
                                          "shrink toward zero" step
                                          (soft-thresholding) to handle
                                          the crack correctly.
```

---

## 2. Recap: Smooth vs. Non-Smooth Parts of the Objective

Every optimization objective we've built in this module (Sections 2 and 4) has this exact two-part structure:

```
objective(delta) = SMOOTH_PART(delta)  +  NON_SMOOTH_PART(delta)

SMOOTH_PART      = loss_attack(x + delta)  [+ L2 term, if using elastic net]
NON_SMOOTH_PART  = c * ||delta||_1   (or  beta * ||delta||_1  for EAD)
```

The **smooth part** is differentiable everywhere -- you can always compute a well-defined gradient, no matter what value `delta` takes. The **non-smooth part** (the L1 term) has that sharp kink exactly at `delta_i = 0` for each entry -- its "slope" jumps discontinuously from -1 (just below zero) to +1 (just above zero), so a single well-defined derivative doesn't exist right at zero.

This specific structure -- "smooth function plus a simple non-smooth penalty" -- is common enough across machine learning (not just adversarial attacks; LASSO regression has the identical structure) that there's a whole family of general-purpose algorithms designed for it, called **proximal gradient methods**. FISTA is the most famous, fastest member of that family for this particular type of problem.

---

## 3. Proximal Gradient Descent, Explained Simply

Here's the core idea, broken into two alternating moves, each one easy on its own:

**Move 1 -- Gradient step (handles the smooth part).** Take a normal gradient descent step, but *only* using the gradient of the smooth part of the objective. Ignore the L1 term entirely for this move.

```
delta_halfway = delta_current - step_size * gradient(SMOOTH_PART, delta_current)
```

**Move 2 -- Proximal step (handles the non-smooth part).** Apply a special correction, called the **proximal operator**, that accounts for the non-smooth penalty. For the L1 penalty specifically, this proximal operator has a simple, exact closed-form solution: it's precisely the **soft-thresholding** function we already met in Section 2 of this module (and reused in Section 4).

```
delta_next = soft_threshold(delta_halfway, threshold)
```

The word "proximal" here means "nearby" -- the proximal step finds a point that is *close to* `delta_halfway` while also respecting the non-smooth penalty as much as reasonably possible. This is a general mathematical concept (the **proximal operator** of any convex function), but you don't need the general definition -- for our purposes, "proximal step for an L1 penalty" is always exactly the soft-thresholding operation we already know.

```
    ONE FISTA/ISTA ITERATION:

    delta_current
         |
         v
    [ GRADIENT STEP ]   <-- uses ONLY the smooth part's gradient
         |
         v
    delta_halfway
         |
         v
    [ PROXIMAL / SOFT-THRESHOLD STEP ]  <-- handles the L1 kink exactly
         |
         v
    delta_next
```

---

## 4. A Tiny Worked Numeric Example

Let's trace through a couple of iterations by hand, extending the running 1D example from Sections 2 and 4.

Objective: `objective(delta) = (delta - 3)^2 + lambda * |delta|`, with `lambda = 4`.

We already know the *exact* answer from Section 2's closed-form solution: `delta* = soft_threshold(3, lambda/2) = soft_threshold(3, 2) = 1.0`.

Let's confirm the proximal gradient algorithm actually converges to that, starting from `delta_0 = 0` with step size `0.25`:

**Iteration 1:**
- Gradient of smooth part `(delta-3)^2` at `delta=0`: derivative is `2*(delta-3) = 2*(0-3) = -6`.
- Gradient step: `delta_halfway = 0 - 0.25 * (-6) = 0 + 1.5 = 1.5`
- Proximal step (soft-threshold with threshold `= lambda * step_size = 4 * 0.25 = 1.0`): `soft_threshold(1.5, 1.0) = 1.5 - 1.0 = 0.5`
- `delta_1 = 0.5`

**Iteration 2:**
- Gradient at `delta=0.5`: `2*(0.5-3) = 2*(-2.5) = -5`
- Gradient step: `delta_halfway = 0.5 - 0.25*(-5) = 0.5 + 1.25 = 1.75`
- Proximal step: `soft_threshold(1.75, 1.0) = 1.75 - 1.0 = 0.75`
- `delta_2 = 0.75`

**Iteration 3:**
- Gradient at `delta=0.75`: `2*(0.75-3) = 2*(-2.25) = -4.5`
- Gradient step: `delta_halfway = 0.75 - 0.25*(-4.5) = 0.75 + 1.125 = 1.875`
- Proximal step: `soft_threshold(1.875, 1.0) = 0.875`
- `delta_3 = 0.875`

```
Iteration:   0        1        2        3       ...      target
delta:       0.0  -->  0.5  -->  0.75  -->  0.875  -->  ...  --> 1.0
```

The sequence is climbing steadily toward `1.0`, the exact answer we computed algebraically. This is exactly the behavior we should expect and exactly what the pseudocode in Section 8 implements in a loop. This particular (non-accelerated) version of the algorithm, doing exactly this "gradient step, then soft-threshold" alternation with no extra tricks, is called **ISTA**.

---

## 5. ISTA: The Non-Accelerated Baseline

**ISTA** (Iterative Shrinkage-Thresholding Algorithm) is precisely the loop we just traced by hand: alternate gradient steps and soft-thresholding steps, using only the *current* iterate at each step.

```python
def ista(x0, gradient_fn, threshold, step_size, num_iters):
    """
    Basic ISTA: no momentum, uses only the current point at each step.
    """
    x = x0
    for i in range(num_iters):
        x_halfway = x - step_size * gradient_fn(x)
        x = soft_threshold(x_halfway, threshold)
    return x
```

ISTA is guaranteed to converge to the true minimum (because the objective is convex, per Section 2's discussion of convexity), but its convergence *rate* is relatively slow: theoretically, the gap between ISTA's current solution and the true optimum shrinks proportionally to `1/k` after `k` iterations. This means to cut your error in half, you roughly need to double your total number of iterations -- workable, but slow for high-dimensional problems (like a 150,000-pixel color image) where every iteration itself is expensive (each one requires a full forward and backward pass through a neural network).

---

## 6. Adding Momentum: From ISTA to FISTA

**FISTA**'s key innovation (Beck & Teboulle, 2009) is to add a **momentum term**: instead of computing the gradient step from only the *current* iterate, compute it from a cleverly weighted combination of the current iterate and the *previous* iterate. This is directly analogous to the "momentum" trick used in modern neural network training optimizers (like SGD with momentum, or Adam) -- if you've been consistently moving in a helpful direction for several steps in a row, lean into that direction a bit more, rather than recomputing from scratch each time as if you'd just arrived at your current spot.

**Plain-English physics analogy:** ISTA is like a ball with zero mass -- at every instant, it moves purely according to the *current* local slope, with no "carry-over" from its previous motion. FISTA is like a ball with real mass and momentum -- it keeps some of its previous velocity, letting it "coast" through flat or noisy regions and builds up speed when consistently rolling in a good direction, converging to the bottom of the valley noticeably faster.

The specific momentum weighting FISTA uses comes from a sequence of numbers `t_k` defined by:

```
t_1 = 1
t_{k+1} = (1 + sqrt(1 + 4 * t_k^2)) / 2
```

And the momentum-blended point used for the *next* gradient step is:

```
y_k = x_k + ((t_k - 1) / t_{k+1}) * (x_k - x_{k-1})
```

You don't need to memorize this formula -- what matters is the shape of the idea: `y_k` is the current point `x_k`, nudged further *in the same direction it was just moving* (`x_k - x_{k-1}` is literally "how far and which way we just moved"), scaled by a factor that is carefully tuned to provably give the fastest possible convergence rate for this class of problems.

---

## 7. The Full FISTA Algorithm

1. **Initialize**: `x_0 = x_1 = starting point` (e.g., `delta = 0`), `t_1 = 1`.
2. **For each iteration `k = 1, 2, 3, ...`:**
   a. Compute the momentum-blended point: `y_k = x_k + ((t_k - 1)/t_{k+1}) * (x_k - x_{k-1})` (using `t_{k+1}` computed from the update rule in Section 6; on the very first iteration this term is zero since there's no previous point yet).
   b. Gradient step from `y_k` (not from `x_k` directly!): `x_{k+1}_halfway = y_k - step_size * gradient(SMOOTH_PART, y_k)`.
   c. Proximal/soft-threshold step: `x_{k+1} = soft_threshold(x_{k+1}_halfway, threshold)`.
   d. Update `t_{k+1}` for the next iteration using the formula from Section 6.
3. **Repeat** until convergence (the change between iterations becomes negligibly small) or a fixed iteration budget is exhausted.
4. **Return** the final `x_k` (this is our sparse perturbation `delta`).

The crucial difference from plain ISTA, stated once more for emphasis: **the gradient in step 2b is computed at the momentum-blended point `y_k`, not at the raw current iterate `x_k`.** That one change is FISTA's entire trick, and it provably improves the convergence rate from ISTA's `O(1/k)` to FISTA's `O(1/k^2)` -- meaning to cut the error by the same amount, FISTA needs roughly the *square root* of the number of iterations ISTA would need. For 1000 iterations of ISTA-level accuracy, FISTA might need only ~32.

---

## 8. Pseudocode

```python
def fista(x0, smooth_gradient_fn, threshold, step_size, num_iters):
    """
    FISTA: accelerated proximal gradient descent.
    smooth_gradient_fn: gradient of the SMOOTH part of the objective only
                         (e.g. attack_loss, or attack_loss + L2 term)
    threshold: the soft-thresholding cutoff (proportional to the L1 weight)
    """
    x_prev = x0
    x_curr = x0
    t_curr = 1.0

    for k in range(num_iters):
        # --- momentum blending step ---
        t_next = (1 + sqrt(1 + 4 * t_curr**2)) / 2
        momentum_coeff = (t_curr - 1) / t_next
        y = [x_curr[i] + momentum_coeff * (x_curr[i] - x_prev[i])
             for i in range(len(x_curr))]

        # --- gradient step, evaluated AT y, not at x_curr ---
        grad = smooth_gradient_fn(y)
        x_halfway = [y[i] - step_size * grad[i] for i in range(len(y))]

        # --- proximal / soft-threshold step ---
        x_next = [soft_threshold(v, threshold) for v in x_halfway]

        x_prev = x_curr
        x_curr = x_next
        t_curr = t_next

    return x_curr
```

**Quick trace of the momentum coefficient over the first few iterations** (starting `t_1 = 1`):

```
k=1: t_1 = 1.0,  t_2 = (1 + sqrt(1+4))/2 = (1+2.236)/2 = 1.618
     momentum_coeff = (1.0 - 1)/1.618 = 0.0   <- no momentum yet (first step)
k=2: t_2 = 1.618, t_3 = (1 + sqrt(1+4*1.618^2))/2 ≈ 2.0
     momentum_coeff = (1.618-1)/2.0 = 0.309   <- now leaning into recent direction
k=3: t_3 = 2.0,   t_4 ≈ 2.414
     momentum_coeff = (2.0-1)/2.414 = 0.414   <- momentum weight is growing
```

Notice momentum starts at exactly zero (there's no "previous direction" yet on the very first step, matching ISTA exactly) and grows over subsequent iterations as a consistent direction of travel is established -- exactly the "build up speed while rolling downhill" physics analogy from Section 6.

---

## 9. Why FISTA Converges Faster -- Intuition

Here's a simplified, non-rigorous way to build intuition for the speedup, without diving into the formal proof:

- ISTA always computes its next gradient step from wherever it currently is, with no "look-ahead."
- FISTA computes its gradient step from a point slightly *ahead* of its current position, in the direction it's been consistently moving. This is like a scout running ahead of a hiking group to check the terrain before the group commits to a direction, rather than the whole group blindly taking one step at a time and re-evaluating from scratch.
- Over many iterations, this look-ahead lets FISTA take effectively larger, better-aimed strides toward the minimum without overshooting as much as a naively larger step size in plain ISTA would.

The formal result (proven in the original FISTA paper) is that this specific momentum weighting is not arbitrary -- it's precisely the schedule that achieves the theoretically optimal `O(1/k^2)` convergence rate for this class of problems (smooth + simple non-smooth objectives), matching a known lower bound for what any "first-order" method (one that only uses gradient information, not second derivatives) can achieve.

---

## 10. FISTA's Role Inside EAD

Tying this back to Section 4: EAD's inner optimization loop (the "gradient step, then soft-threshold step, repeated many times" procedure) *is* FISTA, applied to the elastic-net objective (attack loss + L2 term treated as the smooth part, L1 term treated as the non-smooth part requiring the proximal/soft-threshold step). The "two candidate iterates, keep the better one" trick mentioned in Section 4's discussion of EAD is a direct descendant of FISTA's momentum-blended point `y_k` versus the raw iterate `x_k` -- EAD evaluates the true combined objective on both and keeps whichever is better, adding a small extra safeguard on top of standard FISTA.

This is why FISTA is presented as its own dedicated section in this module: it's not just one attack among many, it's the **general-purpose optimization engine** that makes EAD (and any other L1/elastic-net-regularized attack) computationally practical in the first place.

---

## 11. Real-World Walkthrough -- Comparing ISTA vs. FISTA Convergence Speed

Let's directly compare ISTA and FISTA side by side on the exact same 1D problem from Section 4, so the acceleration effect is visible in real numbers rather than just asymptotic notation.

Recall the objective `objective(delta) = (delta - 3)^2 + lambda * |delta|`, `lambda = 4`, true optimum `delta* = 1.0`, step size `0.25`.

**ISTA's trace** (already computed in Section 4): `0.0 -> 0.5 -> 0.75 -> 0.875 -> ...`, slowly approaching 1.0.

**FISTA's trace**, using the momentum coefficients computed in Section 8 (`0.0, 0.309, 0.414, ...`):

```
k=1 (momentum_coeff = 0.0, identical to ISTA's first step since no history yet):
  y_1 = x_1 = 0.0  (no previous point to blend with)
  gradient at y_1=0.0: 2*(0-3) = -6
  x_halfway = 0.0 - 0.25*(-6) = 1.5
  x_2 = soft_threshold(1.5, 1.0) = 0.5
  (identical to ISTA so far, as expected)

k=2 (momentum_coeff = 0.309):
  y_2 = x_2 + 0.309*(x_2 - x_1) = 0.5 + 0.309*(0.5 - 0.0) = 0.5 + 0.155 = 0.655
  gradient at y_2=0.655: 2*(0.655-3) = -4.69
  x_halfway = 0.655 - 0.25*(-4.69) = 0.655 + 1.1725 = 1.8275
  x_3 = soft_threshold(1.8275, 1.0) = 0.8275
  (FISTA is already at 0.8275, versus ISTA's 0.75 at the same iteration count --
   FISTA has pulled ahead by "looking ahead" using momentum)

k=3 (momentum_coeff = 0.414):
  y_3 = x_3 + 0.414*(x_3 - x_2) = 0.8275 + 0.414*(0.8275-0.5) = 0.8275 + 0.414*0.3275
      = 0.8275 + 0.1356 = 0.9631
  gradient at y_3=0.9631: 2*(0.9631-3) = -4.0738
  x_halfway = 0.9631 - 0.25*(-4.0738) = 0.9631 + 1.01845 = 1.9816
  x_4 = soft_threshold(1.9816, 1.0) = 0.9816
  (FISTA is at 0.9816, versus ISTA's 0.875 at the same iteration count)
```

**Side-by-side comparison table:**

| Iteration | ISTA's delta | FISTA's delta | True optimum |
|---|---|---|---|
| 0 | 0.000 | 0.000 | 1.0 |
| 1 | 0.500 | 0.500 | 1.0 |
| 2 | 0.750 | 0.8275 | 1.0 |
| 3 | 0.875 | 0.9816 | 1.0 |
| ... | slowly -> 1.0 | already ~98% of the way there | 1.0 |

After just 3 iterations, FISTA (0.9816) is already within about 2% of the true optimum, while ISTA (0.875) is still about 12% away -- and this gap would widen further (in relative terms) for higher-dimensional, harder problems, which is exactly why the formal `O(1/k^2)` vs `O(1/k)` convergence rate distinction from Section 5 matters in practice, not just in theory. For a real adversarial attack against a large neural network, where each single iteration requires an expensive forward-and-backward pass, this kind of speedup can mean the difference between an attack loop that finishes in seconds versus one that takes minutes-to-hours to reach the same quality of sparse solution -- directly affecting how many candidate adversarial examples a red team can generate within an engagement's time budget.

## 12. Security Angle

- **Speed matters for red team engagements.** A slow optimizer that takes thousands of iterations per adversarial example, each requiring a full forward/backward pass through a large neural network, can make crafting even a single evasive sample take unacceptably long during a time-boxed engagement. FISTA's accelerated convergence (needing roughly the square root of the iterations that plain ISTA would) is a direct practical enabler of running sparse attacks at scale -- e.g., generating a large batch of evasive malware samples or adversarial images to test a defense's overall robustness, not just a handful of cherry-picked ones.

- **General-purpose tool, not attack-specific.** FISTA itself is a generic convex optimization algorithm used across statistics, signal processing, and compressed sensing -- it is not an "attack tool" in isolation. This is a good illustration of the broader offensive-AI theme: attackers often don't need to invent new mathematics, they need to correctly recognize that an existing, well-studied optimization technique (originally built for, say, medical image reconstruction) applies directly to their adversarial objective.

- **Understanding convergence helps you judge attack claims.** If a paper or tool claims to have found "the sparsest possible" adversarial example using a gradient-based method, remember: FISTA (and ISTA) find the true minimum of the *relaxed* L1/elastic-net objective, not necessarily the true minimum of the original NP-hard L0 problem (Section 1). Knowing this distinction lets you correctly evaluate how strong a "provably minimal" sparsity claim really is.

---

## 13. Key Takeaways

- **FISTA** solves optimization objectives shaped like "smooth part + simple non-smooth part" (exactly the shape of L1/elastic-net-regularized attack objectives from Sections 2 and 4).
- The core mechanism is **proximal gradient descent**: alternate a normal gradient step (on the smooth part only) with a **proximal step** -- for an L1 penalty, this proximal step is exactly the **soft-thresholding** operator.
- **ISTA** is the plain (non-accelerated) version of this alternation; it converges at rate `O(1/k)`.
- **FISTA** adds a **momentum** term (computing the gradient step from a look-ahead point `y_k`, blended from the current and previous iterate), improving the convergence rate to `O(1/k^2)` -- provably optimal for this class of problems.
- A hand-traced numeric example confirmed the iterative "gradient step, then soft-threshold" loop climbs steadily toward the exact closed-form answer computed algebraically in earlier sections.
- FISTA is the **optimization engine underneath EAD** (Section 4) -- it's what makes elastic-net-regularized sparse attacks computationally practical rather than just theoretically nice.
- Security-wise, faster convergence means faster, more scalable adversarial-example generation for red team engagements, and understanding FISTA's convex, relaxed nature helps you correctly interpret claims about "minimal" or "optimal" sparse attacks.

---

*Next up: JSMA (Jacobian-based Saliency Map Attack) -- the classic greedy algorithm that repeatedly picks the pair of features with the highest combined saliency and pushes them to their extreme values, iterating until the model misclassifies.*
