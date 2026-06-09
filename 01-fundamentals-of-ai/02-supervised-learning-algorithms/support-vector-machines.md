# Support Vector Machines (SVM)

## What are SVMs?

A Support Vector Machine finds the **best boundary** that separates two classes of data. But it does not just find ANY boundary -- it finds the one with the **widest possible margin** between the two classes.

### The Analogy: Finding the Widest Road

Imagine two neighborhoods separated by a road. You need to draw the road between them:

- A narrow alley would work, but houses from both sides are dangerously close
- A wide highway gives the most breathing room between the two neighborhoods

SVM builds the **widest possible highway** between two classes. The wider the road, the more confident the model is about its classifications. New data points that fall on one side of the highway belong to one class; points on the other side belong to the other class.

```
    NARROW BOUNDARY (bad)          WIDE MARGIN (SVM - good)

    o o o|x x x                   o o o  |         |  x x x
    o o o|x x x                   o o o  |         |  x x x
    o o o|x x x                   o o o  |  MARGIN |  x x x
    o o o|x x x                   o o o  |         |  x x x
    o o o|x x x                   o o o  |         |  x x x

    The boundary just               The SVM maximizes this
    barely separates them            gap between classes
```

---

## Hyperplane and Margin

### What is a Hyperplane?

A **hyperplane** is the decision boundary that separates the classes. Its dimensionality depends on your data:

| Data Dimensions | Hyperplane Is | Visualization |
|----------------|---------------|---------------|
| 2D (two features) | A line | A line drawn on paper |
| 3D (three features) | A flat plane | A sheet of glass separating two groups |
| nD (n features) | An (n-1) dimensional surface | Cannot visualize, but the math works the same |

### What is the Margin?

The **margin** is the distance between the hyperplane and the nearest data points from each class. SVM maximizes this margin.

```
    Feature 2
    |
    |  o  o                          x
    |    o  o         |          x  x
    |      o  o       |        x  x
    |        o  o     |      x  x
    |     CLASS 0     |   CLASS 1
    |                 |
    |  <-- margin --> | <-- margin -->
    |                 |
    |           Hyperplane
    +----------------------------------------- Feature 1
```

### Why Maximize the Margin?

| Narrow Margin | Wide Margin |
|--------------|-------------|
| Sensitive to noise | Robust to noise |
| Small perturbation can cause misclassification | New points need to be far off to be misclassified |
| Overfits to training data | Better generalization |
| Like walking a tightrope | Like driving on a highway |

---

## Support Vectors

**Support vectors** are the data points that sit closest to the hyperplane -- they are right on the edge of the margin. They are called "support" vectors because they literally support (define) the position of the hyperplane.

```
    Feature 2
    |
    |  o                                    x
    |     o                              x
    |        O  <-- support vector   X  <-- support vector
    |         \                     /
    |          \     MARGIN        /
    |           \    |    |       /
    |            \   |    |      /
    |             \  |    |     /
    |              | |    |  X  <-- support vector
    |        O     | |    | /
    |              | |    |/
    |              |HYPERPLANE
    +---------------------------------------------- Feature 1

    O = support vectors (Class 0, on the margin edge)
    X = support vectors (Class 1, on the margin edge)
    o, x = other data points (do NOT affect the hyperplane)
```

### Key Insight About Support Vectors

The hyperplane is determined ONLY by the support vectors. If you remove any non-support-vector data point, the hyperplane does not change at all. This makes SVMs:

1. **Memory efficient** -- only the support vectors matter, not the entire dataset
2. **Robust** -- unaffected by points far from the boundary
3. **Interpretable** -- you can examine which data points define the boundary

---

## The Kernel Trick

### The Problem: Non-Linear Data

Sometimes data cannot be separated by a straight line:

```
    Can't separate with a line:     What we want:

    |     x  x  x                   |     x  x  x
    |  x           x                |  x    ___    x
    |  x   o  o    x                |  x  / o o \   x
    |  x   o  o    x                |  x | o  o  |  x
    |  x           x                |  x  \_____/   x
    |     x  x  x                   |     x  x  x
    
    No straight line works!         A CURVE would work
```

### The Solution: Map to Higher Dimensions

The kernel trick **transforms the data into a higher dimension** where it CAN be separated by a hyperplane (flat surface).

```
    2D view (not separable):         3D view (NOW separable!):

    x x o o o x x                        x   x
                                         / \
    A flat line cannot                  x   o---o---o   x
    separate o from x                       |       |
                                        x   o---o   x
                                         \ /
                                          x   x

                                    In 3D, a FLAT PLANE can now
                                    separate o (below) from x (above)
```

### Analogy

Imagine you have a table with red and blue marbles mixed together in a circle pattern -- red in the center, blue around the edge. No ruler (straight line) placed on the table can separate them. But if you LIFT the red marbles up (add a third dimension -- height), now you can slide a sheet of paper between the red marbles (above) and blue marbles (below).

The kernel trick does this mathematically without actually computing the higher-dimensional coordinates, which makes it computationally efficient.

### Common Kernel Types

| Kernel | When to Use | Description |
|--------|------------|-------------|
| **Linear** | Data is already linearly separable | No transformation, fastest |
| **RBF (Radial Basis Function)** | Non-linear, general purpose | Maps to infinite dimensions, most popular |
| **Polynomial** | Non-linear, moderate complexity | Maps to polynomial feature space |
| **Sigmoid** | Specific use cases | Similar to neural network activation |

```
    Choosing a kernel:

    Is the data linearly separable?
    |
    +-- YES --> Linear kernel (fastest)
    |
    +-- NO  --> How complex is the boundary?
                |
                +-- Moderate --> Polynomial kernel
                |
                +-- Complex or unknown --> RBF kernel (safe default)
```

---

## Worked Example: Classifying Malware vs Benign Executables

### The Data

We have 8 executable files. Features are **file entropy** (randomness of bytes, 0-8) and **number of suspicious API calls**.

| File | Entropy (x1) | Suspicious APIs (x2) | Label |
|------|-------------|----------------------|-------|
| A | 2.1 | 1 | Benign |
| B | 3.0 | 2 | Benign |
| C | 2.5 | 0 | Benign |
| D | 3.2 | 3 | Benign |
| E | 6.5 | 8 | Malware |
| F | 7.1 | 10 | Malware |
| G | 6.8 | 7 | Malware |
| H | 7.5 | 9 | Malware |

### Step 1: Plot the Data

```
    Suspicious APIs
    10 |                           F(M)
     9 |                              H(M)
     8 |                        E(M)
     7 |                       G(M)
     6 |
     5 |
     4 |             /  <-- Hyperplane
     3 |        D(B)/
     2 |     B(B) /
     1 |  A(B)  /
     0 |  C(B)/
       +--+--+--+--+--+--+--+--+--
        1  2  3  4  5  6  7  8
                 Entropy

    B = Benign,  M = Malware
    The line separates the two classes.
```

### Step 2: Identify Support Vectors

The support vectors are the points closest to the hyperplane:
- **D (Benign)** -- entropy 3.2, APIs 3 -- closest benign point
- **E (Malware)** -- entropy 6.5, APIs 8 -- closest malware point (approximately)

These two points define the margin width.

### Step 3: The SVM Finds the Optimal Hyperplane

The SVM positions the hyperplane exactly between D and E (the support vectors), maximizing the margin.

```
    Suspicious APIs
    10 |                           F
     9 |                              H
     8 |                        E  <-- support vector
     7 |               |       G
     6 |               |
     5 |          margin|margin
     4 |               |
     3 |        D  <-- |support vector
     2 |     B         |
     1 |  A            |
     0 |  C            |
       +--+--+--+--+--+--+--+--+--
        1  2  3  4  5  6  7  8
                 Entropy

    |  = Hyperplane (decision boundary)
    The margin is the space between D and E.
```

### Step 4: Classify a New File

A new executable has entropy = 5.5 and 6 suspicious API calls.

```
    Plot the new point:

    Suspicious APIs
    10 |                           F
     9 |                              H
     8 |                        E
     7 |               |   ?   G      ? = new file (5.5, 6)
     6 |               |
     5 |               |
     4 |               |
     3 |        D      |
     2 |     B         |
     1 |  A            |
     0 |  C            |
       +--+--+--+--+--+--+--+--+--
        1  2  3  4  5  6  7  8

    The new point falls on the RIGHT side of the hyperplane.
    Classification: MALWARE
```

---

## Security Angle

### SVMs in Security Applications

**1. Intrusion Detection Systems (IDS)**
- SVMs classify network connections as normal or attack
- Features: connection duration, bytes transferred, protocol type, flag status, service type
- The NSL-KDD dataset (a famous IDS benchmark) is commonly tested with SVMs
- SVMs handle the high-dimensional feature space well

**2. Malware Analysis**
- **Static analysis:** Features from PE headers, section sizes, imported functions, byte n-grams
- **Dynamic analysis:** Features from API call sequences, registry modifications, file system changes
- SVMs with RBF kernels can capture complex relationships between behavioral features

**3. Network Traffic Classification**
- Identify application protocols (HTTP, SSH, P2P) from encrypted traffic
- Features: packet sizes, inter-arrival times, flow statistics
- Important for detecting covert channels and tunneling

**4. Anomaly Detection (One-Class SVM)**
- A special variant that learns the boundary around "normal" data only
- Does not need labeled attack data -- just examples of normal behavior
- Flags anything outside the learned boundary as anomalous

```
    One-Class SVM:

    +----------------------------+
    |                            |
    |   o  o  o  o               |
    |     o  o  o  o  o          |
    |   o  o  NORMAL  o  o      |
    |     o  o  o  o  o          |
    |       o  o  o              |
    |                            |
    +------- BOUNDARY -----------+

                    x  <-- Anomaly! (outside the boundary)

    Anything inside the boundary = normal
    Anything outside = potentially malicious
```

### How Attackers Evade SVMs

**1. Margin Manipulation**

SVMs are defined by the margin and support vectors. If an attacker can shift their malicious input to the OTHER side of the margin, it gets misclassified:

```
    Before evasion:                After evasion:

    BENIGN | margin | MALWARE     BENIGN | margin | MALWARE
           |        |                    |        |
           |        |  * (attack)        | * (attack, shifted)
           |        |                    |        |
    
    The attacker modifies features     Now it falls on the
    to shift the point LEFT            BENIGN side of the margin
```

**2. Feature-Space Evasion for Malware**

| SVM Feature | Evasion Technique |
|------------|-------------------|
| File entropy | Add padding bytes to lower entropy |
| API call count | Use indirect/dynamic API resolution (LoadLibrary + GetProcAddress) |
| Imported DLLs | Delay-load DLLs so they do not appear in the import table |
| Section names | Rename sections to match common legitimate patterns |
| File size | Inflate the file with benign data |

**3. Kernel-Specific Attacks**

If the attacker knows which kernel is being used:
- **Linear kernel:** The decision boundary is a flat hyperplane. Find the normal vector and move perpendicular to it.
- **RBF kernel:** The boundary is curved. This is harder to evade but also harder for the defender to interpret.

**4. Poisoning Attacks**

SVMs are particularly vulnerable to training data poisoning because only the support vectors matter. If the attacker can inject points that become support vectors, they can shift the hyperplane significantly:

```
    Before poisoning:              After poisoning:

    o o | x x x                    o o    |  x x x
    o o | x x x                    o o  P |  x x x
    o o | x x x                    o o    |  x x x

    |  = original boundary         |  = shifted boundary
                                   P = poisoned point (labeled as benign
                                       but positioned near malware cluster)

    The injected point becomes a support vector
    and SHIFTS the entire decision boundary.
```

This is extremely effective against SVMs because you only need to affect the support vectors, not the entire dataset.

---

## Key Terminology

| Term | Definition |
|------|-----------|
| **Hyperplane** | The decision boundary that separates classes (a line in 2D, plane in 3D) |
| **Margin** | The distance between the hyperplane and the nearest data points from each class |
| **Support vectors** | The data points closest to the hyperplane that define the margin |
| **Maximum margin classifier** | An SVM finds the hyperplane that maximizes the margin |
| **Kernel** | A function that transforms data into higher dimensions for non-linear separation |
| **Kernel trick** | Computing in higher dimensions without explicitly transforming the data |
| **Linear kernel** | No transformation; assumes data is linearly separable |
| **RBF kernel** | Radial Basis Function; maps to infinite dimensions, handles complex boundaries |
| **Soft margin** | Allows some misclassifications (for noisy data); controlled by parameter C |
| **C parameter** | Controls the tradeoff: high C = fewer misclassifications but narrower margin; low C = wider margin but more misclassifications |
| **One-Class SVM** | Variant that learns a boundary around a single class (used for anomaly detection) |
| **Slack variable** | Allows a data point to be inside the margin or misclassified (soft margin) |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|-----------|-----------|
| Effective in high-dimensional spaces | Slow to train on very large datasets (O(n^2) to O(n^3)) |
| Memory efficient (only stores support vectors) | Does not directly provide probability estimates |
| Versatile via different kernels | Sensitive to feature scaling (must normalize data) |
| Works well when dimensions exceed samples | Choosing the right kernel and parameters (C, gamma) is hard |
| Robust to overfitting in high dimensions | Not ideal for very noisy data with overlapping classes |
| Strong theoretical foundations | Difficult to interpret with non-linear kernels |
| Effective with clear margin of separation | Does not scale well to extremely large datasets |

---

## Key Takeaways

1. **SVMs find the widest possible boundary** between two classes. This maximum-margin approach gives strong generalization to new data.

2. **Support vectors are the critical data points.** Only the points closest to the boundary matter. Everything else can be removed without changing the model.

3. **The kernel trick handles non-linear data** by mathematically transforming it into a higher-dimensional space where a flat boundary works. The RBF kernel is the most common choice.

4. **In security, SVMs power IDS, malware classifiers, and anomaly detectors.** One-Class SVM is particularly useful when you only have examples of "normal" behavior.

5. **Attackers evade SVMs by shifting features** to cross the decision boundary. Since SVMs rely on support vectors, poisoning attacks that inject new support vectors can shift the entire boundary with just a few malicious data points.

6. **SVMs are strong but not scalable.** For large datasets (millions of samples), other algorithms like Random Forests or neural networks are often preferred.

7. **For the exam:** Understand the concept of maximum margin, what support vectors are, why the kernel trick is needed for non-linear data, and how poisoning attacks are especially effective against SVMs because they target the support vectors that define the boundary.
