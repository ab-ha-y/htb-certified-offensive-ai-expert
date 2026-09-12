# Targeted FGSM

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - First-Order Attacks | Section: Targeted FGSM

---

## Table of Contents

1. [Untargeted vs. Targeted: What's the Difference?](#1-untargeted-vs-targeted-whats-the-difference)
2. [The Plain-English Idea](#2-the-plain-english-idea)
3. [Why the Gradient Direction Flips](#3-why-the-gradient-direction-flips)
4. [Deriving Targeted FGSM](#4-deriving-targeted-fgsm)
5. [The Formula](#5-the-formula)
6. [Worked Numeric Example: A Tiny 3-Class Model](#6-worked-numeric-example-a-tiny-3-class-model)
7. [Pseudocode](#7-pseudocode)
8. [Visualizing Targeted vs. Untargeted Attacks](#8-visualizing-targeted-vs-untargeted-attacks)
9. [Comparison Table: Untargeted FGSM vs. Targeted FGSM](#9-comparison-table-untargeted-fgsm-vs-targeted-fgsm)
10. [Why Targeted Attacks Are Usually Harder](#10-why-targeted-attacks-are-usually-harder)
11. [Security Angle](#11-security-angle)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. Untargeted vs. Targeted: What's the Difference?

Plain FGSM (previous file) is **untargeted**: its only goal is to make the model *wrong*, no matter which wrong answer it lands on. If a malware classifier is fooled into saying "not malware," the attacker doesn't care whether the model's internal confidence secretly leans toward thinking it's a PDF, a spreadsheet, or a driver -- "wrong" is enough.

**Targeted FGSM** raises the bar: the attacker picks a *specific* class in advance -- say, "make this malware sample get classified specifically as `notepad.exe`-like benign software," or "make this stop sign image get classified specifically as a speed limit sign" -- and crafts a perturbation engineered to push the model toward *that exact* wrong answer, not just any wrong answer.

### The Analogy

Untargeted FGSM is like tripping someone so they fall down -- you don't care which direction they land. Targeted FGSM is like tripping someone so they fall specifically into the swimming pool to their left, not just anywhere on the ground. It takes more precision, and it's a strictly harder task than "just make them fall."

---

## 2. The Plain-English Idea

Recall from the local linearity file that the gradient points in the direction that **increases** the loss the fastest -- and increasing the loss with respect to the *true* label is exactly what untargeted FGSM wants (make the model wrong about the truth).

For a targeted attack, we flip the question: instead of "how do I make the model wrong about `y_true`?" we ask "how do I make the model *confidently believe* the input is `y_target` (some specific class the attacker has chosen, different from the true label)?" That means we want to **decrease** the loss *with respect to the target label*, not increase the loss with respect to the true label. Decreasing a loss is exactly what normal *training* does (recall: "step opposite the gradient, loss goes down" from the local linearity file) -- so targeted FGSM effectively borrows the training-time direction, but points it at a label the model should never actually agree with.

---

## 3. Why the Gradient Direction Flips

Let's be precise about which loss and which gradient we're using, because this is the one part of targeted FGSM that's genuinely easy to get backwards.

```
UNTARGETED FGSM:
  Goal:  make the model WRONG about the TRUE label
  Loss used:      L(model(x), y_true)
  Direction:      step ALONG the gradient of THIS loss
                   (increases loss w.r.t. true label --> model gets confused
                    about the correct answer)
  Formula sign:   x_adv = x + epsilon * sign(gradient_x L(model(x), y_true))
                                          ^ PLUS


TARGETED FGSM:
  Goal:  make the model CONFIDENT about a CHOSEN WRONG label (y_target)
  Loss used:      L(model(x), y_target)
  Direction:      step OPPOSITE the gradient of THIS loss
                   (decreases loss w.r.t. the target label --> model becomes
                    MORE confident the input is y_target, even though it's
                    not the true label)
  Formula sign:   x_adv = x - epsilon * sign(gradient_x L(model(x), y_target))
                                          ^ MINUS
```

The core lesson: **the sign flips from plus to minus, and the label plugged into the loss function changes from the true label to the attacker's chosen target label.** Everything else -- the gradient computation, the sign function, the L-infinity budget -- is identical to plain FGSM.

---

## 4. Deriving Targeted FGSM

We follow the exact same optimization logic as plain FGSM (Section 5 of `fgsm.md`), but with a different objective.

**Step 1**: We want to choose `delta` to *minimize* the loss with respect to the target label (i.e., make the model believe the target label is correct):

```
minimize   L(model(x + delta), y_target)
subject to ||delta||_inf <= epsilon
```

**Step 2**: Apply the same local linearity assumption as before:

```
L(model(x + delta), y_target)  ~=  L(model(x), y_target)  +  gradient_x(L, y_target) . delta
```

**Step 3**: Since `L(model(x), y_target)` is a fixed constant (it doesn't depend on `delta`), minimizing the whole expression is the same as minimizing just the second term, `gradient_x(L, y_target) . delta`.

**Step 4**: Minimizing a dot product subject to an L-infinity ball constraint is solved by pointing `delta` in the *opposite* direction of the gradient's sign, at full budget (the mirror image of Section 5 in `fgsm.md`, which *maximized* the same kind of expression):

```
delta = -epsilon * sign(gradient_x(L(model(x), y_target)))
```

---

## 5. The Formula

```
x_adv = x - epsilon * sign( gradient_x( L(model(x), y_target) ) )

  y_target    = the SPECIFIC wrong class the attacker wants the model to output
                (chosen by the attacker, NOT the true label)
  Everything else is identical to plain FGSM.
```

**Memory aid**: "Minus for a target." Untargeted FGSM adds the sign-gradient of the *true*-label loss (push away from truth). Targeted FGSM subtracts the sign-gradient of the *target*-label loss (pull toward the chosen lie).

---

## 6. Worked Numeric Example: A Tiny 3-Class Model

Let's extend the worked example style to a 3-class problem, since "targeting a specific class" only makes sense once there's more than one wrong answer to choose between.

### The Model

A simple linear "scoring" model with 2 input features and 3 output classes (A, B, C). Each class has its own weight vector and bias; the class with the highest score wins (this is a simplified stand-in for the final layer of many real classifiers, followed by softmax to turn scores into probabilities):

```
score_A = 1.0*x1 + 0.5*x2 + 0.0
score_B = -0.5*x1 + 1.0*x2 + 0.1
score_C = 0.2*x1 - 0.3*x2 + 0.2
```

### The Input

```
x = [x1, x2] = [2.0, 1.0]
true label = A
```

### Step 1: Compute the Scores

```
score_A = 1.0*2.0 + 0.5*1.0 + 0.0 = 2.0 + 0.5 = 2.5
score_B = -0.5*2.0 + 1.0*1.0 + 0.1 = -1.0 + 1.0 + 0.1 = 0.1
score_C = 0.2*2.0 - 0.3*1.0 + 0.2 = 0.4 - 0.3 + 0.2 = 0.3

Scores: A = 2.5, B = 0.1, C = 0.3
```

`A` has the highest score, so the model correctly predicts class A. The attacker now decides: "I don't just want the model wrong -- I specifically want it to say **C**." (Say, class C represents "benign" in a malware classifier, and A represents "malicious" -- the attacker specifically wants to land in the benign bucket, not just any non-malicious-sounding bucket.)

### Step 2: Set Up the Gradient Toward the Target Class C

For a simplified linear scoring setup like this, increasing an input feature's contribution to `score_C` while decreasing its contribution to the other scores is exactly what "the gradient of the loss with respect to `y_target = C`" captures. To keep the arithmetic transparent, we'll use a common simplification for this kind of margin/score-based setup: the gradient that *reduces* loss with respect to target `C` roughly points toward increasing `score_C` relative to the other scores. Using the weight vector for `C` directly as a stand-in for this gradient direction (a standard result for linear/softmax models near the decision boundary):

```
gradient_toward_C = weight_vector_of_C = [0.2, -0.3]
```

Since we want the model to become MORE confident in C (i.e., we want to move toward increasing `score_C`), and increasing `score_C` corresponds to moving *along* its weight vector, the targeted-FGSM "minus the sign of the gradient of the loss" ends up equivalent to moving *along* `sign(weight_vector_of_C)` in this simplified linear picture. (This sign-flip bookkeeping -- loss vs. score, minus vs. plus -- is exactly why Section 3 above spells out the direction so explicitly; different textbooks define the loss with different signs, but the operational rule "move toward increasing the target class's score" is the one that always holds.)

```
sign(gradient_toward_C) = sign([0.2, -0.3]) = [+1, -1]
```

### Step 3: Build the Perturbation, Using Budget epsilon = 0.2

```
delta = epsilon * [+1, -1] = [+0.2, -0.2]
```

(Note the direction here is "add the sign of the direction that increases score_C" -- consistent with our derivation that targeted FGSM moves *toward* increasing the target class's score, which is the "minus the gradient of the target loss" rule from Section 4, expressed in score-space instead of loss-space for this simplified linear example.)

### Step 4: Apply the Perturbation

```
x_adv = x + delta = [2.0, 1.0] + [0.2, -0.2] = [2.2, 0.8]
```

### Step 5: Recompute the Scores on the Adversarial Input

```
score_A = 1.0*2.2 + 0.5*0.8 + 0.0 = 2.2 + 0.4        = 2.6
score_B = -0.5*2.2 + 1.0*0.8 + 0.1 = -1.1 + 0.8 + 0.1 = -0.2
score_C = 0.2*2.2 - 0.3*0.8 + 0.2 = 0.44 - 0.24 + 0.2 = 0.4

Scores: A = 2.6, B = -0.2, C = 0.4
```

### Result

```
BEFORE attack:  A = 2.5 (winner), B = 0.1, C = 0.3
AFTER  attack:  A = 2.6 (still winner), B = -0.2, C = 0.4

score_C moved UP (0.3 -> 0.4) exactly as intended by the targeted push,
but A moved up even more (2.5 -> 2.6), so the model is still not fooled
with this modest epsilon.
```

This mirrors the realistic outcome from the plain FGSM worked example: a single small step nudges things in the right direction (score_C did rise, relatively, versus a same-sized untargeted step which wouldn't specifically favor C), but a bigger epsilon or an iterative approach (next file) is typically needed to fully cross the boundary and make C the winning class. This is expected and realistic -- targeted attacks are intrinsically harder than untargeted ones (see Section 10), and toy 2-feature examples with modest epsilon often need more than one step to succeed, exactly the same way our plain FGSM example did.

---

## 7. Pseudocode

```python
import numpy as np

def targeted_fgsm_attack(model, loss_fn, x, y_target, epsilon):
    """
    model:     a function x -> prediction (logits or probabilities per class)
    loss_fn:   a function (prediction, label) -> scalar loss
    x:         original input, numpy array
    y_target:  the SPECIFIC class the attacker wants the model to predict
               (chosen by the attacker; NOT the true label)
    epsilon:   L-infinity perturbation budget
    """
    x = x.copy()
    x.requires_grad = True

    prediction = model(x)
    # Notice: loss computed against y_target, NOT y_true
    loss = loss_fn(prediction, y_target)

    grad = compute_gradient(loss, wrt=x)

    # KEY DIFFERENCE from untargeted FGSM: SUBTRACT instead of ADD
    perturbation = -epsilon * np.sign(grad)
    x_adv = x + perturbation

    x_adv = np.clip(x_adv, 0.0, 1.0)
    return x_adv


def targeted_fgsm_manual_example():
    """Standalone version of the Section 6 worked example."""
    W = {
        "A": np.array([1.0, 0.5]),
        "B": np.array([-0.5, 1.0]),
        "C": np.array([0.2, -0.3]),
    }
    bias = {"A": 0.0, "B": 0.1, "C": 0.2}
    x = np.array([2.0, 1.0])
    y_target = "C"
    epsilon = 0.2

    def scores(x):
        return {c: np.dot(W[c], x) + bias[c] for c in W}

    before = scores(x)

    # Simplified linear stand-in for "gradient toward increasing y_target's score"
    grad_toward_target = W[y_target]
    delta = epsilon * np.sign(grad_toward_target)
    x_adv = x + delta

    after = scores(x_adv)

    print(f"Scores before: {before}")
    print(f"Perturbation:  {delta}")
    print(f"Scores after:  {after}")
```

---

## 8. Visualizing Targeted vs. Untargeted Attacks

```
                UNTARGETED FGSM                       TARGETED FGSM

         Class B region      Class C region     Class B region      Class C region
        +----------------+----------------+    +----------------+----------------+
        |                |                |    |                |                |
        |                |                |    |                |                |
        |     x_adv <----*                |    |                *----> x_adv     |
        |          (any direction         |    |          (SPECIFIC direction     |
        |           away from A works)    |    |           toward C's region      |
        |                * x (original,   |    |                * x (original,   |
        |                  class A)       |    |                  class A)       |
        +----------------+----------------+    +----------------+----------------+
                Class A region                          Class A region

   Untargeted: any wall works,          Targeted: only the wall bordering
   attacker escapes into ANY            the SPECIFIC target region works;
   neighboring region.                  attacker must aim precisely.
```

---

## 9. Comparison Table: Untargeted FGSM vs. Targeted FGSM

| Aspect | Untargeted FGSM | Targeted FGSM |
|---|---|---|
| **Goal** | Any misclassification | A specific, chosen misclassification |
| **Loss used** | `L(model(x), y_true)` | `L(model(x), y_target)` |
| **Step direction** | `+ epsilon * sign(gradient)` | `- epsilon * sign(gradient)` |
| **Formula** | `x_adv = x + eps * sign(grad L(x, y_true))` | `x_adv = x - eps * sign(grad L(x, y_target))` |
| **Difficulty** | Easier -- many "exits" from the correct region | Harder -- only one specific "exit" counts |
| **Effective epsilon needed** | Typically smaller | Typically larger, for the same success rate |
| **Real-world example** | Make malware "not flagged" (don't care what it's classified as) | Make malware specifically classified as `notepad.exe`-like, to blend into an allow-list |
| **Requires** | Only the true label | The true label AND a chosen target label |

---

## 10. Why Targeted Attacks Are Usually Harder

Think back to the decision-boundary diagram in Section 8. An input sitting inside "Class A's" region is typically bordered by *multiple* other classes' regions. Untargeted FGSM only needs to cross *any one* of those borders -- it has many possible "exits." Targeted FGSM must cross *specifically* the border facing the chosen target class, which may be the *farthest* border, or may require a perturbation direction that doesn't align well with the single gradient step FGSM computes.

This is precisely why targeted attacks more often require the **iterative** refinement covered in the next file (I-FGSM): a single linear step, however well-aimed, is less likely to land precisely inside one specific target region than it is to land in *some* wrong region, especially when the target region is geometrically distant or narrow.

---

## 11. Security Angle

- **Targeted attacks are what turns "evasion" into "impersonation."** An untargeted attack against a face-recognition access control system might just make it fail to recognize the attacker as anyone (a denial-of-service-flavored bypass). A **targeted** attack could make the system specifically recognize the attacker *as a different, authorized person* -- a much more dangerous outcome, enabling actual unauthorized access rather than just noise.
- **Malware classifiers are a classic targeted-attack scenario.** Untargeted evasion just needs "not flagged as malware" (any benign-ish label works). But some detection pipelines route different labels to different downstream handling (e.g., "benign" samples skip a sandbox, while "unknown" samples still get sandboxed) -- so an attacker who wants to skip the sandbox needs a **targeted** attack that lands specifically in the "known-benign" bucket, not just anywhere outside "malicious."
- **Detecting targeted attacks sometimes leaves a distinct statistical fingerprint.** Because targeted perturbations are optimized to move toward one *specific* class, monitoring for inputs that sit unusually close to a specific class boundary (rather than generally "confused" inputs near multiple boundaries) can sometimes help defenders distinguish targeted adversarial traffic from ordinary misclassifications or noise.

---

## 12. Key Takeaways

- **Untargeted FGSM** asks "make the model wrong, I don't care how" -- it increases the loss with respect to the **true** label.
- **Targeted FGSM** asks "make the model specifically believe this chosen wrong answer" -- it decreases the loss with respect to a **chosen target** label.
- The formula flips both the **label used** (`y_true` becomes `y_target`) and the **sign** (`+epsilon` becomes `-epsilon`): `x_adv = x - epsilon * sign(gradient_x(L(model(x), y_target)))`.
- Targeted attacks are intrinsically **harder** than untargeted attacks because they must cross one *specific* decision boundary rather than any of the several boundaries surrounding the correct region.
- The worked 3-class example showed the target class's score genuinely moving in the intended direction, but a modest epsilon and low dimensionality meant a single step wasn't enough to fully flip the winner -- a pattern that motivates the iterative and boundary-aware attacks covered next.

---

*Next up: I-FGSM (Iterative FGSM) -- taking many small FGSM-style steps, with re-projection back into the epsilon-ball after each one, to build attacks that are stronger than a single FGSM shot (targeted or untargeted).*
