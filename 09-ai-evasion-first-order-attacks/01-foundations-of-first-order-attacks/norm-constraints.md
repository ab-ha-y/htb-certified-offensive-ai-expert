# Norm Constraints: L0, L1, L2, and L-infinity

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - First-Order Attacks | Section: Foundations of First-Order Attacks

---

## Table of Contents

1. [What Is a Perturbation?](#1-what-is-a-perturbation)
2. [Why We Need to Measure "How Big" a Perturbation Is](#2-why-we-need-to-measure-how-big-a-perturbation-is)
3. [What Is a Norm? (Plain English First)](#3-what-is-a-norm-plain-english-first)
4. [The Four Norms Attackers Care About](#4-the-four-norms-attackers-care-about)
5. [Worked Numeric Example: One Perturbation, Four Norms](#5-worked-numeric-example-one-perturbation-four-norms)
6. [Visualizing Norm Balls](#6-visualizing-norm-balls)
7. [Choosing a Norm: What It Means for an Attack](#7-choosing-a-norm-what-it-means-for-an-attack)
8. [Norms Comparison Table](#8-norms-comparison-table)
9. [Security Angle](#9-security-angle)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. What Is a Perturbation?

A **perturbation** is simply the difference between an original input and a modified version of it. If you take a photo of a cat, and then change a few pixel values ever so slightly, the "perturbation" is the set of changes you made -- not the photo itself.

Mathematically, if `x` is the original input (e.g., a vector of pixel values or a vector of network-flow features) and `x_adv` is the adversarial (attacked) version of that input, the perturbation is:

```
delta = x_adv - x
```

`delta` (the Greek letter is conventionally used for "a small change") is a vector with the same shape as `x`. Every entry in `delta` says "how much did this specific feature change?"

### The Analogy

Imagine `x` is a printed exam answer sheet, and `delta` is a set of edits made with an eraser and pencil. Some edits are big (erasing a whole answer and rewriting it), some are tiny (fixing a single stray pencil mark). The perturbation is the *edit itself*, independent of what the original answer sheet said.

In adversarial machine learning, the attacker's whole job is to find a `delta` that is:
1. **Small enough** that a human (or a simple sanity check) doesn't notice it, and
2. **Effective enough** that it flips the model's decision.

Those two goals are in tension, and "how small is small enough" is exactly the question norms answer.

---

## 2. Why We Need to Measure "How Big" a Perturbation Is

Suppose you are attacking an image classifier that thinks a photo of a panda is a panda with 99% confidence. You want to nudge the pixels just enough that the model instead says "gibbon."

You could:
- Change **one pixel** by a huge amount (paint one pixel bright red).
- Change **every pixel** by a tiny amount (nudge every pixel value by 0.01).
- Change **a handful of pixels** by a moderate amount.

All three are "perturbations," but they *feel* very different in terms of how visible or costly they are. To compare attacks, reason about defenses, and set attack budgets, we need a single number that summarizes "how big" a perturbation is. That number is called a **norm**.

---

## 3. What Is a Norm? (Plain English First)

A **norm** is a rule for turning a list of numbers (a vector) into a single non-negative number that represents its "size" or "length."

### The Analogy

Think about measuring how much you spent on a shopping trip where you bought several items:

- You could count **how many items** you bought (ignoring price).
- You could add up **the total money spent**.
- You could compute the **straight-line distance** you'd have walked if the prices were coordinates on a map.
- You could report **the single most expensive item** you bought.

Each of these is a valid, different way of summarizing "how much shopping happened." None of them is wrong -- they just highlight different aspects. Norms do exactly this for perturbation vectors: they highlight different aspects of "how much the input changed."

### The General Formula

For a vector `delta = [d1, d2, d3, ..., dn]`, the general "Lp norm" is:

```
||delta||_p = ( |d1|^p + |d2|^p + ... + |dn|^p ) ^ (1/p)
```

Don't worry about memorizing this yet -- Section 4 walks through each specific case (`p = 0, 1, 2, infinity`) with a plain description and a tiny numeric example before you ever see the formula in the abstract like this again.

---

## 4. The Four Norms Attackers Care About

### 4.1 L0 Norm -- Counting Changed Features

**Plain English**: "How many individual features did I touch, regardless of how much I changed each one?"

If you changed 4 pixels out of a million-pixel image (even by a huge amount each), the L0 norm is 4.

```
||delta||_0 = number of non-zero entries in delta
```

**Why attackers care**: L0-constrained attacks produce *sparse* perturbations -- change as few features as possible. This matters when each changed feature has a real-world cost or is individually inspectable, e.g., flipping a handful of bytes in a malware binary, or changing a handful of API calls in a network request, rather than smearing tiny noise across the entire input.

### 4.2 L1 Norm -- Total Amount of Change

**Plain English**: "If I add up the absolute size of every single change, what's the total?"

```
||delta||_1 = |d1| + |d2| + ... + |dn|
```

**Why attackers care**: L1 allows a few large changes or many small changes -- it just caps the *sum*. It tends to produce perturbations that are a mix of sparse and small: a few features change a lot, most don't change at all. This is a natural fit when you want the flexibility of L0-like sparsity but a smoother, easier-to-optimize mathematical objective.

### 4.3 L2 Norm -- Euclidean ("Straight-Line") Distance

**Plain English**: "If I treated the perturbation as an arrow in space, how long is that arrow?" This is the everyday notion of distance you learned in geometry class (the Pythagorean theorem, generalized to many dimensions).

```
||delta||_2 = sqrt( d1^2 + d2^2 + ... + dn^2 )
```

**Why attackers care**: L2 spreads the "budget" out smoothly across many features, tends to produce perturbations that look like faint, spread-out noise, and plays nicely with calculus (it's smooth and differentiable everywhere, including at zero, unlike L1 and L0). This is why many optimization-based attacks (including DeepFool, covered later in this module) default to measuring perturbation size in L2.

### 4.4 L-infinity Norm -- The Worst Single Change

**Plain English**: "What is the single biggest change I made to any one feature?" Ignore everything else -- just report the worst offender.

```
||delta||_inf = max( |d1|, |d2|, ..., |dn| )
```

**Why attackers care**: L-infinity is the most common constraint in classic adversarial example research (including FGSM and I-FGSM, covered later in this module) because it maps naturally onto "every pixel is allowed to change by at most epsilon." It guarantees *no single feature* changes too much, even if *every* feature changes by that maximum amount. This is ideal for images, where humans are fairly insensitive to small, uniform brightness shifts spread across every pixel, but would notice one wildly out-of-place pixel.

---

## 5. Worked Numeric Example: One Perturbation, Four Norms

Let's take one concrete perturbation vector and compute all four norms so you can see exactly what each one emphasizes.

Suppose we perturb 5 features of an input, and the perturbation vector is:

```
delta = [ 0.10, -0.20, 0.00, 0.05, 0.30 ]
          d1     d2     d3    d4    d5
```

### Step 1: L0 Norm (count non-zero entries)

```
d1 = 0.10  -> non-zero
d2 = -0.20 -> non-zero
d3 = 0.00  -> ZERO (doesn't count)
d4 = 0.05  -> non-zero
d5 = 0.30  -> non-zero

||delta||_0 = 4   (4 out of 5 features were touched)
```

### Step 2: L1 Norm (sum of absolute values)

```
||delta||_1 = |0.10| + |-0.20| + |0.00| + |0.05| + |0.30|
            = 0.10 + 0.20 + 0.00 + 0.05 + 0.30
            = 0.65
```

### Step 3: L2 Norm (square root of sum of squares)

```
||delta||_2 = sqrt( 0.10^2 + (-0.20)^2 + 0.00^2 + 0.05^2 + 0.30^2 )
            = sqrt( 0.0100 + 0.0400 + 0.0000 + 0.0025 + 0.0900 )
            = sqrt( 0.1425 )
            ~= 0.3775
```

### Step 4: L-infinity Norm (max absolute value)

```
||delta||_inf = max( 0.10, 0.20, 0.00, 0.05, 0.30 )
              = 0.30
```

### Side-by-Side Result

```
Perturbation:  [0.10, -0.20, 0.00, 0.05, 0.30]

||delta||_0   = 4       (features touched)
||delta||_1   = 0.65    (total change)
||delta||_2   = 0.3775  ("straight-line" size)
||delta||_inf = 0.30    (worst single change)
```

Notice that all four numbers describe *the same underlying perturbation*, yet they tell very different stories. If an attacker's budget is "L-infinity <= 0.3," this exact perturbation is right at the edge of what's allowed. If the budget were "L0 <= 2," this perturbation would violate the constraint immediately (it touches 4 features, not 2).

---

## 6. Visualizing Norm Balls

A **norm ball** is the set of all perturbations whose norm is less than or equal to some budget `epsilon`. Drawing these in 2D (just two features, `d1` and `d2`) makes the differences between norms very intuitive.

```
        L1 BALL (diamond)              L2 BALL (circle)
           d2                             d2
            |                              |
          . +  .                       .--+--.
        .   |    .                   /    |    \
      +-----+-----+  d1            +------+------+  d1
        .   |    .                   \    |    /
          . +  .                       '--+--'
            |                              |

     "Diamond": corners lie          "Circle": every direction
     on the axes -- sparse           costs the same -- change
     solutions (one feature          spread evenly across
     maxed out, other zero)          both features
     are cheap.                      is natural.


      L-INFINITY BALL (square)        L0 BALL (axis "cross", not convex)
           d2                             d2
            |                              |
      +-----+-----+                    ----+----   <-- only points
      |     |     |  d1                    |         exactly ON the
      |     |     |                  -------+-------  d1 axes count
      +-----+-----+                    ----+----     (plus the origin)
            |                              |

     "Square": every feature         Not a smooth shape at all --
     can independently reach         it's just "the axes." Very
     the max epsilon at the          few points qualify, which is
     same time.                      why L0 optimization is hard
                                      (it's a counting problem, not
                                      a smooth geometric one).
```

**Why the shape matters**: When an attack tries to find "the smallest perturbation that crosses the decision boundary," the shape of the norm ball determines *where* on the boundary the cheapest crossing point is likely to be.

- The **L1 diamond** has sharp corners sitting exactly on the axes. Optimizing under an L1 budget tends to land on those corners -- meaning solutions that change very few features a lot, and leave most features untouched (this is why L1 encourages sparsity, similar in spirit to L0 but much easier to compute with).
- The **L2 circle** is perfectly round -- no direction is cheaper than any other, so solutions tend to spread the perturbation smoothly across many features.
- The **L-infinity square** allows every feature to simultaneously sit at the maximum allowed change, which is why L-infinity attacks (like FGSM) often perturb *every single feature* by exactly `epsilon` or `-epsilon`.

---

## 7. Choosing a Norm: What It Means for an Attack

The choice of norm is not just a mathematical technicality -- it directly shapes what the resulting adversarial example looks like and what real-world constraint it models.

| If the attacker's real constraint is...                              | The natural norm to use is... |
|------------------------------------------------------------------------|--------------------------------|
| "I can only flip a few bytes/features without breaking functionality" | L0 |
| "I have a limited total budget of change to spend across features"    | L1 |
| "I want the overall perturbation to be as short/imperceptible as possible, spread out" | L2 |
| "No single pixel/feature may change by more than a fixed amount"      | L-infinity |

Example real-world mappings:

- **Malware evasion**: modifying a PE file's header or inserting a few benign-looking byte sequences maps to an **L0** (or L1) constraint -- you want to touch as few bytes as possible so the binary still executes correctly.
- **Image evasion (classic adversarial examples)**: nudging every pixel by an imperceptibly small amount maps to an **L-infinity** constraint -- a uniform per-pixel cap keeps the whole image looking "clean" to a human eye.
- **Minimal-distance evasion research (e.g., DeepFool)**: asking "what is the smallest possible nudge, in any direction, that flips the decision?" maps most naturally to **L2**, since it corresponds to ordinary Euclidean distance.

---

## 8. Norms Comparison Table

| Norm | Formula | Plain-English Meaning | Shape of Norm Ball (2D) | Typical Attack Use | Optimization Difficulty |
|------|---------|------------------------|---------------------------|----------------------|---------------------------|
| **L0** | count of non-zero entries | "How many features did I touch?" | Just the axes (not convex/smooth) | Sparse feature attacks (e.g., malware byte edits) | Hard -- combinatorial, not differentiable |
| **L1** | sum of `\|di\|` | "What's the total amount of change?" | Diamond | Sparse-ish attacks, feature selection style evasion | Moderate -- convex but has sharp corners (not smooth at 0) |
| **L2** | sqrt(sum of `di^2`) | "How long is the perturbation as a straight-line distance?" | Circle | Minimal-distance attacks (e.g., DeepFool) | Easy -- smooth, differentiable everywhere |
| **L-infinity** | max of `\|di\|` | "What is the single worst change to any one feature?" | Square | Bounded per-pixel attacks (e.g., FGSM, I-FGSM) | Easy -- has a simple closed-form solution via the sign function |

---

## 9. Security Angle

Norm constraints are not academic bookkeeping -- they define the **threat model** of an attack, and defenders design their countermeasures around specific norms.

- **Certified/robust defenses are norm-specific.** A model that is provably robust against L-infinity perturbations of size `epsilon = 8/255` (a very common benchmark in adversarial ML research) offers **no guarantee whatsoever** against an L2 or L0 attack of a totally different magnitude. If you know a defense was hardened against one norm, pivoting to a different norm is often the path of least resistance.
- **Real-world attack surfaces rarely match "textbook" norms perfectly.** A malware author can't just add "L-infinity noise" to a PE file -- doing so would corrupt the file format or break functionality. Translating an academic L-infinity or L2 attack into a domain where perturbations must be valid, functional inputs (executable files, network packets, SQL queries) is one of the core challenges of practical evasion, and usually means falling back to L0/L1-style, feature-limited edits.
- **Norm budgets are audit artifacts.** When you read a research paper or a bug bounty write-up claiming "we evaded detector X with an imperceptible perturbation," the very next question should be: *imperceptible according to which norm, and what was the budget epsilon?* A tiny L-infinity budget on a 1-megapixel image sounds impressive; the same budget on a 10-feature tabular fraud-detection model would be enormous and obviously "cheating."

---

## 10. Key Takeaways

- A **perturbation** (`delta`) is just the difference between an adversarial input and the original input: `delta = x_adv - x`.
- A **norm** is a single number summarizing the "size" of a perturbation vector; different norms highlight different aspects of that size.
- **L0** counts how many features changed (sparsity). **L1** sums the total amount of change. **L2** measures ordinary straight-line (Euclidean) distance. **L-infinity** measures the single worst per-feature change.
- The shape of each norm's "ball" (diamond, circle, square, or just the axes) explains why optimizing under each norm produces visually and structurally different adversarial examples.
- The choice of norm should match the attacker's real-world constraint: L0/L1 for "touch as few features as possible" scenarios (malware, structured data), L-infinity for "cap every single change" scenarios (classic image attacks), and L2 for "find the truly shortest path across the boundary" scenarios (DeepFool).
- Defenses are typically hardened against one specific norm -- understanding which norm a defense targets is a prerequisite for finding its blind spots.

---

*Next up: The Local Linearity Assumption -- why we can approximate a model's complicated loss surface as a simple straight line near a given input, and why that approximation is the mathematical engine behind every attack in this module.*
