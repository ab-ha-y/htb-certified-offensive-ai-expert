# I-FGSM: Iterative Fast Gradient Sign Method

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - First-Order Attacks | Section: I-FGSM

---

## Table of Contents

1. [The Motivation: One Step Isn't Always Enough](#1-the-motivation-one-step-isnt-always-enough)
2. [The Plain-English Idea](#2-the-plain-english-idea)
3. [What Is "Projection Back into the Epsilon-Ball"?](#3-what-is-projection-back-into-the-epsilon-ball)
4. [Deriving I-FGSM from FGSM](#4-deriving-i-fgsm-from-fgsm)
5. [The Formula](#5-the-formula)
6. [Choosing the Step Size Alpha](#6-choosing-the-step-size-alpha)
7. [Worked Numeric Example: Walking Downhill (Uphill) in Small Steps](#7-worked-numeric-example-walking-downhill-uphill-in-small-steps)
8. [Pseudocode](#8-pseudocode)
9. [Visualizing the Iterative Path](#9-visualizing-the-iterative-path)
10. [Why Iteration Increases Strength but Reduces Transferability](#10-why-iteration-increases-strength-but-reduces-transferability)
11. [Comparison Table: FGSM vs. I-FGSM](#11-comparison-table-fgsm-vs-i-fgsm)
12. [Security Angle](#12-security-angle)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. The Motivation: One Step Isn't Always Enough

Both worked examples in the previous two files (plain FGSM and Targeted FGSM) ended the same way: the single gradient step moved the model's output in the *right direction*, but didn't fully flip the decision. This isn't a coincidence -- it's the expected behavior of a method that trusts the local linearity assumption for one, possibly large, single jump.

I-FGSM (Iterative FGSM), introduced by Alexey Kurakin, Ian Goodfellow, and Samy Bengio in *"Adversarial Examples in the Physical World"* (2016), fixes this by asking a simple question: *what if, instead of one big leap of size `epsilon`, we take many small leaps, re-checking our direction after every single one?*

---

## 2. The Plain-English Idea

### The Analogy

Recall the "walking a curvy road while blindfolded, trusting a short straight-line guess" analogy from the local linearity file. Plain FGSM is like being told the road's direction *once*, at the very start, and then walking the *entire* remaining distance in a straight line based on that one reading -- risky if the road curves at all along the way.

I-FGSM is like checking the road's direction every few steps: walk a *little* bit in the direction you were told, stop, ask again "which way now?", walk a little more, and repeat. Because each individual step is small, the local linearity assumption (the "straight line is a good approximation over a short distance" idea) stays valid at every single step, even though the *total* distance traveled by the end can be just as large as -- or even larger than -- a single FGSM jump.

---

## 3. What Is "Projection Back into the Epsilon-Ball"?

There's a subtlety: if you take, say, 10 small steps of size `alpha = epsilon / 4` each, and each step can independently push a feature in whatever direction its *current* local gradient suggests, the *total* accumulated change to that feature could end up being much larger than the original overall budget `epsilon` allows (imagine one feature getting pushed positive on 8 out of 10 steps -- its total drift could exceed `epsilon` even though each individual step was small).

To prevent this, after every single step, I-FGSM **clips** (projects) the *cumulative* perturbation so that no feature's total change exceeds `epsilon` in either direction. This operation is called "projection back into the epsilon-ball," and for the L-infinity norm it has an extremely simple form: just clamp every feature of the *total* perturbation to the range `[-epsilon, +epsilon]`.

### The Analogy

Imagine you're allowed to wander at most 5 meters from a lamppost (your starting point) over the course of a walk, but you're free to change direction as often as you like along the way. After each small step, you check: "am I still within 5 meters of the lamppost?" If a step would take you further than that, you get pulled back to the boundary (5 meters out, in whatever direction you were heading) rather than being allowed to drift arbitrarily far. That "pulling back to the boundary if you've gone too far" is exactly what projection does, feature by feature.

---

## 4. Deriving I-FGSM from FGSM

I-FGSM is best understood as literally just "FGSM, called in a loop, with a safety clip." Concretely:

**Step 1**: Instead of jumping the full `epsilon` in one shot, define a smaller per-step size, `alpha` (with `alpha < epsilon`, often something like `epsilon / T` for `T` total steps, or another small fixed value).

**Step 2**: At each iteration `t`, compute the gradient of the loss **at the current adversarial input** `x_adv_t` (not the original `x` -- this is the crucial re-linearization step motivated in Section 9 of `local-linearity-assumption.md`):

```
grad_t = gradient_x( L(model(x_adv_t), y_true) )
```

**Step 3**: Take one small FGSM-style step from the *current* point:

```
x_adv_(t+1) = x_adv_t + alpha * sign(grad_t)
```

**Step 4**: Project the result back so the *total* perturbation from the original `x` never exceeds the overall budget `epsilon` (and, for valid inputs like images, also clip to the valid data range, e.g., `[0, 1]`):

```
x_adv_(t+1) = clip( x_adv_(t+1), x - epsilon, x + epsilon )
x_adv_(t+1) = clip( x_adv_(t+1), valid_min, valid_max )
```

**Step 5**: Repeat Steps 2-4 for a fixed number of iterations `T`, or until the attack succeeds (the model's prediction flips).

---

## 5. The Formula

```
x_adv_0 = x                                                (start at the original input)

x_adv_(t+1) = clip_{x, epsilon}(  x_adv_t + alpha * sign( gradient_x( L(model(x_adv_t), y_true) ) )  )

  x_adv_t              = the adversarial input after t iterations
  alpha                = the per-step size (small, alpha < epsilon)
  clip_{x, epsilon}(.) = project every feature back into [x_i - epsilon, x_i + epsilon],
                          and also into the valid input range (e.g., [0, 1] for images)
  T                    = total number of iterations
```

For the **targeted** version, swap in `y_target` and flip the sign, exactly as in the previous file:

```
x_adv_(t+1) = clip_{x, epsilon}(  x_adv_t - alpha * sign( gradient_x( L(model(x_adv_t), y_target) ) )  )
```

This combined "iterate + project" approach, when the number of steps is large and the step direction is recomputed carefully, is also the direct ancestor of a very well-known, even stronger attack called **PGD (Projected Gradient Descent)** -- I-FGSM is essentially PGD's simpler, earlier sibling, differing mainly in how the starting point and step-size schedule are chosen. You'll see PGD referenced often in later, more advanced robustness literature.

---

## 6. Choosing the Step Size Alpha

There's a practical tradeoff in picking `alpha`:

- **Too large** (e.g., `alpha = epsilon`, same as plain FGSM in one step): you lose the benefit of iteration entirely -- you're back to trusting one big linear step.
- **Too small** (e.g., `alpha = epsilon / 1000`): each step is very safe and accurate, but you need a huge number of iterations to actually use up the full `epsilon` budget, which costs more compute (more forward+backward passes through the model).

A common practical choice, used in the original I-FGSM paper, is:

```
alpha = epsilon / T          (T = number of iterations, e.g., T = 10)

or a small fixed value like alpha = 1/255 for 8-bit images,
run for enough iterations to traverse the full epsilon budget.
```

---

## 7. Worked Numeric Example: Walking Downhill (Uphill) in Small Steps

Let's reuse the exact logistic regression model from the FGSM worked example, so you can directly compare the one-shot result to the iterative result.

### The Model (same as before)

```
z = w1*x1 + w2*x2 + b,   w1 = 2.0, w2 = -1.5, b = 0.5
prediction = sigmoid(z)
```

### The Input (same as before)

```
x = [1.0, 0.5],  y_true = 1
Total budget: epsilon = 0.3
Per-step size: alpha = 0.1
Number of iterations: T = 3   (note: 3 * 0.1 = 0.3 = epsilon, using the full budget)
```

### Iteration 1

```
Current point: x_adv_0 = [1.0, 0.5]

z_0 = 2.0*1.0 - 1.5*0.5 + 0.5 = 2.0 - 0.75 + 0.5 = 1.75
pred_0 = sigmoid(1.75) ~= 0.852

dL/dz = pred_0 - y_true = 0.852 - 1 = -0.148
grad_0 = dL/dz * [w1, w2] = -0.148 * [2.0, -1.5] = [-0.296, 0.222]
sign(grad_0) = [-1, +1]

step_1 = alpha * sign(grad_0) = 0.1 * [-1, +1] = [-0.1, +0.1]
x_adv_1 = x_adv_0 + step_1 = [1.0 - 0.1, 0.5 + 0.1] = [0.9, 0.6]

Project: is [0.9, 0.6] within [x - eps, x + eps] = [[0.7,1.3],[0.2,0.8]]? Yes, no clipping needed.
```

(This matches the plain-FGSM result exactly, as expected -- the first iteration of I-FGSM, using `epsilon = 0.1` worth of step, is identical to a single-shot FGSM step of the same size.)

### Iteration 2 -- Recompute the Gradient at the NEW Point

```
Current point: x_adv_1 = [0.9, 0.6]

z_1 = 2.0*0.9 - 1.5*0.6 + 0.5 = 1.8 - 0.9 + 0.5 = 1.4
pred_1 = sigmoid(1.4) ~= 0.802

dL/dz = 0.802 - 1 = -0.198
grad_1 = -0.198 * [2.0, -1.5] = [-0.396, 0.297]
sign(grad_1) = [-1, +1]     (SAME sign as before, in this simple linear model)

step_2 = 0.1 * [-1, +1] = [-0.1, +0.1]
x_adv_2 = x_adv_1 + step_2 = [0.9 - 0.1, 0.6 + 0.1] = [0.8, 0.7]

Project: within [[0.7,1.3],[0.2,0.8]]? Yes.
```

### Iteration 3 -- Recompute Again

```
Current point: x_adv_2 = [0.8, 0.7]

z_2 = 2.0*0.8 - 1.5*0.7 + 0.5 = 1.6 - 1.05 + 0.5 = 1.05
pred_2 = sigmoid(1.05) ~= 0.741

dL/dz = 0.741 - 1 = -0.259
grad_2 = -0.259 * [2.0, -1.5] = [-0.518, 0.389]
sign(grad_2) = [-1, +1]     (still the same sign)

step_3 = 0.1 * [-1, +1] = [-0.1, +0.1]
x_adv_3 = x_adv_2 + step_3 = [0.8 - 0.1, 0.7 + 0.1] = [0.7, 0.8]

Project: within [[0.7,1.3],[0.2,0.8]]? Exactly at the boundary (0.7 and 0.8) -- allowed.
```

### Final Result After 3 Iterations

```
z_3 = 2.0*0.7 - 1.5*0.8 + 0.5 = 1.4 - 1.2 + 0.5 = 0.7
pred_3 = sigmoid(0.7) ~= 0.668
```

### Side-by-Side Comparison

```
                          Confidence in "class 1" (true label)
Original input:          0.852
Plain FGSM (1 x 0.1 step): 0.802     (from the previous file, eps=0.1)
Plain FGSM (1 x 0.3 step): ~0.668    (same total budget, ONE big jump -- see below)
I-FGSM (3 x 0.1 steps):    0.668     (same total budget, THREE small jumps)
```

Interesting and worth noting explicitly: in this particular *simple, purely linear* toy model, the one-shot `epsilon = 0.3` FGSM result and the 3-step I-FGSM result land on the exact same final point, `[0.7, 0.8]`, because a purely linear model's gradient direction never actually changes as you move (the "curve" is really a perfectly flat plane everywhere, so re-linearizing changes nothing). **This is specifically a property of this toy linear example, not a general result.** For real, non-linear models (deep neural networks with many stacked non-linear layers), the gradient direction genuinely changes as you move through input space, so I-FGSM's step-by-step recalculation lets it find a path that a single big FGSM jump would miss or overshoot -- which is exactly why I-FGSM is empirically stronger than FGSM in practice, even though our simplified linear example can't fully demonstrate that gap.

---

## 8. Pseudocode

```python
import numpy as np

def i_fgsm_attack(model, loss_fn, x, y_true, epsilon, alpha, num_iters):
    """
    model:      a function x -> prediction
    loss_fn:    a function (prediction, label) -> scalar loss
    x:          original input, numpy array
    y_true:     true label
    epsilon:    TOTAL L-infinity budget (max allowed total perturbation)
    alpha:      per-step size (alpha < epsilon)
    num_iters:  number of iterations, T
    """
    x_adv = x.copy()

    for t in range(num_iters):
        x_adv.requires_grad = True

        prediction = model(x_adv)
        loss = loss_fn(prediction, y_true)

        # Recompute the gradient at the CURRENT point every iteration --
        # this is the key difference from plain FGSM
        grad = compute_gradient(loss, wrt=x_adv)

        # Small step in the sign direction
        x_adv = x_adv + alpha * np.sign(grad)

        # Project back into the epsilon-ball around the ORIGINAL input x
        x_adv = np.clip(x_adv, x - epsilon, x + epsilon)

        # Also clip to the valid input range (e.g., pixel values)
        x_adv = np.clip(x_adv, 0.0, 1.0)

        # Optional early-exit: stop as soon as the attack succeeds
        if model(x_adv) != y_true:
            break

    return x_adv
```

---

## 9. Visualizing the Iterative Path

```
      One big FGSM jump                     I-FGSM: several small,
      (Section 7 of fgsm.md):               re-checked steps:

          x (start)                             x (start)
           *                                     *
            \                                     \
             \  ONE big leap,                      * step 1 (recompute here)
              \ epsilon = 0.3                        \
               \                                       * step 2 (recompute here)
                v                                        \
                 * x_adv                                   * step 3 (recompute here)
                                                              \
      Trusts the ORIGINAL                                     v
      gradient for the ENTIRE                                  * x_adv (final)
      distance traveled.
                                            Trusts the gradient only for
                                            each SHORT hop -- re-measures
                                            direction after every step.
```

Each small hop in the right-hand diagram stays inside the "trustworthy" short-distance region from the local linearity file (Section 8, "Why Small Perturbation Matters"), even though the overall path can travel just as far in total.

---

## 10. Why Iteration Increases Strength but Reduces Transferability

This is one of the most important, and slightly counter-intuitive, practical results in this area of adversarial ML research.

**Why I-FGSM is stronger against its intended target**: because it recomputes the gradient at every step, it can carefully navigate around the curvature and quirks of *that specific model's* loss landscape, finding a more precisely-aimed, efficient path to a misclassification than a single straight-line guess could.

**Why I-FGSM transfers worse to *other* models**: that same precision is a double-edged sword. I-FGSM's path becomes closely tailored to the exact bumps and curves of the *specific* model it was computed against -- it can end up in a narrow, model-specific "pocket" of adversarial input space that exploits quirks unique to that one model's decision boundary. A *different* model, even one trained on similar data, is unlikely to have the exact same quirks in the exact same place, so the perturbation is less likely to fool it. Plain FGSM's single, coarse, "genuinely large-scale" step, by contrast, tends to exploit broader, shared statistical directions in the data that many different models pick up on similarly -- which is exactly why it transfers better (as noted in the previous file's comparison table).

```
   FGSM:      broad, coarse push       -->  exploits SHARED vulnerable
              (one big step)                directions  -->  transfers well

   I-FGSM:    narrow, precise path     -->  exploits MODEL-SPECIFIC
              (many small, re-aimed         quirks  -->  transfers poorly,
              steps)                        but very strong on the exact
                                            target it was computed against
```

---

## 11. Comparison Table: FGSM vs. I-FGSM

| Aspect | FGSM | I-FGSM |
|---|---|---|
| **Number of steps** | 1 | Many (T, a chosen hyperparameter) |
| **Gradient recomputed?** | Once, at the original input | Every iteration, at the current adversarial point |
| **Projection/clipping needed?** | Only a final range clip (e.g., [0,1]) | Yes -- must clip back into the epsilon-ball after every step, plus the range clip |
| **Compute cost** | Very low (1 forward + 1 backward pass) | Higher (T forward + T backward passes) |
| **Attack strength (vs. its exact target)** | Weaker | Stronger |
| **Transferability to other models** | Better | Worse |
| **Norm** | L-infinity | L-infinity (same budget epsilon, spent gradually) |
| **Targeted variant available?** | Yes (previous file) | Yes -- same sign-flip and label-swap rule applies per-step |

---

## 12. Security Angle

- **If you only need to test whether a model is "trivially" robust, start with FGSM; if you need to actually demonstrate a working bypass against one specific, known target model, escalate to I-FGSM.** The choice mirrors a real reconnaissance-vs-exploitation tradeoff: a quick, cheap check first, followed by a more expensive, precisely-aimed attack once you've confirmed a target is worth the investment.
- **I-FGSM's weaker transferability has a direct operational implication: if you're attacking a black-box production model and only have gradient access to a substitute/shadow model, plain FGSM (or an ensemble of several substitute models) is often a better choice than I-FGSM, precisely because I-FGSM tends to overfit its perturbation to the exact quirks of the model it was computed against.** This is a genuinely important practical decision point in real red-team engagements against ML systems, not just an academic footnote.
- **The projection step itself is a place where implementation bugs create false robustness claims.** A common mistake (in both attacker tooling and defensive robustness benchmarks) is projecting back into the epsilon-ball incorrectly -- e.g., clipping relative to the wrong reference point, or forgetting the valid-data-range clip -- silently producing weaker attacks than the theoretical epsilon budget would allow, giving a false sense that a model is "more robust" than it actually is. Always sanity-check attack implementations against a known worked example (like Section 7 above) before trusting benchmark numbers.

---

## 13. Key Takeaways

- I-FGSM takes **many small FGSM-style steps** of size `alpha` (with `alpha < epsilon`), **recomputing the gradient at the current point every time**, rather than trusting one single large step.
- After every step, the **cumulative** perturbation is **projected back into the epsilon-ball** (clipped to `[x - epsilon, x + epsilon]`) so the total change never exceeds the original budget.
- The formula is `x_adv_(t+1) = clip_{x, epsilon}(x_adv_t + alpha * sign(gradient_x(L(model(x_adv_t), y_true))))`, repeated for `T` iterations.
- The worked example showed I-FGSM's 3-step path (in a simple linear toy model) matching a single large FGSM jump exactly -- but in real, non-linear deep networks, re-linearizing at each step lets I-FGSM find paths a single jump would miss, making it **empirically stronger** against its specific target.
- This extra strength comes at a cost: I-FGSM perturbations tend to be **more model-specific and transfer worse** to other models than the coarser, broader perturbations produced by plain FGSM.
- I-FGSM is the direct conceptual ancestor of the more advanced **PGD (Projected Gradient Descent)** attack referenced throughout later adversarial robustness literature.

---

*Next up: DeepFool -- an algorithm that abandons the fixed-epsilon idea entirely, instead iteratively searching for the smallest possible perturbation that reaches the nearest decision boundary, using a fresh linear approximation of the classifier at every step.*
