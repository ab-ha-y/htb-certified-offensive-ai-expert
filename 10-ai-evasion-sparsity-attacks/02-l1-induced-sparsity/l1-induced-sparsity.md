# L1-Induced Sparsity

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - Sparsity Attacks | Section: L1-Induced Sparsity

---

## Table of Contents

1. [Recap: Why We Need a Substitute for L0](#1-recap-why-we-need-a-substitute-for-l0)
2. [What Does "Convex" Mean, in Plain English?](#2-what-does-convex-mean-in-plain-english)
3. [The L1 Norm, Defined Gently](#3-the-l1-norm-defined-gently)
4. [The Diamond vs. the Ball -- Core Geometric Intuition](#4-the-diamond-vs-the-ball----core-geometric-intuition)
5. [A Tiny Worked Example: Why L1 Lands on an Axis](#5-a-tiny-worked-example-why-l1-lands-on-an-axis)
6. [The General Optimization Picture](#6-the-general-optimization-picture)
7. [Regularization Strength: The Lambda Knob](#7-regularization-strength-the-lambda-knob)
8. [L0 vs L1 vs L2 Penalty Shapes, Side by Side](#8-l0-vs-l1-vs-l2-penalty-shapes-side-by-side)
9. [Pseudocode: A Minimal L1-Regularized Attack Loop](#9-pseudocode-a-minimal-l1-regularized-attack-loop)
10. [Setting Up for EAD and FISTA](#10-setting-up-for-ead-and-fista)
11. [Real-World Walkthrough -- Tuning Lambda on a Spam-Feature Attack](#11-real-world-walkthrough----tuning-lambda-on-a-spam-feature-attack)
12. [Security Angle](#12-security-angle)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. Recap: Why We Need a Substitute for L0

In the previous section we established that the L0 norm (a straight count of changed features) is exactly what a sparsity attacker wants to minimize, but minimizing it directly is NP-hard: you'd have to try an astronomical number of feature subsets.

This section introduces the standard workaround used across machine learning (not just adversarial attacks -- this same trick shows up in LASSO regression, compressed sensing, and signal processing): **replace the L0 penalty with the L1 penalty**, which is a close cousin that is *much* easier to optimize but still tends to push most entries of a solution to exactly zero.

This is called a **convex relaxation**: we relax ("soften") a hard, discrete, combinatorial problem into a smooth, continuous, convex one that standard optimization tools (gradient descent and its cousins) can solve efficiently -- while hoping (and, under the right conditions, being able to prove) that the solution to the easy problem is close to the solution of the hard problem.

---

## 2. What Does "Convex" Mean, in Plain English?

Before the geometry section, let's make sure "convex" is not a scary word.

**Convex shape, plain English:** if you pick any two points inside (or on the boundary of) the shape and draw a straight line between them, that entire line stays inside the shape. No dents, no "bites taken out."

```
CONVEX (a circle/ball)              NOT CONVEX (a crescent moon)

      .-----.                              .---.
    /         \                          /       \
   |           |          the line     |    .-.    \
   |  o-----o  |   <-- connecting      |   (   )     |
   |           |       two points       \   `-'     /
    \         /        stays inside       \_       _/
      `-----'                               \-----/
                                       the straight line between
  Any line between two               the two "horn tips" would
  points stays inside.               leave the shape (cut through
                                     the empty crescent gap).
```

**Convex function, plain English:** a function whose graph looks like a bowl (like `y = x^2`), not like a wavy landscape with many hills and valleys. A ball rolling on a convex "bowl" function will always roll down to the single lowest point (the **global minimum**), no matter where it starts. This property matters enormously for optimization: gradient descent (the "roll downhill" algorithm from Module 1) is only *guaranteed* to find the best possible answer when the function it's minimizing is convex. On a bumpy, non-convex landscape, gradient descent can get stuck in a shallow local dip that isn't the true lowest point anywhere in the landscape.

The L0 "norm" is not convex (its penalty jumps discretely between "0" and "1" per feature with no smooth in-between), and worse, the *set of allowed solutions* under an L0 budget is disconnected islands, as we saw in Section 1. The L1 norm, by contrast, is convex, and its constraint region is a single connected convex shape. That single fact is why L1 is tractable and L0 is not.

---

## 3. The L1 Norm, Defined Gently

We already saw the L1 norm's formula in the previous section, but let's slow down and build intuition.

**Plain English:** the L1 norm of a vector is the sum of the absolute values of its entries. Think of it as "total distance traveled if you can only move along the grid lines of a city block" (this is literally why L1 distance is also called **Manhattan distance** or **taxicab distance** -- a taxi in a gridded city can't cut diagonally through buildings, it has to drive along the streets).

**Formula:**

```
||delta||_1 = |delta_1| + |delta_2| + ... + |delta_n|
```

**Numeric example.** For `delta = [3, -4]`:

- L1 distance = |3| + |-4| = 3 + 4 = 7 (drive 3 blocks east, then 4 blocks south = 7 blocks total)
- L2 distance = sqrt(3^2 + 4^2) = sqrt(9+16) = sqrt(25) = 5 (the straight-line "as the crow flies" shortcut)

L1 is always greater than or equal to L2 for the same vector (the taxi route is never shorter than flying straight there), and it becomes *much* larger than L2 when a vector's "energy" is spread across many small entries rather than concentrated in a few large ones. That last fact is the seed of why L1 penalties favor sparsity -- concentrating the same total L1 budget into fewer, larger changes is "cheaper" (in the eyes of an L1 penalty) than spreading it thin.

Wait -- concentrating the budget into fewer entries doesn't change the L1 sum at all if the total absolute value is fixed (7 stays 7 whether it's one entry of 7 or seven entries of 1). So *why* does L1 favor sparsity? The answer isn't in the formula alone -- it's in the geometry of how L1 interacts with an optimization objective, which is exactly what the next section unpacks.

---

## 4. The Diamond vs. the Ball -- Core Geometric Intuition

This is the single most important picture in this entire section. Memorize it.

Consider a 2-feature problem (just two pixels, A and B, so we can draw it on paper). We want to find the perturbation `delta = (delta_A, delta_B)` that:

1. Keeps some "norm budget" (L1 or L2) less than or equal to a fixed value, and
2. Makes some loss function (how wrong the model is) as small as possible.

**The shape of the L2 constraint** `delta_A^2 + delta_B^2 <= r^2` is a **ball/circle**: perfectly round, no corners.

**The shape of the L1 constraint** `|delta_A| + |delta_B| <= r` is a **diamond** (a square rotated 45 degrees): four sharp corners, sitting exactly on the axes.

```
     L2 constraint region (a circle)        L1 constraint region (a diamond)

              delta_B                              delta_B
                |                                     |
             .--+--.                                 /|\
           /     |    \                             / | \
          |      |     |                           /  |  \
    ------+------+------+------ delta_A     -------+---+---+------- delta_A
          |      |     |                           \  |  /
           \     |    /                             \ | /
             `--+--'                                 \|/
                |                                     |

   Smooth boundary, no corners.        Sharp CORNERS sit exactly on the
   The optimum can land anywhere       axes (where delta_A=0 or delta_B=0).
   on the boundary, including          Because the loss function's
   points with BOTH delta_A and        contours (see below) are much more
   delta_B nonzero.                    likely to first touch the diamond
                                       AT a corner, sparsity emerges
                                       naturally.
```

Now overlay the *loss function's* contour lines (imagine them as ellipse-shaped rings, like a topographic map, showing where the model's loss is equal -- think of them as "rings of equal wrongness" radiating out from the point where the loss is smallest). As we shrink the norm-constraint shape (the circle or the diamond) down from a large budget to just barely touching those loss contours, we ask: where does the touching point land?

- With a **round** (L2) boundary, the touching point can land *anywhere* on the circle. There's no special reason it should land exactly on an axis. Generically, both coordinates end up nonzero.
- With a **cornered** (L1) diamond, the touching point is disproportionately likely to land exactly ON one of the sharp corners, because the corners "poke into" the loss contours from the axes. When the touching point is a corner, one of the two coordinates is *exactly zero* -- that's a sparse solution!

```
   Loss contours (ellipses) shrinking toward the optimum,
   intersecting each constraint shape:

   L2 (round) case:                    L1 (diamond) case:

        ____                                /\
      /      \                             /  \
     | ellipse |  touches circle           |    |  ellipse touches the
     |  o------+-- anywhere on the         |  o-+--corner (delta_A = 0,
      \       /   boundary --> both        |    |  delta_B != 0)
        `----'    coords likely nonzero      \  /
                                               \/
                                        --> ONE coordinate is
                                            EXACTLY zero: SPARSE!
```

This is why L1-regularized optimization (famous in statistics as **LASSO regression**) reliably drives some coefficients to exactly zero, while L2-regularized optimization (**Ridge regression**) shrinks coefficients toward zero but almost never makes them exactly zero. The same geometric fact, applied to adversarial perturbations instead of regression coefficients, is why L1-penalized attacks produce sparse perturbations while L2-penalized attacks produce dense, spread-out ones.

---

## 5. A Tiny Worked Example: Why L1 Lands on an Axis

Let's make the diamond-corner intuition fully numeric with the smallest possible example: one feature, `delta`, and a simple quadratic loss.

Suppose our optimization objective (the thing we want to minimize) is:

```
objective(delta) = (delta - 3)^2 + lambda * |delta|
```

Here `(delta - 3)^2` is a loss that is minimized at `delta = 3` (pretend that's the "ideal" attack strength with no penalty at all), and `lambda * |delta|` is our L1 penalty pulling delta back toward zero. `lambda` (the Greek letter used almost universally for a regularization strength) is a knob we control -- bigger lambda means we care more about sparsity, less about matching the "ideal" attack strength exactly.

Because of the sharp "kink" in `|delta|` at zero (its slope jumps from -1 to +1 instantly, with no smooth transition), there's a well-known closed-form answer for this specific 1D problem, called **soft-thresholding** (we'll meet this again by name in the FISTA section):

```
delta* = sign(3) * max(|3| - lambda/2, 0)
```

Let's compute it for a few values of lambda:

| lambda | |3| - lambda/2 | delta* (optimal) | Sparse? |
|---|---|---|---|
| 0 | 3 - 0 = 3 | 3.0 | No penalty, full attack strength |
| 2 | 3 - 1 = 2 | 2.0 | Shrunk, but still nonzero |
| 4 | 3 - 2 = 1 | 1.0 | Shrunk further |
| 6 | 3 - 3 = 0 | **0.0** | Exactly zero! Feature "turned off" |
| 10 | 3 - 5 = -2 -> clipped to 0 | **0.0** | Still exactly zero |

Notice that once `lambda` is large enough (lambda >= 6 in this example), the optimal `delta` snaps to **exactly** zero and stays there, no matter how much larger lambda gets. Compare this to an L2 penalty `lambda * delta^2`, whose optimal solution is `delta* = 3 / (1 + lambda)`, which shrinks toward zero as lambda grows but (do the algebra) never actually reaches exactly zero for any finite lambda. That's the algebraic fingerprint of the diamond-corner geometry from Section 4: L1 penalties can zero out a coordinate completely; L2 penalties can only approach zero asymptotically.

```python
def soft_threshold(x, lam):
    """
    The 'soft-thresholding' operator: shrinks x toward zero by lam,
    but clips at zero instead of overshooting past it.
    This is the building block for L1-regularized optimization (see FISTA).
    """
    if x > lam:
        return x - lam
    elif x < -lam:
        return x + lam
    else:
        return 0.0

# Reproduce the table above (note: formula above used lambda/2, here
# we fold the /2 into how we call it for clarity)
ideal_value = 3.0
for lam in [0, 1, 2, 3, 5]:
    print(lam, "->", soft_threshold(ideal_value, lam))
# 0 -> 3.0
# 1 -> 2.0
# 2 -> 1.0
# 3 -> 0.0
# 5 -> 0.0
```

---

## 6. The General Optimization Picture

Scaling the 1D intuition above up to a full image or feature vector, an L1-relaxed sparsity attack solves something like:

```
minimize over delta:

    loss_attack(x + delta)  +  c * ||delta||_1

subject to:  x + delta stays a valid input (e.g. pixel values in [0, 1])
```

Where:

- `loss_attack(x + delta)` measures how far the model's output on the perturbed input is from the attacker's goal (e.g., how confidently it still predicts the *original*, correct class -- the attacker wants this to be small, meaning the model is now fooled).
- `c` is our regularization strength (the same role `lambda` played above), controlling the tradeoff between "fool the model well" and "keep the perturbation sparse."
- `||delta||_1` is the L1 penalty, which (per Sections 4-5) tends to zero out many entries of `delta`, giving us an approximately sparse perturbation without ever explicitly counting or searching over feature subsets.

This is a **convex-ish, smooth-ish** objective (the exact convexity depends on whether `loss_attack` itself is convex, which for neural networks it typically is not globally, but the L1 term at least contributes helpful, well-understood structure), and critically, it's now something we can attack with efficient, well-studied gradient-based solvers -- which is exactly the subject of the FISTA section later in this module.

---

## 7. Regularization Strength: The Lambda Knob

It's worth pausing on the practical role of `c` (or `lambda`), because tuning it is the entire game in real L1-regularized attacks.

```
        c very small                                c very large
        (weak sparsity pressure)                    (strong sparsity pressure)
             |                                              |
             v                                              v
   +-------------------+                          +-------------------+
   | Attack succeeds    |                          | Perturbation is   |
   | easily, but         |    increasing c  -->    | extremely sparse, |
   | perturbation is     |                          | but attack may    |
   | DENSE (many nonzero |                          | FAIL (not enough  |
   | small entries)      |                          | "room" to fool    |
   +-------------------+                          | the model at all) |
                                                     +-------------------+
```

In practice, algorithms like EAD (Section 4) don't pick a single fixed `c` and hope for the best -- they run a **binary search** over `c`, trying larger and smaller values, to find the smallest `c` (i.e., strongest sparsity pressure that still barely succeeds) that still produces a successful misclassification. This search process is a recurring pattern you'll see in the EAD and FISTA sections.

---

## 8. L0 vs L1 vs L2 Penalty Shapes, Side by Side

| Property | L0 | L1 | L2 |
|---|---|---|---|
| **Measures** | Count of nonzero entries | Sum of absolute values | Euclidean length |
| **Constraint region shape** | Disconnected discrete islands | Diamond (convex, has corners) | Ball/sphere (convex, smooth) |
| **Convex?** | No | Yes | Yes |
| **Produces exact zeros?** | Yes, by definition | Yes, thanks to corners touching loss contours | No, only shrinks toward zero asymptotically |
| **Optimization difficulty** | NP-hard | Efficiently solvable (convex) | Efficiently solvable (convex), easiest of all (smooth everywhere) |
| **Used by** | JSMA (approximately), single-pixel search | EAD, LASSO-style attacks, FISTA | Module 9 dense attacks (e.g. Carlini-Wagner L2) |
| **Typical result** | Very few features change, arbitrary magnitude | Most features exactly zero, a few nonzero | Every feature changes a little |

---

## 9. Pseudocode: A Minimal L1-Regularized Attack Loop

This is a simplified sketch (real EAD/FISTA implementations, covered next, are more careful about step sizes and convergence, but this captures the core idea: alternate between a gradient step on the loss, and a soft-threshold step on the L1 penalty).

```python
def l1_regularized_attack(model, x, target_class, c, steps, step_size):
    """
    A minimal proximal-gradient style attack.
    x: original input (e.g. flattened image, one entry per pixel)
    c: L1 regularization strength
    """
    delta = [0.0] * len(x)

    for step in range(steps):
        # 1. Gradient step: nudge delta to reduce the attack loss
        #    (loss_attack tells us how far we are from fooling the model)
        grad = compute_gradient(model, x, delta, target_class)
        for i in range(len(delta)):
            delta[i] = delta[i] - step_size * grad[i]

        # 2. Proximal / soft-threshold step: shrink toward sparsity
        for i in range(len(delta)):
            delta[i] = soft_threshold(delta[i], c * step_size)

        # 3. Clip so x + delta stays a valid input, e.g. in [0, 1]
        for i in range(len(delta)):
            delta[i] = clip(x[i] + delta[i], 0.0, 1.0) - x[i]

    return delta
```

Notice the two-step rhythm: a normal gradient step to chase the attack objective, then a soft-threshold step to enforce sparsity. This "gradient step, then shrink" pattern is precisely what a **proximal gradient method** is, and FISTA (Section 5) is the fast, accelerated version of exactly this loop.

---

## 10. Setting Up for EAD and FISTA

Two threads from this section feed directly into what's next:

1. **EAD (ElasticNet Attack, Section 4)** takes the L1-regularized objective from Section 6 and adds a second, L2 penalty term alongside it, giving the attacker a dial between "pure sparsity" (more L1 weight) and "smooth, low-magnitude changes" (more L2 weight) -- getting some of the sparsity benefits of L1 while keeping the numerically well-behaved, smooth optimization landscape that L2 provides.

2. **FISTA (Section 5)** is the specific, efficient algorithm used to actually *solve* the L1-regularized (or elastic-net-regularized) optimization problem. The "gradient step, then soft-threshold step" loop sketched in Section 9 is the seed of FISTA; FISTA just adds a clever "momentum" trick (borrowing information from previous steps) to converge dramatically faster.

---

## 11. Real-World Walkthrough -- Tuning Lambda on a Spam-Feature Attack

Let's walk through a complete, numeric, multi-feature example of the "shrink lambda knob" idea, extending Section 5's 1D soft-thresholding case to several features at once -- this is much closer to what a real L1-regularized attack loop actually does.

**Setup.** Reuse the spirit of Module 1's spam classifier, but now imagine we are the *attacker*: we want to modify a spam email's feature vector so it slips past the filter as "ham" (legitimate), while touching as few features as possible.

The filter's (simplified, invented) score function, where higher score means "more spam-like":

```
Feature (current spam email values):
  x1 = count_exclamation_marks   = 5
  x2 = contains_"free"            = 1
  x3 = contains_"winner"           = 1
  x4 = count_links                  = 4
  x5 = sender_reputation_score        = 0.1   (0 = bad, 1 = great)

score(x) = 0.5*x1 + 2.0*x2 + 2.5*x3 + 0.8*x4 - 3.0*x5
predict "SPAM" if score(x) > 3.0
```

Current score: `0.5*5 + 2.0*1 + 2.5*1 + 0.8*4 - 3.0*0.1 = 2.5 + 2.0 + 2.5 + 3.2 - 0.3 = 9.9`. Way above the 3.0 threshold -- correctly flagged SPAM.

**Setting up the L1-regularized attack objective.** Suppose (for teaching simplicity) the "ideal, unregularized" attack -- the change that would perfectly zero out the spam score with no sparsity concern at all -- computes to the following per-feature target deltas (found by an unconstrained gradient descent run first, ignoring sparsity):

```
ideal_delta = [ delta_x1=-5, delta_x2=-1, delta_x3=-1, delta_x4=-4, delta_x5=+0.8 ]
```

(i.e., the "no sparsity" attack removes every exclamation mark, every trigger word, every link, and maxes out sender reputation -- a dense change touching all 5 features.)

**Applying soft-thresholding feature-by-feature** (using the same `soft_threshold(x, lambda/2)` mechanic from Section 5, treating each ideal_delta as the "x" input to shrink), for a few lambda values:

| Feature | ideal_delta | lambda=0 | lambda=2 | lambda=6 | lambda=10 |
|---|---|---|---|---|---|
| x1 (exclamations) | -5.0 | -5.0 | -4.0 | -2.0 | 0.0 |
| x2 ("free") | -1.0 | -1.0 | 0.0 | 0.0 | 0.0 |
| x3 ("winner") | -1.0 | -1.0 | 0.0 | 0.0 | 0.0 |
| x4 (links) | -4.0 | -4.0 | -3.0 | -1.0 | 0.0 |
| x5 (reputation) | +0.8 | +0.8 | 0.0 | 0.0 | 0.0 |
| **L0 (features touched)** | | **5** | **2** | **2** | **0** |

Look closely at the `lambda=2` column: features `x2` and `x3` (each with `|ideal_delta| = 1.0`) get shrunk all the way to exactly zero (since `1.0 - 2/2 = 0.0`), while `x1` and `x4` (with larger ideal deltas of 5.0 and 4.0) survive with reduced magnitude, and `x5` (ideal delta only 0.8) also gets zeroed. The L1 penalty has automatically identified that "removing exclamation marks and links" carries more attack value per unit of penalty than "tweaking sender reputation or removing a couple of trigger words," and sacrificed the low-value features first while preserving the high-value ones -- all without ever being told explicitly which features "matter most." That emergent prioritization is the direct, practical payoff of the diamond-corner geometry from Section 4.

**Checking attack success at lambda=2:** new feature vector: `x1=1, x2=1, x3=1, x4=1, x5=0.1` (unchanged). New score: `0.5*1 + 2.0*1 + 2.5*1 + 0.8*1 - 3.0*0.1 = 0.5+2.0+2.5+0.8-0.3 = 5.5`. Still above 3.0 -- **attack fails** at this sparsity level; we shrunk too aggressively.

**Backing off to lambda=1** (less sparsity pressure): shrinking each ideal_delta by `1/2 = 0.5`: `delta = [-4.5, -0.5, -0.5, -3.5, +0.3]`, giving new values `x1=0.5, x2=0.5, x3=0.5, x4=0.5, x5=0.4`. New score: `0.5*0.5+2.0*0.5+2.5*0.5+0.8*0.5-3.0*0.4 = 0.25+1.0+1.25+0.4-1.2 = 1.7`. Below 3.0 -- **attack succeeds**, and all 5 features are still touched somewhat (L0=5 at this lambda), though each by a smaller amount than the "ideal" dense attack.

This mirrors exactly the binary-search-over-lambda process described in Section 7: too much sparsity pressure (lambda=2) failed the attack; too little (lambda=0, the ideal dense case) succeeds but isn't sparse at all; a real EAD-style attack would binary search between these to find the sparsest lambda that still crosses the threshold -- which for this toy example turns out to be somewhere between 1 and 2, worth narrowing down further if we wanted the truly minimal sparse solution.

## 12. Security Angle

- **Why L1 relaxation matters practically for attackers:** it turns an intractable search problem into something solvable in seconds with standard gradient-based tooling that already exists in every deep learning framework (PyTorch, TensorFlow). An attacker doesn't need custom combinatorial-search code -- they need a training loop with an extra soft-threshold step.

- **Realistic feature-cost modeling.** In malware or network-traffic evasion, not all features are equally "safe" to change. An attacker can weight the L1 penalty per-feature (e.g., `c_i * |delta_i|` instead of a single shared `c`), effectively saying "changing this API-call feature is cheap, but changing this file-size feature is expensive/risky, so only touch it if the gain is huge." This lets the L1 framework directly encode real-world attack cost, not just abstract sparsity.

- **Detectability tradeoff.** Because L1-regularized attacks explicitly trade off "attack success" against "how sparse the change is" via the tunable `c`, they give a defender-relevant lever: a red team can produce a family of adversarial examples ranging from "very sparse but might not always succeed" to "always succeeds but touches more features," letting them map out exactly how much stealth costs in success rate against a specific target model.

---

## 13. Key Takeaways

- The **L1 norm** (sum of absolute values, aka "Manhattan/taxicab distance") is used as a convex, tractable **relaxation** of the intractable L0 norm.
- Convexity means: no shortcuts get skipped, gradient descent reliably finds the true minimum, and the constraint region is one connected shape (unlike L0's disconnected discrete islands).
- The **geometric reason** L1 induces sparsity: the L1 constraint region is a diamond with sharp corners sitting exactly on the coordinate axes. Loss-function contours are disproportionately likely to first touch the diamond at one of those corners, which forces some coordinates to be *exactly* zero. The L2 ball has no corners, so its optimal touching point generically keeps every coordinate nonzero.
- The **soft-thresholding** operator (`shrink toward zero, clip at zero`) is the concrete mathematical mechanism that produces exact zeros in 1D L1-regularized problems, and it is a key building block reused directly inside FISTA.
- The regularization strength `c` (or `lambda`) is a dial: small `c` favors attack success at the cost of density; large `c` favors sparsity at the risk of attack failure. Real attacks (like EAD) binary-search over `c`.
- This section directly sets up **EAD** (L1 + L2 combined objective) and **FISTA** (the efficient solver for that objective), both covered next.

---

*Next up: Saliency-Based Feature Selection -- how gradient information lets an attacker rank features by "how much they matter" to the model's decision, so a limited perturbation budget can be spent on the highest-impact features first.*
