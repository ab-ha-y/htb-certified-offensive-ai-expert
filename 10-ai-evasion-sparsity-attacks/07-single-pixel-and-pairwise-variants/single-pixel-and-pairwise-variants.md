# Single-Pixel and Pairwise Variants

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - Sparsity Attacks | Section: Single-Pixel and Pairwise Variants

---

## Table of Contents

1. [The Analogy: The Last Grain of Sand](#1-the-analogy-the-last-grain-of-sand)
2. [Recap: The Extreme End of the L0 Spectrum](#2-recap-the-extreme-end-of-the-l0-spectrum)
3. [Why Is This Even Possible? Intuition #1 -- High Dimensionality](#3-why-is-this-even-possible-intuition-1----high-dimensionality)
4. [Why Is This Even Possible? Intuition #2 -- Proximity to Decision Boundaries](#4-why-is-this-even-possible-intuition-2----proximity-to-decision-boundaries)
5. [A Tiny Worked Numeric Example](#5-a-tiny-worked-numeric-example)
6. [Why Gradient Methods Struggle Here: The Black-Box Problem](#6-why-gradient-methods-struggle-here-the-black-box-problem)
7. [Differential Evolution: A Gradient-Free Search](#7-differential-evolution-a-gradient-free-search)
8. [Pseudocode: One-Pixel Attack via Differential Evolution](#8-pseudocode-one-pixel-attack-via-differential-evolution)
9. [Pairwise and Few-Pixel Variants](#9-pairwise-and-few-pixel-variants)
10. [What This Demonstrates About Model Fragility](#10-what-this-demonstrates-about-model-fragility)
11. [Comparing All Sparsity Techniques in This Module](#11-comparing-all-sparsity-techniques-in-this-module)
12. [Real-World Walkthrough -- A Full Differential Evolution Trace](#12-real-world-walkthrough----a-full-differential-evolution-trace)
13. [Security Angle](#13-security-angle)
14. [Key Takeaways](#14-key-takeaways)

---

## 1. The Analogy: The Last Grain of Sand

There's an old philosophical puzzle called the **Sorites paradox** (the "heap paradox"): if you have a heap of sand and remove one grain at a time, at what exact point does it stop being a "heap"? No single grain seems responsible, yet eventually the heap is gone.

Adversarial single-pixel attacks pose a strange inverse version of this question for neural networks: if you have a photo the model confidently and correctly calls a "horse," and you change **exactly one pixel** -- one thirty-two-thousandth of the image, say, in a 32x32 photo -- can that alone make the model call it a "truck" instead? Astonishingly, for many real, undefended neural network image classifiers, the answer is **yes**, sometimes with over 70% success rate across a test set, and often with high confidence in the wrong answer.

This section is the extreme endpoint of everything this module has built toward. Section 1 introduced the L0 budget `k`. Every subsequent section (L1 relaxation, saliency, EAD, FISTA, JSMA) has been a strategy for approximately minimizing L0. This section asks: **what happens when we push k all the way down to its absolute floor -- k=1 or k=2?**

```
   L0 BUDGET SPECTRUM, FROM DENSE TO MOST EXTREME:

   k = ALL PIXELS       k = moderate          k = a handful       k = 1
   (dense attack,       (EAD/JSMA typical      (JSMA/EAD           (single-pixel
    Module 9 style)      result)                pushing hard)       attack, this
                                                                      section)
        |                     |                       |                  |
   +---+---+---+          +---+---+---+          +---+---+---+     +---+---+---+
   | . | . | . |          |   | X |   |          |   | X |   |     |   |   |   |
   +---+---+---+   -->    +---+---+---+   -->    +---+---+---+ --> +---+---+---+
   | . | . | . |          |   |   | X |          |   |   |   |     |   | X |   |
   +---+---+---+          +---+---+---+          +---+---+---+     +---+---+---+
```

---

## 2. Recap: The Extreme End of the L0 Spectrum

Recall from Section 1 that an L0 budget of `k` means "the attacker may change at most `k` features, of any magnitude." A **single-pixel attack** is simply the special case `k = 1`: exactly one pixel in the entire image is allowed to change, and it can change to *any* valid color/intensity value.

A **pairwise (or few-pixel) attack** relaxes this slightly to `k = 2` or `k = a small handful` (e.g., 3-5), trading a bit of extremeness for a meaningfully higher chance of success, since there's more "room" to find an effective combination.

The seminal work here is Su, Vargas, and Sakurai's 2019 paper "One Pixel Attack for Fooling Deep Neural Networks," which demonstrated exactly this: modifying just one pixel (out of 1024 total in a 32x32 CIFAR-10 image) was enough to fool a range of trained convolutional neural networks a majority of the time, in a fully **black-box** setting (no gradient access needed at all -- more on why that matters in Sections 6-7).

---

## 3. Why Is This Even Possible? Intuition #1 -- High Dimensionality

**Plain-English setup.** A 32x32 color image has `32 * 32 * 3 = 3072` numbers describing it (3 color channels per pixel). Each one of those 3072 numbers is one "dimension" the model's decision function lives in. That's a lot of dimensions for a human to intuit about -- we're used to thinking in 2D or 3D, but 3072-dimensional space behaves in ways that violate everyday intuition.

**Why high dimensionality helps the attacker.** In very high-dimensional spaces, decision boundaries between classes can be extremely close to almost every "normal" data point, along *some* direction -- even if the boundary is far away along most other directions. Think of it this way: a model's decision boundary doesn't need to be uniformly distant from a data point in every one of 3072 directions; it just needs to be far in the directions that matter for "normal" images to look correctly classified, while potentially being razor-thin in some obscure combination of a few specific pixel-channel values that never naturally occurs in real photos (because real photos are constrained to a much lower-dimensional "natural image manifold" -- a technical way of saying "real photos only occupy a tiny, structured sliver of the full 3072-dimensional space of all possible pixel combinations").

```
    LOW-DIMENSIONAL INTUITION (misleading!)          HIGH-DIMENSIONAL REALITY

    In 2D, the boundary between "cat"                In 3072D, the boundary might
    and "dog" regions looks like it needs            be very far in ~3000 of the
    a substantial push to cross:                      directions, but VERY close
                                                        in just 1-2 specific pixel-
        CAT region  |  DOG region                      channel directions that
                     |                                  happen to matter a lot to
             x ------|-----> (needs a big push          this specific trained model
                      to cross in 2D)                    (but rarely occur/matter
                                                          in natural images)
```

This isn't a proof, just an intuition pump -- but it captures the core reason single-pixel attacks are *plausible* at all: with thousands of dimensions, the attacker only needs to find the *one* (or few) directions where the boundary happens to be unusually close, out of thousands of candidate directions to search.

---

## 4. Why Is This Even Possible? Intuition #2 -- Proximity to Decision Boundaries

Complementary to the dimensionality argument: neural networks trained via standard supervised learning (Module 1's ML pipeline) are optimized purely to get the *training and test accuracy* right -- nothing in typical training explicitly rewards the model for having decision boundaries that are *uniformly far* from every real data point in *every* direction of the input space (that would be an explicit robustness objective, which standard training does not include unless deliberately added, e.g., via adversarial training).

As a result, trained models often have decision boundaries that snake very close to many real data points along at least a few directions -- not because the model is "bad" by ordinary accuracy standards, but because ordinary accuracy standards never asked the model to avoid this. A useful mental picture: the model's decision regions for "horse" and "truck" might be shaped like two puzzle pieces with a very jagged, close-fitting boundary in a few spots, even though the bulk of each region is comfortably far from the boundary.

```
     A "puzzle piece" decision boundary in a few dimensions:

     HORSE region                    TRUCK region
     .................  \  /  .......................
     .................   \/   .......................
     .................   /\   .......................  <-- boundary snakes
     .................  /  \  .......................      very close to
                                                              some real images
                                                              in a few spots

     A correctly classified "horse" image might sit RIGHT
     next to this jagged boundary along the "pixel #547,
     blue channel" direction specifically, even while being
     safely far from the boundary in almost all other
     directions.
```

Single-pixel search is, in effect, a fast way of *probing* for exactly one of these close-boundary spots, without needing to understand the boundary's full shape.

---

## 5. A Tiny Worked Numeric Example

Let's build the smallest possible illustration using our familiar toy linear classifier, extended slightly to show why "spend the whole budget on the single most sensitive feature" (k=1) can sometimes succeed where you might not expect it to.

Recall our classifier from Section 1: `score(x) = 2*A + 0.1*B + 0.1*C + 3*D`, predicting CAT if `score > 2.5`.

We already showed that a *linear* model with these particular coefficients allows a k=1 attack (change only D) to succeed, precisely because D's coefficient (3.0) is large enough that pushing it alone crosses the threshold.

**Now let's see when k=1 would fail**, to understand the boundary case. Suppose instead the coefficients were much more evenly spread: `score(x) = 0.7*A + 0.7*B + 0.7*C + 0.7*D`, with the same threshold (predict CAT if `score > 2.5`), and the same starting point `A=0.4, B=0.9, C=0.9, D=0.2` (score = `0.7*(0.4+0.9+0.9+0.2) = 0.7*2.4 = 1.68`).

If we push only D to its max (1.0): `score = 0.7*(0.4+0.9+0.9+1.0) = 0.7*3.2 = 2.24` -- still below 2.5. **k=1 fails** with these evenly-spread coefficients, no matter which single feature we max out (by symmetry, they're all equally weak).

This tiny example reveals the general principle: **single-pixel/single-feature attacks succeed when the model's sensitivity is highly concentrated in a small number of directions** (like our first example, where D's coefficient dominated), and they fail when sensitivity is spread evenly (like our second example). Real, high-dimensional neural networks often (not always) exhibit pockets of exactly this kind of concentrated sensitivity in at least a few input dimensions per test image, especially images near their decision boundary to begin with -- which is why one-pixel attacks succeed on a meaningful fraction, though not all, of test images.

---

## 6. Why Gradient Methods Struggle Here: The Black-Box Problem

You might expect: "just use the saliency-based greedy method from Section 3 or JSMA (Section 6), pick the single highest-saliency pixel, and push it to the extreme." This does work sometimes, but the original single-pixel attack research deliberately avoided relying on gradients at all, for two important reasons:

1. **True black-box realism.** In many real attack scenarios (e.g., attacking a remote ML API you don't control), you genuinely don't have access to the model's internals or its gradients -- you can only submit inputs and observe outputs (predicted label, and sometimes confidence scores). A gradient-dependent method like JSMA or EAD simply cannot be run at all in this setting.

2. **The single best pixel by gradient magnitude isn't necessarily the single best pixel for actually flipping the class.** Gradients (as emphasized repeatedly in Sections 3 and 6) are a **local, linear approximation** of the model's behavior right at the current input. Pushing one pixel all the way from, say, 0.5 to 1.0 is a large, non-local change -- far outside the tiny neighborhood where the linear gradient approximation is trustworthy. A pixel with a modest gradient right now might, after being pushed to an extreme value, produce a much bigger effect than the gradient predicted (or a much smaller one) because the model's true behavior is nonlinear over that larger range.

Because of both points, single-pixel attack research turned to **gradient-free (black-box) optimization** methods that only need to *query* the model (submit an input, observe the output) -- no internal access required at all.

---

## 7. Differential Evolution: A Gradient-Free Search

**Differential Evolution (DE)** is a general-purpose, population-based, gradient-free optimization algorithm, originally developed for optimizing arbitrary functions where you can't (or don't want to) compute derivatives. Here's the plain-English mechanics, before any specifics of the attack:

1. **Maintain a population** of candidate solutions (here: candidate `(x_coord, y_coord, red, green, blue)` tuples describing "which pixel, and what new color, to try"). Start with a batch of random candidates.
2. **Each generation (iteration):** for every candidate, create a new "mutant" candidate by combining it with a few *other* randomly chosen candidates from the population (e.g., taking their differences and adding a scaled version to the original -- this "difference vector" step is where the algorithm's name comes from).
3. **Evaluate** each mutant by actually querying the target model: apply the proposed pixel change to the image, and check how much it reduced the confidence in the true label (or increased confidence in the target label). This model query is the *only* thing DE needs from the model -- no gradients, no internals, just a fitness score.
4. **Selection:** if the mutant scores better than the original candidate it was derived from, it replaces that candidate in the population for the next generation. Otherwise, the original candidate survives.
5. **Repeat** for many generations. Over time, the population converges toward candidates that are highly effective at flipping the model's prediction.

```
    DIFFERENTIAL EVOLUTION, ONE GENERATION:

    Population: [ (x1,y1,r1,g1,b1), (x2,y2,r2,g2,b2), (x3,y3,r3,g3,b3), ... ]
                        |
                        v
    For each candidate: combine with 2-3 random OTHER candidates
    to produce a "mutant" candidate
                        |
                        v
    QUERY THE MODEL with (original image + mutant's pixel change)
    Get back: confidence in true label (want this LOW)
                        |
                        v
    If mutant beats original candidate --> replace it
    Else --> keep original
                        |
                        v
    Repeat for many generations --> population converges toward
    highly effective single-pixel changes
```

This is a **query-efficient** approach: the 2019 paper reported successful attacks using on the order of a few hundred to a few thousand model queries per image -- far more than a single gradient computation (JSMA/EAD need far fewer total forward/backward passes since they have direct access to informative gradients), but entirely feasible against a real black-box API, and dramatically cheaper than exhaustively trying every possible pixel/color combination (recall from Section 1: a 32x32x3 image has `32*32=1024` possible pixel positions, times 256^3 possible RGB colors per pixel -- brute force is obviously infeasible).

---

## 8. Pseudocode: One-Pixel Attack via Differential Evolution

```python
def one_pixel_attack(model, image, true_label, population_size=400,
                      max_generations=100):
    """
    Simplified single-pixel attack via differential evolution.
    Each candidate is (x, y, r, g, b): which pixel to change, and to what color.
    """
    height, width, channels = image.shape

    def random_candidate():
        x = random_int(0, width - 1)
        y = random_int(0, height - 1)
        r, g, b = random_int(0, 255), random_int(0, 255), random_int(0, 255)
        return [x, y, r, g, b]

    def fitness(candidate):
        # Apply the proposed single-pixel change to a COPY of the image
        modified = copy(image)
        x, y, r, g, b = candidate
        modified[y][x] = [r, g, b]

        # QUERY the model -- this is the only information we need
        confidence = model.predict_confidence(modified, true_label)
        return confidence   # LOWER is better for the attacker

    population = [random_candidate() for _ in range(population_size)]

    for generation in range(max_generations):
        new_population = []
        for candidate in population:
            # Pick 2 other random candidates to build a "mutant"
            a, b_other = random_choice(population, k=2)
            mutant = mutate(candidate, a, b_other)  # differential evolution step
            mutant = clip_to_valid_ranges(mutant, width, height)

            if fitness(mutant) < fitness(candidate):
                new_population.append(mutant)
            else:
                new_population.append(candidate)
        population = new_population

        # Early stop if any candidate already flipped the prediction
        best = min(population, key=fitness)
        if model.predict_label(apply(image, best)) != true_label:
            return best, True   # success!

    best = min(population, key=fitness)
    return best, model.predict_label(apply(image, best)) != true_label
```

---

## 9. Pairwise and Few-Pixel Variants

The exact same differential-evolution search generalizes directly to `k=2` (pairwise) or `k=3,4,5...` (few-pixel) attacks: simply expand each candidate from one `(x, y, r, g, b)` tuple to `k` of them, e.g. `[(x1,y1,r1,g1,b1), (x2,y2,r2,g2,b2)]` for k=2. The search space grows correspondingly larger (more parameters to optimize per candidate), but the DE mechanics stay identical.

**Practical tradeoff observed in the literature:**

| Budget (k) | Success Rate (typical) | Search Difficulty | Perceptibility |
|---|---|---|---|
| k=1 (single pixel) | Moderate (varies widely by model/dataset, often 20-70% on small images like CIFAR-10) | Hardest to find (very few valid solutions exist) | Extremely low -- often invisible even when pointed out |
| k=2-3 (pairwise/few-pixel) | Meaningfully higher | Easier (more "room" in the search space to find a working combination) | Still very low, but occasionally noticeable on close inspection |
| k=5+ | Approaches JSMA/EAD-level success rates | Comparable to greedy/optimization methods | Starts to overlap with what saliency-guided methods achieve more efficiently |

This table also explains *why* JSMA and EAD (which use gradient information to guide the search intelligently) are generally preferred once the budget rises even slightly above 1-2 -- pure black-box search (DE) is most valuable specifically at the k=1 extreme, where gradient-based local approximations are least trustworthy (per Section 6) and the "surgical precision vs. exhaustive querying" tradeoff most favors querying.

---

## 10. What This Demonstrates About Model Fragility

Stepping back, the significance of single-pixel attacks isn't really about pixels at all -- it's a **stress test that reveals something fundamental and slightly uncomfortable about how standard neural network training works**:

- **Accuracy is not the same as robustness.** A model can achieve 95%+ test accuracy (by the standard ML pipeline metrics from Module 1) while still having decision boundaries that pass unnervingly close to a large fraction of correctly-classified test images, along at least one obscure direction each. High accuracy tells you the model is right *most of the time on natural data*; it tells you nothing about how far the nearest "wrong-answer" input is in the full space of all possible inputs.

- **The attack surface is the entire input space, not just "realistic-looking" perturbations.** Standard training never explicitly checks "is there some input arbitrarily close to this one, even an unrealistic one that no photographer would ever produce, that the model gets wrong?" Single-pixel attacks are proof that the answer is very often yes.

- **It's a diagnostic tool, not just an attack.** Researchers use one-pixel/few-pixel success rates as a **quantitative robustness metric**: models trained with adversarial training or other robustness-improving techniques (topics for other parts of this course) generally show *lower* success rates for single-pixel and few-pixel attacks, providing an empirical, testable signal of whether a defense is actually improving robustness or just adding superficial obstacles.

---

## 11. Comparing All Sparsity Techniques in This Module

To tie the whole module together, here is a single summary table spanning every technique covered, contrasted once more against Module 9's dense attacks:

| Technique | Norm Focus | Access Needed | Selection Method | Typical Sparsity | Optimization Guarantees |
|---|---|---|---|---|---|
| **Module 9 dense attacks (FGSM/PGD/C&W-L2)** | L2 / L-infinity | White-box (gradients) | Uniform nudge across all features | None -- every feature typically changes | Strong (convex or near-convex, well-studied) |
| **L0 exhaustive search (Section 1)** | L0 (exact) | White-box or black-box | Brute-force subset enumeration | Perfectly minimal, if it ever finishes | Guaranteed optimal, but computationally infeasible |
| **L1 relaxation (Section 2)** | L1 (proxy for L0) | White-box (gradients) | Convex optimization (soft-thresholding) | Approximately sparse | Strong (convex) |
| **Saliency greedy selection (Section 3)** | ~L0 (heuristic) | White-box (gradients) | Rank by gradient magnitude, greedy pick | Sparse, budget-controlled | None formal; heuristic |
| **EAD (Section 4)** | L1 + L2 (elastic net) | White-box (gradients) | Convex optimization (FISTA) | Sparse, damped magnitude | Strong (convex, FISTA-optimal rate) |
| **FISTA (Section 5)** | (solver, not a norm itself) | White-box (gradients) | Accelerated proximal gradient descent | N/A -- it's the engine, not the attack | Provably optimal convergence rate for its problem class |
| **JSMA (Section 6)** | ~L0 (heuristic, pairwise) | White-box (Jacobian) | Greedy pairwise saliency ranking | Very sparse | None formal; heuristic |
| **Single-pixel/few-pixel (Section 7)** | L0 (extreme, k=1-5) | Black-box (queries only) | Differential evolution (gradient-free) | Most extreme sparsity possible | None formal; heuristic, stochastic search |

---

## 12. Real-World Walkthrough -- A Full Differential Evolution Trace

Let's trace a small, fully worked-through run of differential evolution (Section 7) against a toy black-box image classifier, so the "population evolves toward success" story from Section 7 becomes concrete rather than abstract.

**Setup.** A tiny 3x3 grayscale "image" classifier (9 pixels, values in `[0, 1]`) predicts "A" or "B." We are attacking it as a pure black box: we can only submit a modified image and read back a confidence score for the true label "A" (lower is better for us). We do not have gradients. Our budget is k=1 (single pixel).

**Population initialization.** We start with a small population of 4 random candidates (a real attack would use hundreds, we use 4 here purely for a traceable-by-hand example), each describing "which of the 9 pixel positions to change, and to what value":

```
Candidate 1: (position=2, value=0.9)   fitness (confidence in "A") = 0.81
Candidate 2: (position=5, value=0.1)   fitness = 0.77
Candidate 3: (position=7, value=0.6)   fitness = 0.85
Candidate 4: (position=0, value=0.3)   fitness = 0.90
```

(Recall: lower fitness is better for the attacker -- it means the model is now less confident in the correct label "A.")

**Generation 1 -- mutation.** For Candidate 1 `(2, 0.9)`, differential evolution picks two other random candidates (say Candidates 3 and 4) and creates a mutant by nudging Candidate 1's value using their *difference*:

```
mutant_value = candidate1.value + F * (candidate3.value - candidate4.value)
             = 0.9 + 0.5 * (0.6 - 0.3)     [F=0.5 is a standard scaling factor]
             = 0.9 + 0.15 = 1.05  -->  clipped to 1.0

mutant position: kept the same as candidate1's position (2), or occasionally
                  also perturbed, depending on the specific DE variant used

Mutant A: (position=2, value=1.0)
```

**Querying the model with Mutant A:** submit the image with pixel 2 set to 1.0, read back confidence in "A": suppose it returns **0.62** (lower than Candidate 1's original 0.81). Mutant A **wins** -- it replaces Candidate 1 in the population for the next generation.

We repeat this same mutate-and-compare process for Candidates 2, 3, and 4 (each combined with two other randomly chosen population members), and suppose the results are:

```
Candidate 2 (5, 0.1) vs its mutant (5, 0.05):        mutant fitness 0.80 > 0.77 original wins, keep original
Candidate 3 (7, 0.6) vs its mutant (7, 0.95):         mutant fitness 0.55 < 0.85, MUTANT WINS
Candidate 4 (0, 0.3) vs its mutant (3, 0.4):          mutant fitness 0.88 < 0.90, MUTANT WINS (barely)
```

**Population after Generation 1:**

```
Candidate 1: (position=2, value=1.0)   fitness = 0.62   (improved from 0.81)
Candidate 2: (position=5, value=0.1)   fitness = 0.77   (unchanged, mutant lost)
Candidate 3: (position=7, value=0.95)  fitness = 0.55   (improved from 0.85)
Candidate 4: (position=3, value=0.4)   fitness = 0.88   (improved from 0.90)
```

The population's best candidate is now Candidate 3, at fitness 0.55 -- getting closer to flipping the prediction, but not yet below whatever threshold separates "A" from "B" (suppose that threshold is 0.5).

**Generation 2 (abbreviated).** The process repeats: each candidate is combined with two random others to form a new mutant, evaluated via a model query, and kept only if it improves on the parent. Suppose after this generation, Candidate 3's lineage has evolved to `(position=7, value=1.0)` with fitness **0.42** -- now below the 0.5 threshold. **The model's prediction flips to "B."** The attack has succeeded, having changed exactly one pixel (position 7, from its original value to 1.0), discovered entirely through black-box queries with zero gradient access.

**Tallying the query cost.** Across the 2 generations traced above, we made roughly `4 candidates x 2 generations = 8` model queries (a real run would use far more candidates and generations, typically hundreds to a few thousand total queries, matching the figures cited in Section 7) -- still dramatically cheaper than the brute-force alternative of trying all `9 positions x 256 possible intensity values = 2304` combinations exhaustively (and that's just for a tiny 3x3 image; a real 32x32x3 image's brute-force space, per Section 1's combinatorics, would be astronomically larger).

This walkthrough demonstrates the full DE loop end to end: population initialization, mutation via scaled differences between random population members, greedy replacement based purely on black-box fitness queries, and convergence toward a successful single-pixel adversarial example -- all without a single gradient computation, in contrast to every other technique in this module.

## 13. Security Angle

- **Black-box realism for red team engagements.** Single-pixel/few-pixel attacks via differential evolution are directly applicable when attacking a real deployed ML API where you have no internal access -- exactly the situation a red team usually faces against a client's production system. This makes DE-based sparse search one of the most practically deployable techniques in this entire module for realistic external engagements, compared to the white-box (gradient-requiring) methods in Sections 2-6.

- **Physical-world plausibility.** A single manipulated pixel maps conceptually to real physical-world tampering scenarios: one glitched/stuck pixel from a faulty or tampered camera sensor, one adversarial sticker covering a tiny image region, or one corrupted byte in a transmitted image file. Demonstrating that a single such minimal change can flip a safety-critical classifier (e.g., in an autonomous vehicle's traffic-sign recognition) is a highly compelling way to communicate risk to non-technical stakeholders.

- **Extremely difficult to detect via simple integrity checks.** A single-pixel change (or a handful) will pass essentially any coarse image-integrity check that isn't pixel-exact (e.g., perceptual hashing algorithms often tolerate small localized changes by design, since they're meant to be robust to minor recompression artifacts). This means single-pixel evasion can potentially slip past defenses designed to catch "obviously tampered" images while still flipping the underlying classifier's decision.

- **Robustness benchmarking for defenders.** As covered in Section 10, the success rate of single/few-pixel attacks against a model is a useful, standardized robustness metric. When evaluating a defended model (or advising a client on one), running this attack family provides a concrete, reproducible number ("18% of test images can be flipped by changing just 1 pixel") that is far more actionable than a vague claim of "the model is robust."

---

## 14. Key Takeaways

- **Single-pixel attacks** are the extreme endpoint of the L0 budget spectrum introduced in Section 1: `k=1`, sometimes `k=2-5` for pairwise/few-pixel variants.
- Two complementary intuitions explain why this is even possible: **(1) high dimensionality** means decision boundaries only need to be close in a *few* of thousands of directions, not all of them; **(2) standard training never explicitly enforces uniform distance from decision boundaries**, so boundaries can snake unnervingly close to correctly-classified points along at least a few directions.
- A tiny worked example showed the general principle directly: single-feature attacks succeed when a model's sensitivity is **concentrated** in one dominant direction, and fail when sensitivity is **evenly spread**.
- Gradient-based methods (JSMA, EAD) are often avoided for the k=1 extreme because (a) true black-box scenarios have no gradient access at all, and (b) pushing a single feature all the way to an extreme value is too large a jump for the *local, linear* gradient approximation to remain trustworthy.
- **Differential evolution** is the standard gradient-free, query-based search technique used for practical single-pixel/few-pixel attacks: a population of candidate pixel-changes evolves over generations, using only black-box model queries (no internals needed) as its fitness signal.
- Single-pixel attack success rates serve as a genuine **robustness diagnostic**: they reveal that high test accuracy and true robustness are different properties, and they give defenders a concrete, reproducible metric for evaluating defenses.
- Across this entire module, the throughline is: **L0 is the "true" sparsity measure but is NP-hard to optimize directly (Section 1); every other technique (L1 relaxation, saliency, EAD, FISTA, JSMA, and finally black-box single-pixel search) is a different practical strategy for approximating a minimal-L0 solution**, in contrast to Module 9's dense attacks, which never try to minimize the *number* of changed features at all.

---

*This concludes Module 10: AI Evasion - Sparsity Attacks. Next up: continue to whichever module in your study plan covers black-box and query-based attack methods in more depth, or proceed to the Skills Assessment for this module.*
