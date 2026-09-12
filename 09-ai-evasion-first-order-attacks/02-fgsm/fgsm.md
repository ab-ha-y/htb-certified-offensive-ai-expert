# FGSM: The Fast Gradient Sign Method

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - First-Order Attacks | Section: FGSM

---

## Table of Contents

1. [What Problem Is FGSM Solving?](#1-what-problem-is-fgsm-solving)
2. [The Plain-English Idea](#2-the-plain-english-idea)
3. [Recap: The Two Ingredients FGSM Needs](#3-recap-the-two-ingredients-fgsm-needs)
4. [Why the SIGN, Not the Raw Gradient?](#4-why-the-sign-not-the-raw-gradient)
5. [Deriving FGSM from the Local Linearity Assumption](#5-deriving-fgsm-from-the-local-linearity-assumption)
6. [The Formula](#6-the-formula)
7. [Worked Numeric Example: A Tiny Logistic Model](#7-worked-numeric-example-a-tiny-logistic-model)
8. [Pseudocode](#8-pseudocode)
9. [Visualizing the FGSM Step](#9-visualizing-the-fgsm-step)
10. [What Epsilon Controls -- A Comparison Table](#10-what-epsilon-controls-a-comparison-table)
11. [Strengths and Limitations](#11-strengths-and-limitations)
12. [Security Angle](#12-security-angle)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. What Problem Is FGSM Solving?

FGSM (Fast Gradient Sign Method) was introduced by Ian Goodfellow, Jonathon Shlens, and Christian Szegedy in their 2015 paper *"Explaining and Harnessing Adversarial Examples."* It answers a very specific question:

> "Given a model, a correctly-classified input, and a tiny perturbation budget `epsilon` (measured in L-infinity norm), what single, cheap computation produces a perturbation that is *likely* to flip the model's prediction?"

FGSM's answer is deliberately simple: compute the gradient of the loss with respect to the input **once**, and immediately convert it into a full-strength perturbation. No iteration, no search, no expensive optimization loop -- just one forward pass and one backward pass through the model.

---

## 2. The Plain-English Idea

### The Analogy

Imagine you're blindfolded and standing somewhere on a hilly landscape, and someone whispers in your ear which direction is uphill for each of your 4 compass directions (north, south, east, west) -- but only tells you "uphill" or "downhill" for each one, not by how much. If you were told "north is uphill, east is downhill, south is uphill, west is downhill," the most aggressive thing you could do, using only that limited "which way" information, is take one confident step: north, and south simultaneously isn't possible, so really: one full step in the *combined* direction of "every direction that was called uphill."

FGSM does exactly this for every feature in your input simultaneously: for each feature, it asks "does increasing this feature increase the loss (uphill) or decrease it (downhill)?" and then takes a full-sized step of size `epsilon` in the "uphill" direction for every single feature, all at once, in one shot.

---

## 3. Recap: The Two Ingredients FGSM Needs

You already met both of these in the Foundations section:

1. **The gradient** of the loss with respect to the input, `gradient(L, x)` -- one number per feature, telling you which direction increases the loss (Section 3, `local-linearity-assumption.md`).
2. **A norm budget**, `epsilon`, measured in the **L-infinity** norm -- capping how much any single feature is allowed to change (Section 4.4, `norm-constraints.md`).

FGSM combines these two ingredients in the cheapest possible way that still respects the L-infinity budget.

---

## 4. Why the SIGN, Not the Raw Gradient?

This is the single most important design decision in FGSM, and it comes directly from the L-infinity norm ball discussed in the foundations section.

Recall (Section 6 of `norm-constraints.md`) that the L-infinity ball is a **square** (or, in high dimensions, a hypercube) -- every feature is independently allowed to reach the *maximum* allowed change of `epsilon` at the same time. Given a fixed L-infinity budget, if your goal is to maximize `gradient . delta` (the approximate increase in loss, from Section 6 of `local-linearity-assumption.md`), the best strategy is:

> For each feature `i`, set `delta_i` to `+epsilon` if `gradient_i` is positive, or `-epsilon` if `gradient_i` is negative -- i.e., always push each feature by the *maximum allowed amount, in whichever direction that particular feature's gradient says helps.*

This is exactly what the mathematical **sign function** computes:

```
sign(g) =  +1   if g > 0
           -1   if g < 0
            0   if g == 0
```

Using only the *sign* of the gradient (rather than its exact magnitude) means every feature gets pushed by the full `epsilon`, maximizing the total effect within the L-infinity budget. This is *not* an approximation for convenience -- it is provably the exact optimal choice of `delta` (within the L-infinity ball) for maximizing the linear approximation `gradient . delta`, which is why the name of the method includes "sign," not "raw gradient" or "scaled gradient."

### Quick Numeric Illustration

Suppose the gradient for 4 features is `[3.2, -0.01, -7.5, 0.0004]`, and our L-infinity budget is `epsilon = 0.03`.

```
gradient        = [ 3.2,  -0.01,  -7.5,  0.0004]
sign(gradient)  = [ +1,     -1,     -1,     +1  ]
delta = eps *
sign(gradient)  = [+0.03, -0.03, -0.03, +0.03]
```

Notice that the *tiny* gradient value `0.0004` and the *huge* gradient value `7.5` both get converted into the exact same magnitude of perturbation, `0.03` -- only the sign matters. This might feel wasteful at first ("shouldn't a huge gradient get a bigger push?"), but remember: we are working under a strict L-infinity budget that caps every feature at `epsilon` regardless. Given that hard cap, using the full allowed budget on *every* feature (weighted only by direction) is what maximizes the total linear effect, per the high-dimensional accumulation argument from the previous file.

---

## 5. Deriving FGSM from the Local Linearity Assumption

Let's connect this directly back to the math from the Foundations section.

**Step 1**: We want to choose `delta` to maximize the (approximate) increase in loss:

```
maximize   gradient(L, x) . delta
subject to ||delta||_inf <= epsilon
```

**Step 2**: This is a well-known type of optimization problem (maximizing a linear function subject to an L-infinity ball constraint). Its exact solution, for each feature `i` independently, is:

```
delta_i = epsilon * sign(gradient_i)
```

**Step 3**: Stack all the per-feature solutions into one perturbation vector:

```
delta = epsilon * sign(gradient(L, x))
```

**Step 4**: Add the perturbation to the original input to get the adversarial example:

```
x_adv = x + epsilon * sign(gradient(L, x))
```

That's the entire derivation. Every piece traces directly back to concepts already introduced: the gradient (which direction increases loss), the local linearity assumption (why a single linear step is trustworthy for small `epsilon`), and the L-infinity norm ball (why the sign function, not the raw gradient, is the correct maximizer).

---

## 6. The Formula

```
x_adv = x + epsilon * sign( gradient_x( L(model(x), y_true) ) )

  x            = original input
  y_true       = the TRUE label (untargeted FGSM pushes AWAY from this)
  model(x)     = the model's prediction on x
  L(...)       = the loss function comparing prediction to true label
  gradient_x   = gradient of the loss with respect to the INPUT x
                 (not the model's internal parameters!)
  sign(...)    = +1 / -1 / 0 per element
  epsilon      = the attack budget (L-infinity norm cap)
```

**Important clarification**: during normal training, gradients are computed with respect to the model's *parameters* (weights), and the model changes. During an FGSM attack, the model is completely frozen -- gradients are computed with respect to the *input*, and the input changes instead. Same underlying calculus machinery (backpropagation), completely different target of the computation.

---

## 7. Worked Numeric Example: A Tiny Logistic Model

Let's build the smallest possible concrete example: a **logistic regression** binary classifier with 2 input features, and manually walk through an entire FGSM attack, by hand.

### The Model

```
z = w1*x1 + w2*x2 + b
prediction = sigmoid(z) = 1 / (1 + e^(-z))     (outputs a probability between 0 and 1)

Weights (already trained, frozen):  w1 = 2.0,  w2 = -1.5,  b = 0.5
```

The model predicts "class 1" if `prediction > 0.5`, and "class 0" otherwise.

### The Input

```
x = [x1, x2] = [1.0, 0.5]
true label, y_true = 1     (the correct answer is "class 1")
```

### Step 1: Forward Pass -- Compute the Model's Current Prediction

```
z = (2.0 * 1.0) + (-1.5 * 0.5) + 0.5
  = 2.0 - 0.75 + 0.5
  = 1.75

prediction = sigmoid(1.75) = 1 / (1 + e^(-1.75)) ~= 0.852
```

The model is 85.2% confident this is "class 1" -- and it's correct (`y_true = 1`), so this is a strong, correct prediction. Good target for an evasion attack.

### Step 2: Compute the Loss

Using binary cross-entropy loss (a standard loss for probabilities), for `y_true = 1`:

```
L = -log(prediction) = -log(0.852) ~= 0.160
```

A small loss, as expected -- the model is doing well.

### Step 3: Compute the Gradient of the Loss with Respect to the Input

For logistic regression with binary cross-entropy loss, the gradient with respect to the pre-activation `z` simplifies neatly to `(prediction - y_true)`:

```
dL/dz = prediction - y_true = 0.852 - 1 = -0.148
```

Then, by the chain rule, the gradient with respect to each input feature is `dL/dz` multiplied by that feature's weight (since `z = w1*x1 + w2*x2 + b`, so `dz/dx1 = w1` and `dz/dx2 = w2`):

```
dL/dx1 = dL/dz * w1 = -0.148 * 2.0  = -0.296
dL/dx2 = dL/dz * w2 = -0.148 * -1.5 =  0.222

gradient = [-0.296, 0.222]
```

### Step 4: Take the Sign of the Gradient

```
sign(gradient) = [sign(-0.296), sign(0.222)]
               = [   -1,           +1      ]
```

### Step 5: Build the Perturbation, Using Budget epsilon = 0.1

```
delta = epsilon * sign(gradient)
      = 0.1 * [-1, +1]
      = [-0.1, +0.1]
```

### Step 6: Apply the Perturbation

```
x_adv = x + delta
      = [1.0, 0.5] + [-0.1, +0.1]
      = [0.9, 0.6]
```

### Step 7: Forward Pass on the Adversarial Input -- Did It Work?

```
z_adv = (2.0 * 0.9) + (-1.5 * 0.6) + 0.5
      = 1.8 - 0.9 + 0.5
      = 1.4

prediction_adv = sigmoid(1.4) = 1 / (1 + e^(-1.4)) ~= 0.802
```

### Result

```
BEFORE attack: prediction = 0.852  (confidently "class 1", correct)
AFTER  attack: prediction = 0.802  (still "class 1", but less confident)
```

In this particular tiny, low-dimensional example, the single FGSM step reduced the model's confidence from 85.2% to 80.2%, but did not fully flip the decision (it's still above the 0.5 threshold). This is intentional and realistic: with only 2 dimensions and a modest `epsilon`, one step often isn't enough to cross the boundary -- this is exactly why, per the high-dimensional effects file, FGSM tends to be far more devastating on high-dimensional inputs (thousands of pixels, each contributing a small nudge that adds up), and why the next files in this module (Targeted FGSM, I-FGSM) exist to push harder or more precisely.

**Try it yourself**: if you increase `epsilon` to `0.3` in this same example, `x_adv = [1.0 - 0.3, 0.5 + 0.3] = [0.7, 0.8]`, giving `z_adv = 2.0*0.7 - 1.5*0.8 + 0.5 = 1.4 - 1.2 + 0.5 = 0.7`, and `sigmoid(0.7) ~= 0.668` -- confidence keeps dropping as `epsilon` grows, illustrating the direct epsilon-to-effect relationship.

---

## 8. Pseudocode

```python
import numpy as np

def fgsm_attack(model, loss_fn, x, y_true, epsilon):
    """
    model:    a function x -> prediction (e.g., probability or logits)
    loss_fn:  a function (prediction, y_true) -> scalar loss
    x:        original input, numpy array
    y_true:   true label
    epsilon:  L-infinity perturbation budget
    """
    # Forward pass: track gradient of loss w.r.t. INPUT (not weights)
    x = x.copy()
    x.requires_grad = True                 # conceptual flag; frameworks like
                                            # PyTorch/TensorFlow handle this
                                            # automatically via autograd

    prediction = model(x)
    loss = loss_fn(prediction, y_true)

    # Backward pass: compute dL/dx (backpropagation)
    grad = compute_gradient(loss, wrt=x)   # one value per input feature

    # FGSM step: full-strength push in the sign direction, within epsilon budget
    perturbation = epsilon * np.sign(grad)
    x_adv = x + perturbation

    # Optional: clip to valid input range (e.g., pixel values in [0, 1])
    x_adv = np.clip(x_adv, 0.0, 1.0)

    return x_adv


def fgsm_numpy_manual_example():
    """Standalone version of the Section 7 worked example, no framework needed."""
    w = np.array([2.0, -1.5])
    b = 0.5
    x = np.array([1.0, 0.5])
    y_true = 1
    epsilon = 0.1

    def sigmoid(z):
        return 1 / (1 + np.exp(-z))

    z = np.dot(w, x) + b
    pred = sigmoid(z)

    # For logistic regression + binary cross-entropy, dL/dz simplifies to (pred - y_true)
    dL_dz = pred - y_true
    grad = dL_dz * w                       # chain rule: dL/dx_i = dL/dz * dz/dx_i = dL/dz * w_i

    delta = epsilon * np.sign(grad)
    x_adv = x + delta

    z_adv = np.dot(w, x_adv) + b
    pred_adv = sigmoid(z_adv)

    print(f"Original prediction:   {pred:.3f}")
    print(f"Perturbation:          {delta}")
    print(f"Adversarial input:     {x_adv}")
    print(f"Adversarial prediction:{pred_adv:.3f}")
```

---

## 9. Visualizing the FGSM Step

```
                 DECISION BOUNDARY (where prediction = 0.5)
                              |
                              |
    Correctly classified     |     Misclassified
    region (class 1)         |     region (class 0)
                              |
              x (original)   |
                *             |
                 \            |
                  \  FGSM step: one full-strength
                   \ jump of size epsilon, along
                    \ sign(gradient)
                     v
                      * x_adv
                              |
                              |

   Compare to a "rolling downhill" gradient-descent picture from training:
   FGSM is the mirror image -- instead of rolling DOWN the loss hill to
   reduce error, it takes one confident LEAP UP the loss hill, in the
   steepest-ascent direction allowed by the epsilon budget.
```

---

## 10. What Epsilon Controls -- A Comparison Table

| Epsilon Value | Perturbation Visibility | Likely Effect on Prediction | Typical Use |
|---|---|---|---|
| Very small (e.g., 0.001-0.01 on [0,1]-scaled pixels) | Imperceptible to humans | May only reduce confidence slightly, may not flip decision (as seen in Section 7) | Testing model sensitivity / robustness benchmarks |
| Small-to-moderate (e.g., 0.03-0.1) | Faint, barely visible noise on images | Often flips predictions on high-dimensional inputs (images), less reliably on low-dimensional tabular inputs | Standard adversarial robustness research budget (e.g., 8/255 ~= 0.031) |
| Large (e.g., 0.3+) | Visible distortion/artifacts | Very likely to flip the decision, but may also be noticeable to a human reviewer or trigger simple anomaly filters | Stress-testing, worst-case robustness evaluation |

---

## 11. Strengths and Limitations

| Aspect | FGSM |
|---|---|
| **Speed** | Extremely fast -- exactly one forward pass + one backward pass, regardless of model size |
| **Steps** | Single-step (non-iterative) |
| **Norm used** | L-infinity |
| **Targeted or untargeted** | Untargeted by default (pushes away from the true label, doesn't choose a specific wrong label) -- see the next file for the targeted variant |
| **Attack strength** | Weaker than iterative methods; often doesn't fully flip predictions with small epsilon on low-dimensional inputs, as shown in Section 7 |
| **Transferability** | Comparatively strong -- FGSM perturbations tend to transfer well to *other* models trained on similar data, because they exploit broad, shared linear directions in the loss landscape rather than overfitting to one specific model's quirks |
| **Requires** | White-box access to gradients (or a substitute/shadow model to approximate them) |

---

## 12. Security Angle

- **FGSM is the "hello world" of adversarial attacks, but it is not a toy.** Because it requires only one gradient computation, it is cheap enough to run as a routine robustness check against any model you have gradient access to (e.g., an internal model you're red-teaming, or an open-source model whose architecture and weights you can download).
- **Transferability is the practical black-box angle.** Even without direct API access to a target production model's internals, an attacker can train a *substitute model* on similar data, craft an FGSM perturbation against that substitute, and have a reasonable chance the same perturbation also fools the real target -- because FGSM tends to exploit broadly-shared vulnerable directions rather than target-specific quirks.
- **Defenses aimed specifically at FGSM (e.g., simple input smoothing, or training briefly on FGSM examples -- "FGSM adversarial training") are a common first line of defense, but are frequently insufficient against the stronger, iterative attacks covered later in this module.** If you find a target is robust to single-step FGSM, that does not imply it is robust to I-FGSM or DeepFool -- always escalate your testing.
- **Because FGSM commits to a full `epsilon`-sized step in every feature simultaneously, it is also a useful diagnostic:** if a target model is *not* fooled even by a full-strength FGSM step at a reasonably generous epsilon, that's a meaningfully stronger robustness signal than failing to fool it with a much weaker or partial perturbation.

---

## 13. Key Takeaways

- FGSM computes the loss gradient with respect to the **input** (not the model's weights) in a single forward+backward pass, then takes **one full-strength step** of size `epsilon` in the direction of the gradient's **sign**.
- Using the **sign** of the gradient (rather than its raw magnitude) is not a shortcut -- it is the exact mathematically optimal choice for maximizing the linear loss approximation within an **L-infinity** budget.
- The formula is: `x_adv = x + epsilon * sign(gradient_x(L(model(x), y_true)))`.
- The worked logistic-regression example showed a real, if modest, effect: confidence dropped from 0.852 to 0.802 with `epsilon = 0.1` on a 2-feature input -- and would drop further as `epsilon` increases or as the input's dimensionality grows (per the high-dimensional effects file).
- FGSM is fast, untargeted by default, uses the L-infinity norm, and tends to **transfer well** across different models, but is generally **weaker** than iterative methods on any single fixed target.
- FGSM is the direct computational payoff of everything covered in the Foundations section: norm constraints (why L-infinity implies "sign"), the local linearity assumption (why one linear step is trustworthy for small epsilon), and high-dimensional effects (why this cheap trick is so devastating on images).

---

*Next up: Targeted FGSM -- the variant where the attacker doesn't just want "any wrong answer," but a specific chosen wrong answer, and how that changes the direction of the gradient step.*
