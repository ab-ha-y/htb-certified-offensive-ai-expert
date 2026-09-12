# DeepFool: Finding the Nearest Decision Boundary

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - First-Order Attacks | Section: DeepFool

---

## Table of Contents

1. [A Different Question Entirely](#1-a-different-question-entirely)
2. [The Plain-English Idea](#2-the-plain-english-idea)
3. [Refresher: Decision Boundaries](#3-refresher-decision-boundaries)
4. [The Simplest Case: A Binary Linear Classifier](#4-the-simplest-case-a-binary-linear-classifier)
5. [Distance to a Hyperplane -- The Key Formula](#5-distance-to-a-hyperplane-the-key-formula)
6. [Worked Numeric Example: Binary Linear Case](#6-worked-numeric-example-binary-linear-case)
7. [Why One Step Usually Isn't Enough for Non-Linear Models](#7-why-one-step-usually-isnt-enough-for-non-linear-models)
8. [The Multi-Class Case: Finding the NEAREST Boundary](#8-the-multi-class-case-finding-the-nearest-boundary)
9. [The Full Iterative Algorithm](#9-the-full-iterative-algorithm)
10. [Worked Numeric Example: Multi-Class, Two Iterations](#10-worked-numeric-example-multi-class-two-iterations)
11. [Pseudocode](#11-pseudocode)
12. [Visualizing DeepFool vs. FGSM](#12-visualizing-deepfool-vs-fgsm)
13. [Comparison Table: DeepFool vs. FGSM Family](#13-comparison-table-deepfool-vs-fgsm-family)
14. [Security Angle](#14-security-angle)
15. [Key Takeaways](#15-key-takeaways)

---

## 1. A Different Question Entirely

Every attack so far in this module (FGSM, Targeted FGSM, I-FGSM) starts from the same question: *"Given a fixed perturbation budget `epsilon`, how do I use it as effectively as possible?"* The attacker picks `epsilon` in advance, and the algorithm tries to make the most of it.

DeepFool, introduced by Seyed-Mohsen Moosavi-Dezfooli, Alhussein Fawzi, and Pascal Frossard in *"DeepFool: a simple and accurate method to fool deep neural networks"* (2016), flips this question around entirely:

> "I don't want to fix a budget in advance. I want to find the **smallest possible perturbation**, whatever that turns out to be, that is just barely enough to flip the model's decision."

This is a genuinely different goal. Instead of "spend my budget as effectively as possible," it's "find out what the true minimum cost of fooling this model actually is." That number -- the minimal perturbation size DeepFool finds -- is itself a useful, standalone measurement of how "fragile" a model's decision is at that specific point, often reported as a **robustness metric** in adversarial ML research.

---

## 2. The Plain-English Idea

### The Analogy

Recall the decision-boundary diagrams from the previous files, where an input sits inside a region bordered by other classes' regions. FGSM is like standing in a room and taking one confident, full-strength step toward *a* wall (any wall, or a specific chosen wall for the targeted variant), hoping that step happens to be big enough to walk through it.

DeepFool is like instead first looking around the room, measuring the exact distance to **every** wall, walking directly toward the **closest** one, and stopping the instant you touch it -- using the least possible effort to leave the room. And since real rooms (real model decision regions) usually have curved walls rather than perfectly flat ones, you look around and re-measure again after each small move, adjusting your path as you get closer to make sure you're still heading toward the true nearest exit.

---

## 3. Refresher: Decision Boundaries

A **decision boundary** is the set of points where a model is exactly torn between two (or more) classes -- right on the fence, 50/50. On one side of the boundary, the model predicts class A; on the other side, class B. Every attack in this module is fundamentally about finding a way to cross one of these boundaries.

```
        Class A region          |          Class B region
                                 |
              x (input,          |
               classified A)     |
                *                |
                                  |  <-- the decision boundary:
                                  |      model is exactly 50/50
                                  |      undecided right here
```

For a **linear** classifier (like the logistic regression model used in earlier worked examples), the decision boundary is a perfectly flat line (in 2D), plane (in 3D), or more generally a **hyperplane** (in higher dimensions). For a real deep neural network, the boundary is generally a complicated, curved surface -- but, thanks to the local linearity assumption from the Foundations section, it looks approximately flat if you zoom in closely enough to any single point on it.

---

## 4. The Simplest Case: A Binary Linear Classifier

DeepFool's core idea is easiest to see with a linear binary classifier, where the decision boundary really is a perfectly flat hyperplane (not just "approximately flat" -- exactly flat, everywhere). We'll build up the intuition here first, then generalize.

Consider a linear classifier:

```
f(x) = w . x + b        (a "score"; predicts class 1 if f(x) > 0, class 0 if f(x) < 0)
```

The decision boundary is exactly the set of points where `f(x) = 0`. Given a starting point `x0` (correctly classified, so `f(x0) > 0`, say), DeepFool asks: *what is the shortest path from `x0` to the hyperplane `f(x) = 0`?*

Geometrically, the shortest path from a point to a hyperplane is always a **straight line perpendicular to the hyperplane** -- and the direction perpendicular to the hyperplane `f(x) = w.x + b = 0` is exactly the direction of the weight vector `w` (this is a standard, provable fact from linear algebra: the gradient of a hyperplane's defining function always points perpendicular to that hyperplane).

---

## 5. Distance to a Hyperplane -- The Key Formula

The exact (not approximate) formula for the shortest distance from a point `x0` to the hyperplane `f(x) = w.x + b = 0` is:

```
distance = |f(x0)| / ||w||_2

  |f(x0)|   = the absolute value of the score at x0 (how far from zero the score currently is)
  ||w||_2   = the L2 norm (Euclidean length) of the weight vector -- this is also EXACTLY
              the gradient of f(x) with respect to x, since gradient_x(w.x + b) = w
```

And the minimal perturbation that reaches the boundary, `delta`, is that distance, traveled in the direction of steepest descent toward the boundary (i.e., opposite the gradient if `f(x0) > 0` and we want to decrease the score down to exactly zero):

```
delta = - ( f(x0) / ||w||_2^2 ) * w
```

**Where this formula comes from, intuitively**: we want to move exactly far enough, in the direction of `w` (perpendicular to the boundary), that the score `f(x0 + delta)` becomes exactly `0`. Since `f(x0 + delta) = f(x0) + w.delta` (linear functions add exactly, with no approximation error at all -- unlike the general local linearity *assumption* used for non-linear models), setting `delta = -c*w` for some scalar `c` and solving `f(x0) + w.(-c*w) = 0` gives `c = f(x0) / ||w||_2^2`, which is exactly the formula above.

---

## 6. Worked Numeric Example: Binary Linear Case

Let's reuse the same logistic-regression-style linear scoring function from the FGSM examples, but this time compute the *exact minimal* perturbation, rather than a fixed-epsilon step.

### The Model

```
f(x) = w1*x1 + w2*x2 + b,   w = [2.0, -1.5],  b = 0.5
(decision boundary: f(x) = 0)
```

### The Input

```
x0 = [1.0, 0.5]
```

### Step 1: Compute the Current Score

```
f(x0) = 2.0*1.0 + (-1.5)*0.5 + 0.5 = 2.0 - 0.75 + 0.5 = 1.75
```

The score is `1.75`, comfortably on the positive ("class 1") side.

### Step 2: Compute the Weight Vector's L2 Norm

```
||w||_2 = sqrt(2.0^2 + (-1.5)^2) = sqrt(4.0 + 2.25) = sqrt(6.25) = 2.5
```

### Step 3: Compute the Exact Distance to the Boundary

```
distance = |f(x0)| / ||w||_2 = 1.75 / 2.5 = 0.7
```

This says: the input is exactly `0.7` units (in ordinary Euclidean/L2 distance) away from the decision boundary. Not "approximately" -- this is exact, because the model is exactly linear.

### Step 4: Compute the Minimal Perturbation Vector

```
delta = - ( f(x0) / ||w||_2^2 ) * w
      = - ( 1.75 / 6.25 ) * [2.0, -1.5]
      = - 0.28 * [2.0, -1.5]
      = [-0.56, 0.42]
```

### Step 5: Apply and Verify

```
x_adv = x0 + delta = [1.0 - 0.56, 0.5 + 0.42] = [0.44, 0.92]

f(x_adv) = 2.0*0.44 + (-1.5)*0.92 + 0.5
         = 0.88 - 1.38 + 0.5
         = 0.00      <-- exactly zero! We landed precisely ON the boundary.
```

### Sanity Check: The L2 Norm of the Perturbation Itself

```
||delta||_2 = sqrt((-0.56)^2 + (0.42)^2) = sqrt(0.3136 + 0.1764) = sqrt(0.49) = 0.7
```

This exactly matches the `distance = 0.7` we computed in Step 3, confirming the formula is self-consistent: the perturbation's own L2 length equals the point-to-boundary distance we set out to find.

**Practical note**: landing exactly on `f(x) = 0` means the model is now perfectly 50/50 undecided -- in practice, algorithms add a tiny extra nudge (sometimes called an "overshoot" factor, e.g., multiplying `delta` by `1.02` instead of `1.0`) to make sure the perturbed point actually crosses to the other side rather than sitting exactly on the fence.

---

## 7. Why One Step Usually Isn't Enough for Non-Linear Models

The formula in Section 5 is **exact** only because the toy model `f(x) = w.x + b` is truly, exactly linear everywhere. Real neural networks are not linear -- they only look approximately linear in a small neighborhood around any single point (this is exactly the local linearity assumption from the Foundations section). This means:

1. Compute the gradient at the *current* point, and pretend the model is linear (matching that gradient) everywhere.
2. Use the exact hyperplane-distance formula from Section 5 to jump straight to where *that linear approximation's* boundary would be.
3. Because the real model isn't actually linear, this jump usually **undershoots or slightly overshoots** the real, curved boundary.
4. So: recompute the gradient at the *new* point, and repeat -- exactly the same "re-linearize and step" philosophy as I-FGSM, but now aiming precisely at the (locally-estimated) nearest boundary distance, rather than a fixed-size step.

This is why DeepFool is fundamentally an **iterative** algorithm, even though each individual step uses an exact, closed-form calculation (rather than a fixed step size like I-FGSM's `alpha`).

---

## 8. The Multi-Class Case: Finding the NEAREST Boundary

Real classification problems usually have more than two classes, which means there isn't just one decision boundary -- there's one boundary between the current (correct) class and *every other* class. DeepFool's full algorithm computes the distance to **each** of these boundaries (using the linear-approximation version of the Section 5 formula, one per class) and then steps toward whichever one is **closest**.

For a multi-class model where each class `k` has a score function `f_k(x)` (e.g., the pre-softmax logits), the "boundary" between the current predicted class `c` (the class with the highest score) and any other class `k` is where their scores become equal: `f_c(x) = f_k(x)`, or equivalently `f_k(x) - f_c(x) = 0`. The distance formula from Section 5 generalizes directly, using `(f_k - f_c)` in place of `f`, and `gradient(f_k) - gradient(f_c)` in place of `w`:

```
distance_to_class_k = | f_k(x) - f_c(x) | / || gradient(f_k)(x) - gradient(f_c)(x) ||_2
```

DeepFool computes this distance for **every other class `k`**, picks the class `k*` with the **smallest** distance (the nearest boundary), and steps toward that one specific boundary.

```
k* = argmin over all k != c of  distance_to_class_k

("argmin" just means: "the k that makes this distance smallest" --
 i.e., find the boundary you can reach most cheaply)
```

---

## 9. The Full Iterative Algorithm

Putting Sections 7 and 8 together, here is the complete DeepFool procedure:

```
1. Start at x_adv = x (the original, correctly-classified input); let c = the current predicted class.

2. Repeat until the predicted class changes (or a max iteration count is hit):

   a. For every OTHER class k (k != c), compute:
        - the score difference:    w_k = f_k(x_adv) - f_c(x_adv)
        - the gradient difference: g_k = gradient(f_k)(x_adv) - gradient(f_c)(x_adv)
        - the distance to k's boundary: dist_k = |w_k| / ||g_k||_2

   b. Find k* = the class with the SMALLEST dist_k (the nearest boundary).

   c. Compute the minimal step toward that ONE nearest boundary
      (using the Section 5 formula, applied to w_k* and g_k*):
         delta_step = - (w_k* / ||g_k*||_2^2) * g_k*

   d. Move to the new point:  x_adv = x_adv + delta_step   (often scaled
      by a small "overshoot" factor > 1, e.g., 1.02, to ensure the boundary
      is actually crossed rather than just barely touched)

   e. Re-check the model's prediction on the new x_adv, and re-set c
      if needed for the next loop.

3. Return the final x_adv, and (optionally) the total perturbation ||x_adv - x||
   as a measurement of the model's local robustness at this input.
```

---

## 10. Worked Numeric Example: Multi-Class, Two Iterations

Let's use the same 3-class linear scoring model from the Targeted FGSM file, and this time find DeepFool's minimal, un-targeted perturbation -- toward whichever boundary is nearest, no target chosen in advance.

### The Model (same as Targeted FGSM's example)

```
score_A = 1.0*x1 + 0.5*x2 + 0.0
score_B = -0.5*x1 + 1.0*x2 + 0.1
score_C = 0.2*x1 - 0.3*x2 + 0.2
```

### The Input

```
x0 = [2.0, 1.0],  current prediction = A (score_A = 2.5, the highest, from the earlier worked example)
```

### Iteration 1: Compute Distance to Each Other Class's Boundary

**Distance to B's boundary:**

```
w_B = score_B - score_A = 0.1 - 2.5 = -2.4
gradient(f_B) - gradient(f_A) = [-0.5, 1.0] - [1.0, 0.5] = [-1.5, 0.5]
||g_B||_2 = sqrt((-1.5)^2 + 0.5^2) = sqrt(2.25 + 0.25) = sqrt(2.5) ~= 1.581

dist_B = |-2.4| / 1.581 ~= 1.518
```

**Distance to C's boundary:**

```
w_C = score_C - score_A = 0.3 - 2.5 = -2.2
gradient(f_C) - gradient(f_A) = [0.2, -0.3] - [1.0, 0.5] = [-0.8, -0.8]
||g_C||_2 = sqrt((-0.8)^2 + (-0.8)^2) = sqrt(0.64 + 0.64) = sqrt(1.28) ~= 1.131

dist_C = |-2.2| / 1.131 ~= 1.946
```

### Compare and Pick the Nearest

```
dist_B ~= 1.518
dist_C ~= 1.946

B is CLOSER (smaller distance) --> k* = B
```

Notice this is a genuinely useful, non-obvious result: `C`'s raw score gap from `A` (`2.2`) is actually *smaller* than `B`'s score gap (`2.4`), so if you only looked at the raw score difference (as a naive attacker might), you'd have guessed C was the "closer" target. But once you divide by each boundary's own gradient-based scaling factor (`||g||_2`), it turns out **B's boundary is actually the nearer one in true input-distance terms**. This is exactly why DeepFool does this careful per-class normalization rather than just picking the class with the smallest raw score gap.

### Compute the Step Toward B

```
delta_step = - (w_B / ||g_B||_2^2) * g_B
           = - (-2.4 / 2.5) * [-1.5, 0.5]
           = - (-0.96) * [-1.5, 0.5]
           = 0.96 * [-1.5, 0.5]
           = [-1.44, 0.48]
```

### Apply the Step

```
x_adv_1 = x0 + delta_step = [2.0 - 1.44, 1.0 + 0.48] = [0.56, 1.48]
```

### Check the New Scores

```
score_A = 1.0*0.56 + 0.5*1.48 + 0.0 = 0.56 + 0.74       = 1.30
score_B = -0.5*0.56 + 1.0*1.48 + 0.1 = -0.28 + 1.48 + 0.1 = 1.30
score_C = 0.2*0.56 - 0.3*1.48 + 0.2 = 0.112 - 0.444 + 0.2 = -0.132
```

`score_A` and `score_B` are now exactly tied at `1.30` -- we landed almost precisely on the A-vs-B decision boundary, exactly as the linear-hyperplane math predicted (any tiny residual difference here is just rounding). Since this toy model is exactly linear (not just locally linear), **one iteration was enough** to reach the true boundary exactly -- mirroring the exact behavior from the binary example in Section 6. For a real, non-linear deep network, this same procedure would need a **second iteration**: recompute the gradients at `x_adv_1`, find the (possibly now-different) nearest boundary from this new point, and take another small corrective step, because the true curved boundary would not coincide perfectly with the first iteration's flat, linear estimate of it.

### Applying the Overshoot to Actually Cross

```
delta_step_scaled = 1.02 * [-1.44, 0.48] = [-1.469, 0.490]
x_adv_final = [2.0 - 1.469, 1.0 + 0.490] = [0.531, 1.490]

score_A = 1.0*0.531 + 0.5*1.490 = 0.531 + 0.745 = 1.276
score_B = -0.5*0.531 + 1.0*1.490 + 0.1 = -0.266 + 1.490 + 0.1 = 1.324

Now score_B (1.324) > score_A (1.276)  -->  prediction has flipped to B!
```

The small `1.02` overshoot factor was exactly enough to nudge past the exact boundary and actually flip the winning class, confirming the attack succeeded with what is (up to that tiny overshoot margin) essentially the smallest possible perturbation.

---

## 11. Pseudocode

```python
import numpy as np

def deepfool_attack(model_scores, model_gradients, x, num_classes,
                      max_iters=50, overshoot=1.02):
    """
    model_scores:    function x -> array of per-class scores/logits, length num_classes
    model_gradients: function x -> array of per-class gradient vectors (one gradient
                      vector per class, each the same shape as x)
    x:               original input, numpy array
    num_classes:     number of classes
    max_iters:       safety cap on iterations
    overshoot:       small multiplier (>1) to ensure the boundary is actually crossed
    """
    x_adv = x.copy()
    scores0 = model_scores(x_adv)
    c = np.argmax(scores0)          # current predicted class
    original_class = c

    for iteration in range(max_iters):
        scores = model_scores(x_adv)
        grads = model_gradients(x_adv)

        c = np.argmax(scores)
        if c != original_class:
            break                    # already flipped -- done

        best_dist = np.inf
        best_step = None

        for k in range(num_classes):
            if k == c:
                continue
            w_k = scores[k] - scores[c]
            g_k = grads[k] - grads[c]
            norm_g_k = np.linalg.norm(g_k, ord=2)
            if norm_g_k == 0:
                continue
            dist_k = abs(w_k) / norm_g_k

            if dist_k < best_dist:
                best_dist = dist_k
                # minimal step toward THIS class's boundary
                best_step = -(w_k / (norm_g_k ** 2)) * g_k

        # Take the step toward the single nearest boundary found this round,
        # scaled slightly past the exact boundary
        x_adv = x_adv + overshoot * best_step

    return x_adv, iteration + 1, np.linalg.norm(x_adv - x, ord=2)
```

---

## 12. Visualizing DeepFool vs. FGSM

```
                    FGSM: fixed epsilon,                DeepFool: no fixed epsilon,
                    ONE direction, ONE size              iteratively finds the
                                                          NEAREST boundary, minimal size

     Class B      Class A       Class C          Class B      Class A       Class C
    +--------+------------+--------+           +--------+------------+--------+
    |        |            |        |           |        |            |        |
    |        |     * x    |        |           |        |     * x    |        |
    |        |    /(any        \   |           |        |    |        |        |
    |        |   / fixed-size    \ |           |        |    | short, |        |
    |        |  /  step, may       |           |        |    | exact  |        |
    |        | /   over/undershoot)|           |        |    | step   |        |
    |        |* x_adv              |           |        |    v        |        |
    +--------+------------+--------+           +--------+ * x_adv    +--------+
                                                                 (nearest wall,
                                                                  minimal travel)
```

FGSM's arrow length is fixed by `epsilon`, chosen before looking at the geometry at all. DeepFool's arrow length is *discovered* by the algorithm itself, and always points at whichever wall (class boundary) turns out to be geometrically nearest.

---

## 13. Comparison Table: DeepFool vs. FGSM Family

| Aspect | FGSM | I-FGSM | DeepFool |
|---|---|---|---|
| **Perturbation size decided by** | Attacker, fixed in advance (`epsilon`) | Attacker, fixed in advance (`epsilon`, spent over `T` steps) | The algorithm itself -- discovers the minimal size needed |
| **Norm typically used** | L-infinity | L-infinity | L2 |
| **Number of steps** | 1 | Many, fixed count `T` | Many, until the prediction flips (variable) |
| **Targeted or untargeted** | Untargeted (targeted variant exists) | Untargeted (targeted variant exists) | Untargeted by default (always aims at the *nearest* boundary, whichever class that is) |
| **Goal** | Maximize damage within a fixed budget | Maximize damage within a fixed budget, more precisely | Minimize the size of the perturbation needed for any misclassification |
| **Output includes a "robustness score"?** | No | No | Yes -- the final perturbation size is itself a meaningful measurement of local robustness |
| **Speed** | Fastest | Slower than FGSM | Typically slower than FGSM, comparable to or slower than I-FGSM, since it must evaluate distances to every class's boundary each iteration |
| **Typical perceptibility of result** | Depends entirely on the chosen epsilon | Depends entirely on the chosen epsilon | Tends to be the smallest possible, often more imperceptible than a similarly-successful FGSM/I-FGSM attack |

---

## 14. Security Angle

- **DeepFool's output size doubles as a robustness benchmark.** Because it finds the (approximately) minimal perturbation needed to fool a model at a given point, running DeepFool over many test inputs and averaging the resulting perturbation sizes gives a standard, widely-used metric for comparing how robust different models are -- smaller average DeepFool distance means the model's decision boundaries sit closer to real data points, i.e., it is more fragile.
- **DeepFool perturbations are often harder for simple "is this image suspiciously noisy" defenses to catch, precisely because they are engineered to be as small as possible.** A defense that flags inputs with unusually large L-infinity or L2 deviation from typical clean inputs may catch an aggressively-tuned FGSM/I-FGSM attack more easily than a DeepFool attack, which by design uses close to the theoretical minimum amount of change.
- **DeepFool requires evaluating (or approximating) the gradient for every other class at every iteration, which is more computationally and access-intensive than FGSM's single gradient.** In a black-box or rate-limited API setting, this can make DeepFool impractical without a locally-held substitute model -- another reason attackers often reach for FGSM/I-FGSM first in constrained real-world engagements, and reserve DeepFool for white-box research, benchmarking, or high-value targets where the extra query/compute cost is justified by the payoff of a minimally-detectable perturbation.
- **The "nearest boundary" framing is itself a powerful reconnaissance tool.** Even beyond crafting a working perturbation, running DeepFool against a target model tells you *which other class is geometrically closest* to any given input -- valuable intelligence about a model's confusion patterns and potential blind spots, useful for planning subsequent, more targeted attacks (e.g., the Targeted FGSM approach from an earlier file).

---

## 15. Key Takeaways

- DeepFool abandons the fixed-`epsilon` philosophy entirely, instead **iteratively searching for the smallest perturbation** that reaches the **nearest** decision boundary.
- For an exactly linear classifier, the distance from a point to a boundary hyperplane `f(x) = w.x + b = 0` has an **exact, closed-form answer**: `distance = |f(x0)| / ||w||_2`, and the minimal perturbation is `delta = -(f(x0)/||w||_2^2) * w`.
- For real, non-linear models, DeepFool applies this exact linear formula as a **local approximation** at each step, then **re-linearizes at the new point** and repeats -- the same "re-check after every small move" philosophy as I-FGSM, but aimed at minimal distance rather than a fixed step size.
- In the multi-class case, DeepFool computes the distance to **every** other class's boundary and always steps toward whichever one is genuinely **nearest** -- which, as the worked example showed, is not always the class with the smallest raw score gap; the gradient-based scaling matters.
- A small **overshoot** multiplier is applied to the final step to ensure the perturbed input actually crosses the boundary rather than landing exactly on it.
- DeepFool typically produces **smaller, harder-to-detect** perturbations than FGSM/I-FGSM (using the L2 norm rather than L-infinity), at the cost of more computation and more required access to per-class gradient information.

---

*Next up: Having covered the core single-step and iterative first-order attack algorithms (FGSM, Targeted FGSM, I-FGSM, and DeepFool), the next module builds on these foundations to cover stronger, budget-aware iterative methods (such as PGD) and the black-box, query-based and transfer-based attacks used when gradient access is not directly available.*
