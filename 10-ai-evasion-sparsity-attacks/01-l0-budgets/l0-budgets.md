# L0 Budgets

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - Sparsity Attacks | Section: L0 Budgets

---

## Table of Contents

1. [The Big Picture: Two Philosophies of Attack](#1-the-big-picture-two-philosophies-of-attack)
2. [What Is a "Norm," Really?](#2-what-is-a-norm-really)
3. [The L0 "Norm" -- Counting, Not Measuring](#3-the-l0-norm----counting-not-measuring)
4. [A Tiny Worked Example](#4-a-tiny-worked-example)
5. [Why L0 Minimization Is So Hard](#5-why-l0-minimization-is-so-hard)
6. [Visualizing the Search Space](#6-visualizing-the-search-space)
7. [L0 vs. L1 vs. L2 vs. L-infinity, Side by Side](#7-l0-vs-l1-vs-l2-vs-l-infinity-side-by-side)
8. [Why This Motivates Everything Else in This Module](#8-why-this-motivates-everything-else-in-this-module)
9. [Real-World Walkthrough -- Budgeting an Evasion Against a Malware Classifier](#9-real-world-walkthrough----budgeting-an-evasion-against-a-malware-classifier)
10. [Security Angle](#10-security-angle)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. The Big Picture: Two Philosophies of Attack

Before diving into any math, let's set up the mental model for this entire module, because it is the single most important idea to carry forward.

Imagine you want to fool an image classifier into thinking a photo of a "stop sign" is actually a "speed limit sign." There are two very different ways to do this:

- **Philosophy A -- "Change everything a tiny bit."** Nudge *every single pixel* in the image by a tiny, almost invisible amount. No individual pixel changes enough for a human to notice, but the combined effect across thousands of pixels pushes the model across its decision boundary. This is the approach used by the dense attacks you studied elsewhere in this course (FGSM, PGD, and friends), which are usually described using the **L2** (Euclidean "as the crow flies" distance) or **L-infinity** (the single largest change to any one pixel) norms.

- **Philosophy B -- "Change as few things as possible."** Instead of touching every pixel, find the *smallest possible set* of pixels (maybe just 3, maybe just 1) and change only those, potentially by a large amount each. Everything else in the image stays byte-for-byte identical.

**This module is entirely about Philosophy B.** We call this a **sparsity attack**, because the resulting perturbation (the "noise" you add) is *sparse* -- mostly zeros, with just a handful of non-zero entries.

```
DENSE ATTACK (Philosophy A -- "everything a tiny bit")
+---+---+---+---+---+---+---+---+
| . | . | . | . | . | . | . | . |   Every pixel shifts by a
+---+---+---+---+---+---+---+---+   small amount (the dots
| . | . | . | . | . | . | . | . |   represent "slightly
+---+---+---+---+---+---+---+---+   changed"). Invisible to
| . | . | . | . | . | . | . | . |   the eye, but the model
+---+---+---+---+---+---+---+---+   "feels" the accumulated
| . | . | . | . | . | . | . | . |   drift across all pixels.
+---+---+---+---+---+---+---+---+


SPARSE ATTACK (Philosophy B -- "as few as possible")
+---+---+---+---+---+---+---+---+
|   |   |   |   |   |   |   |   |   Almost every pixel is
+---+---+---+---+---+---+---+---+   untouched (blank = 0
|   |   | X |   |   |   |   |   |   change). Only 2 pixels
+---+---+---+---+---+---+---+---+   (marked X) are changed,
|   |   |   |   |   |   |   | X |   but each one might change
+---+---+---+---+---+---+---+---+   drastically (e.g. from
|   |   |   |   |   |   |   |   |   black to white).
+---+---+---+---+---+---+---+---+
```

Both attacks can achieve the exact same goal (fool the model), but they optimize for a different notion of "small." Dense attacks minimize *how far* you moved overall. Sparse attacks minimize *how many* things you touched at all. That distinction is what the "L0 norm" formalizes, and it's the foundation for every other technique in this module (L1 relaxations, saliency maps, EAD, FISTA, JSMA, single-pixel attacks).

---

## 2. What Is a "Norm," Really?

If you haven't seen the word "norm" before, here's the plain-English version before any formulas.

A **norm** is just a rule for turning a list of numbers (a vector) into a single number that represents "how big" that vector is. Think of it as a generalized ruler.

**Tiny numeric example.** Suppose our perturbation (the changes we make to an image) is the vector:

```
delta = [ 0.01, 0.00, -0.30, 0.00, 0.02 ]
```

This vector has 5 entries, one for each of 5 pixels we could have changed. Different norms answer different questions about this vector:

| Question | Norm | Plain English |
|---|---|---|
| "How many entries are nonzero?" | **L0** | Count of changed features |
| "What's the sum of absolute values?" | **L1** | Total amount of ink spilled |
| "What's the straight-line length?" | **L2** | Euclidean distance, like a ruler in n-dimensional space |
| "What's the single biggest entry?" | **L-infinity** | The worst single change |

For `delta = [0.01, 0.00, -0.30, 0.00, 0.02]`:

- **L0** = 3 (three nonzero entries: 0.01, -0.30, 0.02)
- **L1** = |0.01| + |0| + |-0.30| + |0| + |0.02| = 0.33
- **L2** = sqrt(0.01² + 0² + 0.30² + 0² + 0.02²) ≈ sqrt(0.0001 + 0.09 + 0.0004) ≈ 0.301
- **L-infinity** = max(|0.01|, 0, |-0.30|, 0, |0.02|) = 0.30

Notice how L0 completely ignores *magnitude* -- it doesn't care whether an entry is 0.001 or 1000, only whether it is exactly zero or not. That single property is what makes L0 so different (and so useful) for sparsity attacks.

---

## 3. The L0 "Norm" -- Counting, Not Measuring

Mathematically, the true definition of a norm requires certain properties (like scaling linearly: doubling every value should double the norm). The "L0 norm" technically violates one of those rules (it doesn't scale at all -- multiplying a nonzero value by 1000 doesn't change its L0 contribution), so purists call it a **pseudo-norm**. In practice, everyone in the adversarial ML literature just calls it "the L0 norm," so we will too.

**General formula:**

```
||delta||_0 = number of i such that delta_i != 0
```

In plain English: go through every entry of the perturbation vector, count how many are not exactly zero, and that count is your L0 value.

**Why this matters for attacks.** An L0-constrained attack says: "I am allowed to change at most *k* features/pixels, but I can change each of those k features by *any amount* (up to the valid pixel range, e.g. 0-255 or 0.0-1.0)." This is the mathematical way of writing "change as few things as possible."

```python
def l0_norm(delta):
    """Count how many entries of the perturbation are nonzero."""
    count = 0
    for value in delta:
        if value != 0:
            count += 1
    return count

# Example
delta = [0.01, 0.00, -0.30, 0.00, 0.02]
print(l0_norm(delta))   # -> 3
```

An **L0 budget** of `k` means the attacker's optimization problem looks like:

```
minimize      loss(x + delta)          # make the model misclassify
subject to    ||delta||_0 <= k          # touch at most k features
              x + delta stays a valid input (e.g. pixel values in [0, 255])
```

Where `loss(x + delta)` is a function measuring how far the model's prediction on the perturbed input `x + delta` is from what the attacker wants (e.g., how confidently it still says "stop sign" instead of "speed limit sign" -- the attacker wants this loss to be small).

---

## 4. A Tiny Worked Example

Let's make this completely concrete with a toy classifier that only looks at 4 features (imagine a tiny 2x2 grayscale image, pixels A, B, C, D, each between 0 and 1).

Suppose the model's decision rule (invented for teaching purposes) is:

```
predict "CAT" if   2*A + 0.1*B + 0.1*C + 3*D  > 2.5
predict "DOG" otherwise
```

Original image: `A=0.4, B=0.9, C=0.9, D=0.2`

Check: `2*(0.4) + 0.1*(0.9) + 0.1*(0.9) + 3*(0.2) = 0.8 + 0.09 + 0.09 + 0.6 = 1.58` → below 2.5 → predicted **DOG**.

We want it to say **CAT** instead. Let's compare two attack strategies:

**Dense strategy (small nudge everywhere):** raise every feature by 0.3.
`A=0.7, B=1.0(capped), C=1.0(capped), D=0.5`
Check: `2*(0.7) + 0.1*(1.0) + 0.1*(1.0) + 3*(0.5) = 1.4 + 0.1 + 0.1 + 1.5 = 3.1` → CAT. Success, but we touched all 4 features (L0 = 4).

**Sparse strategy (L0 budget = 1):** We're only allowed to change ONE feature. Which one gives the most "bang for the buck"? Looking at the coefficients (2, 0.1, 0.1, 3), feature `D` has the largest coefficient (3), so changing D moves the decision score the fastest per unit of change. Push `D` from 0.2 to 1.0 (max allowed), leave everything else untouched.

Check: `2*(0.4) + 0.1*(0.9) + 0.1*(0.9) + 3*(1.0) = 0.8 + 0.09 + 0.09 + 3.0 = 3.98` → CAT. Success, and we only touched **1 feature** (L0 = 1)!

This toy example already previews the next section's idea (saliency): the attacker looked at *which feature has the biggest effect on the decision* and spent the whole budget there. That's exactly the intuition behind saliency-based feature selection, covered later in this module.

---

## 5. Why L0 Minimization Is So Hard

Here is the crux of this section, and the reason the rest of the module exists.

**The naive way to solve an L0-constrained (or L0-minimizing) attack** would be: try every possible subset of pixels of size k, and for each subset, run an optimization to see if perturbing exactly those pixels can flip the classification. Keep the subset that works with the smallest total change (or the smallest k that works at all).

**The problem: combinatorial explosion.** For an image with `n` pixels, the number of ways to choose `k` of them to perturb is:

```
C(n, k) = n! / (k! * (n-k)!)
```

**Numeric gut-check.** A tiny 28x28 grayscale image (like MNIST digits) has n = 784 pixels. Suppose we only want to change k = 5 pixels:

```
C(784, 5) = 784! / (5! * 779!)  ≈ 3.7 * 10^12   (3.7 trillion combinations)
```

That's 3.7 trillion subsets to check -- for a tiny 28x28 image and only 5 changed pixels. A realistic photo (e.g., 224x224x3 color channels = 150,528 "pixels") makes this number so large it is meaningless to even write down. And for *each* subset, you'd still need to run an optimization (e.g., gradient descent) to find the best perturbation values, multiplying the cost further.

**Formally:** minimizing the L0 norm subject to a misclassification constraint (or vice versa) is known to be **NP-hard** in the general case, closely related to classic NP-hard problems like *sparse recovery* and *subset selection* in compressed sensing and statistics. "NP-hard" is a term from computer science meaning, informally: *no known algorithm can solve every instance of this problem quickly (in polynomial time) as the problem size grows, and most experts believe no such algorithm exists.* You don't need the formal complexity-theory proof to internalize the practical consequence: **brute-force L0 optimization does not scale**, even for small images.

```
     BRUTE FORCE SEARCH SPACE (n=784 pixels, k=5)

     Try subset {1,2,3,4,5}     -> optimize -> fail
     Try subset {1,2,3,4,6}     -> optimize -> fail
     Try subset {1,2,3,4,7}     -> optimize -> fail
     ...
     ... 3,700,000,000,000 subsets later ...
     ...
     Try subset {780,781,782,783,784} -> optimize -> maybe works

     This would take longer than a human lifetime on any
     realistic hardware, for a 28x28 image.
```

Because exact L0 optimization is intractable, every practical sparsity attack in the literature is really an approximation strategy that tries to get *close to* a minimal L0 solution without exhaustively searching. The remaining sections of this module are exactly that toolbox of approximations:

- **L1-induced sparsity** -- swap the hard L0 count for a "softer," optimization-friendly L1 penalty that tends to produce sparse results anyway (Section 2).
- **Saliency-based feature selection** -- use gradient information to *rank* features by importance instead of searching all subsets, and greedily pick the top-ranked ones (Section 3).
- **EAD and FISTA** -- efficient optimization machinery that combines L1 sparsity with good attack success rates (Sections 4-5).
- **JSMA** -- a greedy, saliency-driven algorithm that picks features one (or two) at a time (Section 6).
- **Single-pixel attacks** -- the extreme k=1 case, made feasible by clever search heuristics like differential evolution instead of brute force (Section 7).

---

## 6. Visualizing the Search Space

It helps to picture the difference between the *feasible region* of an L0-constrained problem versus an L1 or L2-constrained one.

For L1 and L2 constraints, the set of "allowed" perturbation vectors forms a smooth, **connected, convex shape** (a diamond for L1, a ball/sphere for L2 -- explored in detail in the next section). Convex shapes are wonderful for optimization: gradient descent can smoothly slide along the boundary and is guaranteed to find good (often globally optimal, for convex problems) solutions.

For an L0 constraint, the feasible region is **not smooth or connected at all**. It's a scattered collection of individual "axis-aligned" flat subspaces (e.g., "only pixel 3 and pixel 47 can move, everything else pinned to exactly zero"), and there is a different, separate one of these flat pieces for *every* possible subset of k pixels. There is no continuous path through this region -- moving from "perturb pixels {3, 47}" to "perturb pixels {3, 48}" is a discrete jump, not a small step.

```
L2 / L1 feasible region:            L0 feasible region:
(smooth, connected, one piece)      (disconnected islands, one
                                      island per subset of size k)

        _______                         .    .   .
       /       \                       .  .    .   .
      /         \        vs.          .   . .    .
      \         /                      .    . .   .
       \_______/                        .   .    .
                                     (each dot = a different
   Gradient descent can slide         choice of WHICH k pixels
   smoothly along this boundary.      to touch -- no smooth path
                                       between choices)
```

This is the geometric reason gradient-based optimization struggles directly with L0: gradients tell you which *direction* to nudge continuous values, but they say nothing about which *discrete subset* of pixels you should be allowed to touch at all.

---

## 7. L0 vs. L1 vs. L2 vs. L-infinity, Side by Side

| Norm | What It Measures | Typical Attack Goal | Optimization Difficulty | Module Coverage |
|---|---|---|---|---|
| **L0** | Number of changed features (count, ignores magnitude) | Change as *few* pixels/features as possible | NP-hard exactly; needs heuristics | This module (L0 budgets, JSMA, single-pixel) |
| **L1** | Sum of absolute changes (total "ink" used) | Sparse-*ish* changes, favors many-zero solutions | Convex; solvable efficiently (e.g. FISTA) | This module (L1 relaxation, EAD, FISTA) |
| **L2** | Euclidean/straight-line distance | Small *overall* change spread across many features | Convex; smooth, easy for gradient descent | Module 9 dense attacks (e.g. C&W L2) |
| **L-infinity** | The single largest change to any one feature | Every feature changes by at most epsilon | Convex; easy for gradient descent | Module 9 dense attacks (e.g. FGSM, PGD) |

**The key contrast to memorize:** Module 9's dense attacks (L2, L-infinity) treat "small" as "small in total magnitude, spread thin across everything." This module's sparsity attacks treat "small" as "touching almost nothing, even if what you do touch changes a lot." A single-pixel attack (Section 7) might change one pixel from black to pure white (a huge magnitude change) while having an L2 norm smaller than a dense attack that nudges every pixel by 1%.

---

## 8. Why This Motivates Everything Else in This Module

To summarize the causal chain that the rest of this module follows:

```
L0 is the "correct" way to measure sparsity
        |
        v
But minimizing L0 exactly is NP-hard (combinatorial explosion)
        |
        v
So we need APPROXIMATIONS that are tractable:
        |
        +--> Relax L0 to L1 (convex, solvable) ............ Section 2
        |
        +--> Use gradients to RANK features by importance
        |    instead of searching all subsets (saliency) ... Section 3
        |
        +--> Combine L1 + L2 penalties for a balanced,
        |    efficiently-solvable objective (EAD) .......... Section 4
        |
        +--> Use fast proximal-gradient solvers for the
        |    L1-regularized problem (FISTA) ................ Section 5
        |
        +--> Greedily pick top-saliency feature PAIRS,
        |    one step at a time (JSMA) ...................... Section 6
        |
        +--> Push the extreme case: what's the smallest k
             for which an attack still works? (single-pixel).. Section 7
```

Every subsequent section in this module is best understood as "a different answer to the question: given that we can't brute-force L0, what's a smart trick to get close to a sparse solution anyway?"

---

## 9. Real-World Walkthrough -- Budgeting an Evasion Against a Malware Classifier

Let's ground everything in this section with a complete, concrete (simplified) walkthrough, the same way Module 1 walked through a full spam-detection pipeline.

**Setup.** A security vendor ships a static malware classifier that scans a PE (Windows executable) file and extracts 12 binary features -- each one either "present" (1) or "absent" (0):

```
Feature vector for a real malicious sample "evil.exe":

| # | Feature                          | Value |
|---|-----------------------------------|-------|
| 1 | imports_CreateRemoteThread        | 1     |
| 2 | imports_VirtualAllocEx             | 1     |
| 3 | imports_WriteProcessMemory          | 1     |
| 4 | has_high_entropy_section (packed)   | 1     |
| 5 | imports_RegSetValueEx                | 1     |
| 6 | has_valid_digital_signature           | 0     |
| 7 | contains_string_"cmd.exe"              | 1     |
| 8 | contains_string_"powershell"            | 1     |
| 9 | section_count_over_6                     | 1     |
| 10| imports_InternetOpenUrlA                  | 1     |
| 11| icon_resource_present                      | 0     |
| 12| compile_timestamp_plausible                  | 1     |
```

The classifier's (simplified, invented for teaching) decision rule assigns a weight to each feature and flags "MALICIOUS" if the weighted sum exceeds a threshold of 4.0:

```
weights = [1.4, 1.3, 1.1, 0.9, 0.6, -0.5, 0.4, 0.4, 0.3, 0.3, -0.2, 0.1]

score(x) = sum(weight_i * x_i)
predict "MALICIOUS" if score(x) > 4.0, else "BENIGN"
```

**Current score** for evil.exe (all the 1-valued features contributing, feature 6 and 11 subtracting since they're 0 and have negative weight anyway so contribute 0 right now):

```
score = 1.4 + 1.3 + 1.1 + 0.9 + 0.6 + 0 + 0.4 + 0.4 + 0.3 + 0.3 + 0 + 0.1 = 6.7
```

6.7 > 4.0, so the file is correctly flagged **MALICIOUS**.

**The attacker's constraint.** Unlike an image, this attacker cannot make "tiny nudges" to every feature -- these are discrete, binary, and each one has a real-world *cost*:

- Removing `imports_CreateRemoteThread` or `imports_WriteProcessMemory` might **break the malware's actual functionality** (it needs those API calls to do its job). High cost, likely infeasible.
- Turning on `has_valid_digital_signature` requires obtaining a code-signing certificate. High cost, slow, and not guaranteed to work if the cert gets revoked or flagged.
- Adding a plausible-looking `icon_resource_present` (feature 11, currently 0, weight -0.2) is nearly free -- it's cosmetic.
- Removing the string `"powershell"` (feature 8, weight 0.4) by obfuscating it is cheap and low-risk.
- Reducing `section_count_over_6` (feature 9) by merging PE sections is moderate effort but achievable with standard packer tooling.

**This is exactly an L0-budgeted attack problem**, except the "budget" isn't just a count -- it's a count *weighted by real-world feasibility*. A rational attacker restricts their search to the subset of features that are cheap to flip, then asks: "what is the smallest number of *cheap* features I need to flip to cross back under the 4.0 threshold?"

**Trying k=1 (cheapest available flip):** Turn on feature 11 (icon, weight -0.2, free) -- new score = 6.7 - 0.2 = 6.5. Still way above 4.0. **Fails.**

**Trying k=2:** Add feature 11 AND remove feature 8 (drop the "powershell" string, weight 0.4) -- new score = 6.7 - 0.2 - 0.4 = 6.1. Still fails.

**Trying k=3:** Also merge sections to flip feature 9 off (weight 0.3) -- new score = 6.1 - 0.3 = 5.8. Still fails.

**Trying k=4, adding the digital signature (feature 6, weight -0.5, but assume the attacker manages to get one -- expensive but let's see the effect):** new score = 5.8 - (0 - (-0.5)) ... careful: feature 6 flips from 0 to 1, and its weight is -0.5, so this SUBTRACTS 0.5 from the score: new score = 5.8 - 0.5 = 5.3. Still fails, and this was the expensive one.

Even after touching 4 of the 5 "cheap-ish" features, the score (5.3) remains well above the 4.0 threshold, because the dominant weight is concentrated in the "expensive to remove" functional features (1-5). This walkthrough demonstrates something important that a pure math exercise can hide: **an L0 budget attack can fail not because the search algorithm is bad, but because the cheaply-modifiable features simply don't carry enough weight** to cross the boundary -- exactly the "evenly spread vs. concentrated sensitivity" lesson explored more rigorously in Section 5 of the Single-Pixel section (Section 7 of this module). This is precisely why real attackers combine feature-cost-aware sparsity search (this section) with saliency ranking (Section 3) -- to identify, among all *feasible* edits, which ones give the most score reduction per unit of cost/risk.

## 10. Security Angle

**Why attackers care about sparsity specifically:**

- **Stealth and plausibility.** A perturbation that changes 3 out of 10,000 features is much harder for a human reviewer or an anomaly-detection system to notice than one that subtly shifts all 10,000. If a malware analyst diffs a "modified" file against the original, fewer changed bytes means a smaller, less suspicious diff.

- **Real-world feature constraints.** In domains like malware or spam classification, features are often things like "presence of API call X" or "keyword Y appears." These are typically **binary or categorical**, not continuous pixel intensities. You can't apply a "tiny L-infinity nudge" to a binary feature -- it's either present or absent. Sparsity attacks (flip only a handful of discrete features) are often the *only* physically meaningful attack model in these domains.

- **Cost of modification.** Some features are cheap to change (e.g., adding a benign-looking string to a file) and others are expensive or risky (e.g., removing a core malicious payload, which might break functionality). An attacker with an L0 budget is really asking: "what is the minimum number of cheap edits I need to make to flip the classifier's decision, while preserving the malware's actual function?"

- **Hardware/physical attacks.** Single-pixel or few-pixel changes are directly relevant to physical-world attacks: a single sticker on a stop sign, one glitched pixel from a faulty camera sensor, or one manipulated byte in a network packet header. Section 7 explores this extreme case directly.

---

## 11. Key Takeaways

- The **L0 "norm"** counts how many entries of a perturbation vector are nonzero. It measures *how many* things changed, completely ignoring *how much* each one changed.
- **Sparsity attacks** minimize L0 (or constrain it to a budget `k`): touch as few features/pixels as possible, even if each change is large. This is the opposite philosophy of dense L2/L-infinity attacks from Module 9, which spread tiny changes across everything.
- Exact L0 minimization is **combinatorially explosive** -- checking `C(n, k)` subsets becomes computationally infeasible even for small images (3.7 trillion subsets for a 28x28 image with k=5). It is classified as **NP-hard**.
- The L0 feasible region is **disconnected** (scattered discrete "islands," one per subset choice), unlike the smooth convex shapes of L1/L2/L-infinity constraints, which is exactly why standard gradient descent cannot solve L0 problems directly.
- This intractability is the **motivation** for every technique later in this module: L1 relaxation, saliency-based ranking, EAD, FISTA, and JSMA are all practical workarounds that approximate a sparse solution without brute-forcing every subset.
- In security terms, sparsity matters because many real feature spaces (malware, spam, network traffic) are **discrete/binary**, and fewer changed features means a **stealthier, cheaper, less detectable** attack.

---

*Next up: L1-Induced Sparsity -- how replacing the "hard" L0 count with the "soft," convex L1 penalty gives us a tractable optimization problem that still tends to produce sparse solutions, setting the stage for EAD and FISTA.*
