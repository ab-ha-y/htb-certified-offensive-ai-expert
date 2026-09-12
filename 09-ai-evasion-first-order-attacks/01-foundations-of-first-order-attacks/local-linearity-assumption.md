# The Local Linearity Assumption

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - First-Order Attacks | Section: Foundations of First-Order Attacks

---

## Table of Contents

1. [Why This Section Comes Before Any Attack](#1-why-this-section-comes-before-any-attack)
2. [First, What Is a Loss Function?](#2-first-what-is-a-loss-function)
3. [Next, What Is a Gradient?](#3-next-what-is-a-gradient)
4. [Putting It Together: The Loss Landscape](#4-putting-it-together-the-loss-landscape)
5. [The Local Linearity Assumption, Explained](#5-the-local-linearity-assumption-explained)
6. [The Math: A First-Order Taylor Approximation](#6-the-math-a-first-order-taylor-approximation)
7. [Worked Numeric Example: Approximating a Curve with a Line](#7-worked-numeric-example-approximating-a-curve-with-a-line)
8. [Why "Small Perturbation" Matters for the Assumption to Hold](#8-why-small-perturbation-matters-for-the-assumption-to-hold)
9. [What Happens When the Assumption Breaks Down](#9-what-happens-when-the-assumption-breaks-down)
10. [Security Angle](#10-security-angle)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. Why This Section Comes Before Any Attack

Every attack in this module -- FGSM, Targeted FGSM, I-FGSM, and DeepFool -- relies on one shared trick: instead of solving the hard problem "find the input that fools this complicated model," they solve the easy problem "find the input that fools a *simplified, straight-line version* of this model, valid only in a tiny neighborhood around my starting point."

That simplification is called the **local linearity assumption**. Before we can explain why it's reasonable, we need two building blocks that are used everywhere in machine learning: the **loss function** and the **gradient**. If you already know these cold, skim Sections 2-3 and jump to Section 5.

---

## 2. First, What Is a Loss Function?

A **loss function** is a formula that takes the model's prediction and the true answer, and outputs a single number saying "how wrong was this prediction?" A high number means "very wrong." A low number (ideally zero) means "correct."

### The Analogy

Imagine you're playing a game of "guess the number" where the answer is secretly 50.

- You guess 48. The "loss" (distance from the truth) is small: `|50 - 48| = 2`.
- You guess 10. The loss is large: `|50 - 10| = 40`.
- You guess 50. The loss is zero -- you nailed it.

A loss function is just a formal, mathematically convenient version of "distance from correct," designed so that:
1. It's always zero or positive (you can't be "negatively wrong").
2. Bigger mistakes produce bigger loss values.
3. It's smooth enough that calculus tools (like gradients, see Section 3) can be used to figure out *which direction* would reduce the mistake.

### A Concrete Numeric Example

Say we have a tiny classifier that outputs a "confidence score" for the label "cat," ranging from 0 (definitely not a cat) to 1 (definitely a cat). One very common loss for this setup is the **squared error loss**:

```
loss = (true_label - predicted_score)^2
```

If the image really is a cat (`true_label = 1`) and the model predicts `0.9`:

```
loss = (1 - 0.9)^2 = (0.1)^2 = 0.01     <-- small loss, model is close to correct
```

If the model instead predicts `0.2` for a true cat image:

```
loss = (1 - 0.2)^2 = (0.8)^2 = 0.64     <-- large loss, model is very wrong
```

### The Attacker's Twist

During normal training, engineers adjust the model's **parameters** to make the loss *smaller* (the model gets better at its job). During an evasion attack, the attacker does the opposite: they hold the model's parameters fixed and instead adjust the **input** to make the loss *larger* -- i.e., they deliberately push the model toward being wrong. Same formula, opposite goal, different thing being changed.

---

## 3. Next, What Is a Gradient?

A **gradient** tells you, for a small nudge to the input, which direction increases the loss the fastest and by how much, per unit of nudge. It generalizes the idea of "slope" from a single number (like the slope of a hill) to many numbers at once (one slope per input feature).

### The Analogy: Slope of a Hill

If you're standing on a hillside, the **slope** at your exact location tells you: "if you take one step forward, do you go up or down, and how steeply?" A slope of `+3` means "steep uphill" if you step forward; a slope of `-3` means "steep downhill." The slope is a *local* property -- it's only accurate for a tiny step from where you're currently standing, not for a mile-long hike.

A gradient is exactly this "slope" idea, but computed **separately for every input feature** and bundled into one vector. If your input has 3 features, the gradient has 3 numbers -- one "slope" telling you how the loss changes if you nudge feature 1, another for feature 2, another for feature 3.

### A Concrete Numeric Example

Suppose our toy model computes a score using a simple linear formula with two input features `x1` and `x2`:

```
score = 2*x1 + 3*x2
loss  = (target - score)^2
```

Say the current input is `x1 = 1, x2 = 1`, and the target we're trying to move *away* from (for an attacker) or *toward* (for training) is `target = 10`.

```
score = 2*(1) + 3*(1) = 5
loss  = (10 - 5)^2 = 25
```

The gradient of the loss with respect to each input feature answers: "if I nudge `x1` up slightly, does the loss go up or down, and by roughly how much per unit nudge?" Using basic calculus (don't worry about deriving this by hand -- frameworks like PyTorch/TensorFlow compute this automatically via a process called **backpropagation**):

```
d(loss)/d(x1) = -2 * (target - score) * 2 = -2 * 5 * 2 = -20
d(loss)/d(x2) = -2 * (target - score) * 3 = -2 * 5 * 3 = -30

gradient = [-20, -30]
```

**Reading this result**: the gradient says "increasing `x1` by a tiny amount *decreases* the loss by about 20 units per unit of `x1` (since the number is negative, moving in the positive x1 direction reduces loss)." Since the attacker wants to *increase* loss (make the model wrong), they would move `x1` and `x2` in the direction that makes the loss bigger, which is the **opposite** sign of this gradient in this particular example (because the gradient here points toward decreasing loss). In general, moving *with* the gradient decreases loss fastest (this is the basis of "gradient descent," used in training) and moving in the *sign-flipped, opposite* direction of that same gradient increases loss fastest (this is the basis of evasion attacks, sometimes called "gradient ascent" on the loss).

### The Key Property to Remember

```
GRADIENT = the direction of steepest INCREASE in the loss,
           for an infinitesimally small step.

  - Training:  step OPPOSITE the gradient  -->  loss goes DOWN  -->  model improves
  - Evasion:   step ALONG the gradient     -->  loss goes UP    -->  model gets fooled
```

This single idea -- "the gradient points toward increasing loss, so an attacker steps along it" -- is the mathematical engine behind FGSM, Targeted FGSM, I-FGSM, and (in a more refined form) DeepFool. Every remaining file in this module is really just a variation on "how exactly do we use the gradient to build the perturbation?"

---

## 4. Putting It Together: The Loss Landscape

If we imagine every possible input as a point on a map, and the loss value at each point as its "altitude," we get a **loss landscape** -- a bumpy, hilly surface (in reality, one with thousands or millions of dimensions, but we can only draw 2 or 3 at a time).

```
                        LOSS LANDSCAPE (simplified to 1 input feature)

     loss
      |
      |                                    ,-.
      |                                   /   \
      |                        ,-.       /     \
      |                       /   \     /       \
      |          ,-.         /     \   /         \
      |         /   \       /       \-'           \
      |        /     '-----'                        \
      |  -----'                                       '-----
      +---------------------------------------------------------> input value
                    ^
              our current input x
              (sitting partway up a slope)
```

The model's prediction, and therefore the loss, changes as you move across this landscape. A **local** gradient only tells you the slope exactly at your current point -- it says nothing about what's happening far away (the landscape could flatten out, curve back down, or have a cliff just past where the arrow points).

---

## 5. The Local Linearity Assumption, Explained

The **local linearity assumption** says: *for a sufficiently small step, the bumpy loss landscape looks approximately like a flat, tilted plane (or, in 1D, a straight line) matching the slope at your current position.*

### The Analogy

If you zoom into a curvy road on a map far enough, any short segment of it looks almost perfectly straight, even though the whole road curves a lot overall. A car's GPS uses this trick constantly: over the next 10 meters, "go straight" is a perfectly good approximation, even on a winding mountain road, because 10 meters is small compared to how sharply the road curves.

```
    ZOOMED OUT (curve is obvious)         ZOOMED IN (looks like a line)

         ___                                    /
        /   \                                  /
       /     \___                             /
      /                                      /
     /                                      /
    curvy road, big picture           tiny segment of the same road
```

Machine learning models -- especially deep neural networks -- have extremely complicated, bumpy loss landscapes. But near any *single specific point* (our starting input `x`), if we only move a tiny distance `epsilon`, the landscape behaves almost exactly like its **tangent plane** at that point -- a flat approximation defined entirely by the gradient we computed in Section 3.

This is the entire justification for why attacks like FGSM can get away with computing the gradient *once* and using it to build an effective perturbation, instead of solving some enormously expensive global optimization problem.

---

## 6. The Math: A First-Order Taylor Approximation

The formal name for "approximate a curvy function with a straight line based on its slope at one point" is a **first-order Taylor approximation** (named after mathematician Brook Taylor). "First-order" means we only use the *first* derivative (the slope/gradient) and ignore how the slope itself is changing (that would require a *second*-order term, which is more complex and expensive to compute).

For a loss function `L` that depends on the input `x`, and a small perturbation `delta`, the Taylor approximation says:

```
L(x + delta)  ~=  L(x)  +  gradient(L, x) . delta

  L(x + delta)         = the loss AFTER perturbing
  L(x)                 = the loss BEFORE perturbing (a known baseline)
  gradient(L, x)        = the gradient of the loss at the original point x
  gradient(L, x) . delta = the "dot product" -- multiply matching entries
                           of the gradient and the perturbation, then sum them
```

The dot product `gradient . delta` is exactly the "flat, tilted plane" approximation described above: it tells you how much the loss changes for a given perturbation `delta`, *assuming* the local linearity assumption holds.

An attacker's whole goal, then, becomes: *choose `delta` (subject to a norm-budget constraint from the previous section) to make `gradient(L, x) . delta` as large as possible.* This is a much simpler problem than optimizing the real, complicated `L(x + delta)` directly -- and it turns out to have a clean closed-form solution, which is exactly what FGSM computes (see the next file in this module).

---

## 7. Worked Numeric Example: Approximating a Curve with a Line

Let's use one concrete number to see the approximation in action, then check how good it actually is.

Suppose our loss function (as a function of a single input feature `x`) happens to be:

```
L(x) = x^2
```

(A parabola -- simple enough to compute exactly, so we can check our linear approximation against the real answer.)

Say our current input is `x = 3`. Let's compute:

**Step 1: The true loss at x = 3**

```
L(3) = 3^2 = 9
```

**Step 2: The gradient (slope) at x = 3**

For `L(x) = x^2`, the derivative (gradient, in 1D) is `dL/dx = 2x`. At `x = 3`:

```
gradient = 2 * 3 = 6
```

**Step 3: Use the linear (first-order Taylor) approximation to predict the loss after a small perturbation, say delta = 0.1**

```
L(x + delta)  ~=  L(x)  +  gradient * delta
              ~=  9  +  6 * 0.1
              ~=  9  +  0.6
              ~=  9.6   (approximation)
```

**Step 4: Compute the actual loss at x + delta = 3.1, and compare**

```
L(3.1) = 3.1^2 = 9.61   (actual)
```

Our linear approximation said `9.6`; the real answer is `9.61`. That's an error of only `0.01` -- extremely close!

**Step 5: Now try a much bigger perturbation, delta = 2, to see the approximation degrade**

```
Approximation:  L(3+2) ~= 9 + 6*2 = 9 + 12 = 21
Actual:         L(5) = 5^2 = 25

Error = |25 - 21| = 4   (much worse!)
```

### What This Demonstrates

```
   delta = 0.1   -->   approximation error = 0.01   (tiny, safe to use)
   delta = 2.0   -->   approximation error = 4.00    (large, unreliable)
```

The bigger the step, the worse the straight-line approximation gets, because the real function (a parabola here; an enormously more complex, bumpy surface for a real neural network) curves away from the tangent line the farther you travel. This single numeric pattern -- small steps are trustworthy, large steps are not -- is the entire mathematical justification for why adversarial attacks constrain the perturbation to a small norm-ball (Section 1 of this module) and why iterative attacks like I-FGSM take *many small steps* rather than one giant leap (covered later in this module).

---

## 8. Why "Small Perturbation" Matters for the Assumption to Hold

```
    SMALL STEP                              LARGE STEP
    ==========                              ==========

    L(x)                                    L(x)
      \                                       \
       \___ true curve                         \___ true curve
        \  \_ tangent line (close!)              \      \
         \  \                                     \      \_ tangent line
          \__\                                      \        (way off!)
           x  x+delta                                x         x+delta

    Tangent line and true curve            Tangent line and true curve
    nearly overlap for a short              diverge sharply over a
    distance --> approximation              longer distance --> approximation
    is trustworthy                          becomes unreliable
```

This is precisely *why* norm constraints (from the previous file) exist in the first place: the attack budget `epsilon` isn't just about staying imperceptible to a human -- it is also what keeps the attacker's own linear-approximation math valid. An attack that ignores the budget and takes a huge step is, mathematically, "flying blind" -- the gradient direction it computed at the *starting* point may no longer point toward increased loss once it has traveled that far.

---

## 9. What Happens When the Assumption Breaks Down

If the perturbation is too large, or the loss landscape is unusually "wiggly" near the input (which happens more often in deep, highly non-linear networks), the local linear approximation can become misleading in a specific, exploitable way:

- The gradient computed at the *original* point `x` might suggest a direction that, once you actually take a large step in it, no longer increases the loss as expected -- you may even overshoot past the nearest decision boundary and land back on the *correct* side, or land in a totally different, unhelpful region of input space.
- This is exactly the motivation for **iterative** attacks (I-FGSM, and more sophisticated methods like PGD, covered later): take a small step, then **recompute the gradient at the new point**, then take another small step. Each individual step trusts the local linear approximation (because each step is small), while the overall path can still travel a large total distance.
- It is also the core idea behind **DeepFool** (the final file in this module): rather than assuming one global epsilon budget and hoping the linear approximation holds for the whole jump, DeepFool repeatedly re-linearizes the model at each new point and asks, "given *this* linear approximation, what is the smallest step to the nearest boundary?" -- explicitly working around the fact that a single big linear step is unreliable.

---

## 10. Security Angle

- **Gradient access is the single most valuable piece of information an attacker can have.** If you can compute (or estimate) the gradient of a model's loss with respect to its input, you have a fast, principled way to find inputs that break it. This is the core reason **white-box attacks** (attacker has full model access, including gradients) are dramatically more efficient than **black-box attacks** (attacker only sees inputs/outputs and must estimate the gradient indirectly, e.g., by querying the model many times and finite-differencing, or by training a substitute/shadow model to steal an approximate gradient).
- **Any defense that restricts gradient access (e.g., adding noise to outputs, limiting query rates, obfuscating the model) is really trying to break the attacker's ability to trust their own local linear approximation.** Understanding *why* the linear approximation works tells you exactly what a defender is trying to disrupt, and why techniques like "gradient masking" (making gradients uninformative near the input, without actually making the model more robust) are a common but often superficial defense that transfer attacks can bypass.
- **Every attack budget you will see referenced in research (`epsilon = 8/255`, `epsilon = 0.03`, etc.) exists because of this section.** It is not an arbitrary "stealth" number -- it is chosen to be small enough that the first-order Taylor approximation remains trustworthy while still being large enough to actually flip the model's decision.

---

## 11. Key Takeaways

- A **loss function** turns "how wrong was the prediction?" into a single number; bigger mistakes produce bigger loss.
- A **gradient** is a vector of "slopes" -- one per input feature -- that says which direction increases the loss fastest, for an infinitesimally small step.
- Moving **opposite** the gradient decreases loss (used in training); moving **along** the gradient increases loss (used in evasion attacks).
- The **local linearity assumption** says the loss landscape looks approximately like a flat, tilted plane near any single point, as long as the step taken is small -- this is a **first-order Taylor approximation**.
- The worked example showed the approximation error growing from `0.01` (tiny step) to `4.0` (large step) on the exact same function -- concretely demonstrating why "small perturbation" is not just a stealth requirement but a *mathematical validity* requirement.
- When perturbations get too large, the linear approximation breaks down, which motivates **iterative** attacks (re-linearize at each new point) rather than one giant single-shot jump.

---

*Next up: High-Dimensional Effects -- why perturbations that are individually tiny in every single feature can still add up to a large, decision-flipping effect once you have thousands or millions of features to play with.*
