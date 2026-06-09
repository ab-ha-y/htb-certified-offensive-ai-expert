# Principal Component Analysis (PCA)

## What Is PCA?

Principal Component Analysis (PCA) is a **dimensionality reduction** technique. It
takes data with many features (dimensions) and compresses it into fewer features
while keeping as much of the important information as possible.

### The Shadow Analogy

Imagine you are holding a 3D wire sculpture of a cat. The sculpture is complex --
it has height, width, and depth. Now shine a flashlight at it from a specific angle
and look at the shadow on the wall. That shadow is a **2D representation** of the
3D object.

If you pick a bad angle, the shadow looks like a blob -- you lose important
information. But if you pick the **best angle**, the shadow clearly looks like a cat.
You reduced 3 dimensions to 2, but you preserved the most important shape information.

**PCA finds the best angle to shine the flashlight.** It identifies the directions
(axes) along which the data varies the most and projects everything onto those axes.

```
  3D OBJECT                         2D SHADOW (PCA Projection)
  ==========                        ==========================

       /\                               /\
      /  \   <-- depth                  /  \
     / /\ \                            / /\ \
    / /  \ \                          / /  \ \
   /_/    \_\                        /_/    \_\
   ||      ||                        |        |
   ||      || <-- height             |        |
   ||      ||                        |        |
   |________|                        |________|
       ^
       width                    Lost depth, kept height + width
                                (the two most informative dimensions)
```

---

## Why Dimensionality Reduction Matters

### The Curse of Dimensionality

As the number of features grows, several problems emerge:

1. **Computation becomes slow.** Distance calculations in 1000 dimensions are expensive.
2. **Data becomes sparse.** In high dimensions, data points are far apart from each
   other, making clustering and classification unreliable.
3. **Overfitting increases.** Models find spurious patterns in high-dimensional noise.
4. **Visualization is impossible.** Humans can only see 2D or 3D.

```
  DIMENSIONS AND DATA SPARSITY
  ============================

  1D: 10 points on a line
  |--x--x-x--x--x---x-x--x-x--x--|
  Points are close together. Patterns are easy to find.

  2D: 10 points on a plane
  +----------------------------------+
  |  x          x                    |
  |                    x             |
  |        x                   x    |
  |                                  |
  |     x          x        x       |
  |                      x          |
  +----------------------------------+
  Points are more spread out. Harder to see structure.

  100D: 10 points in 100-dimensional space
  Every point is essentially alone in a vast empty space.
  Distances between ALL pairs become nearly identical.
  Clustering breaks down. This is the curse.
```

PCA combats this by reducing 100 dimensions to, say, 5 -- keeping the dimensions
that carry the most information and discarding the rest.

---

## Eigenvalues and Eigenvectors in Plain English

These terms sound intimidating but the concepts are straightforward.

### Eigenvectors: The Directions of Maximum Spread

An **eigenvector** is a direction in your data along which the data is most spread
out (has the most variance).

Think of a football (American). If you measure the spread of the ball in every
direction:

- Along the long axis: **maximum spread** -- this is the first eigenvector (PC1).
- Along the short axis: **less spread** -- this is the second eigenvector (PC2).

```
  EIGENVECTORS OF A FOOTBALL-SHAPED DATA CLOUD
  =============================================

               PC2 (short axis)
                ^
                |
          . . . | . . .
        . . . . | . . . .
      . . . . . | . . . . .
  ----. . . . . + . . . . .----> PC1 (long axis)
      . . . . . | . . . . .
        . . . . | . . . .
          . . . | . . .
                |

  PC1 captures the MOST variance (the long direction)
  PC2 captures the NEXT most variance (the short direction)
  PC1 and PC2 are perpendicular (orthogonal) to each other
```

### Eigenvalues: How Much Spread Each Direction Has

An **eigenvalue** tells you **how much variance** (information) each eigenvector
captures. Larger eigenvalue = more important direction.

| Component | Eigenvalue | Variance Explained |
|---|---|---|
| PC1 (long axis) | 8.5 | 85% |
| PC2 (short axis) | 1.5 | 15% |
| **Total** | **10.0** | **100%** |

If PC1 already captures 85% of the variance, you could drop PC2 and keep only PC1.
You would go from 2D to 1D and still retain 85% of the information.

---

## How PCA Works Step-by-Step

### Step 1: Standardize the Data

Center each feature to mean=0 and scale to standard deviation=1. This ensures
features with large ranges do not dominate.

```
  BEFORE STANDARDIZATION          AFTER STANDARDIZATION
  Feature A: [100, 200, 300]      Feature A: [-1.22, 0, 1.22]
  Feature B: [0.1, 0.2, 0.3]     Feature B: [-1.22, 0, 1.22]
```

### Step 2: Compute the Covariance Matrix

The covariance matrix captures how each pair of features varies together.

### Step 3: Calculate Eigenvectors and Eigenvalues

From the covariance matrix, extract the eigenvectors (directions of maximum variance)
and eigenvalues (amount of variance in each direction).

### Step 4: Sort and Select Top Components

Rank eigenvectors by their eigenvalues (largest first). Select the top K eigenvectors
-- these are your **principal components**.

### Step 5: Project the Data

Multiply the original data by the selected eigenvectors to transform it into the
lower-dimensional space.

```
  PCA PIPELINE
  ============

  Original Data     Standardize    Covariance    Eigen-         Select     Project
  (n features)  -->  (mean=0,  -->  Matrix    --> decomposition --> Top K --> (K features)
                      std=1)                      (vectors +       PCs
                                                   values)

  [500 features] --> [500 features] --> [500x500 matrix] --> [500 vectors] --> [10 PCs] --> [10 features]
```

---

## ASCII Diagram: 2D to 1D Projection

Here is a concrete visual of PCA reducing 2 dimensions to 1.

```
  ORIGINAL 2D DATA                    STEP 1: Find PC1 direction
  =================                   =========================

  Y                                   Y
  |        .                          |        .     /
  |      . .                          |      . .   / <-- PC1 (direction
  |    .   .                          |    .   . /       of max variance)
  |  .    .                           |  .    ./
  | .   .                             | .   ./
  |.  .                               |.  ./
  +-----------> X                     +-----------> X

  STEP 2: Project onto PC1           RESULT: 1D representation
  =========================          ========================

  Y     /                            PC1 axis:
  |    /. <-- drop perpendicular     |--x-x--x-x--x-x--x--|
  |   /. |    to PC1 line
  |  / . |                           All the information along
  | / .  |                           the PC1 direction is kept.
  |/ .   |                           Information perpendicular
  +-----------> X                    to PC1 is lost (but it was
        \__|                         small anyway -- that is why
     projections land                PC1 was chosen).
     on the PC1 line
```

---

## Variance Explained

After running PCA, you get a **variance explained** breakdown that tells you how
much information each component captures.

```
  VARIANCE EXPLAINED (SCREE PLOT)
  ================================

  % Variance
  Explained
   50% |  ###
       |  ###
   40% |  ###
       |  ###   ###
   30% |  ###   ###
       |  ###   ###
   20% |  ###   ###
       |  ###   ###   ###
   10% |  ###   ###   ###   ###
       |  ###   ###   ###   ###   ###   ###
    0% +------+------+------+------+------+------
         PC1    PC2    PC3    PC4    PC5    PC6

  Cumulative: 45%   75%   88%   95%   98%   100%

  Decision: Keep PC1 through PC4 (95% of variance retained).
  Reduced from 6 dimensions to 4.
```

A common rule of thumb: **keep enough components to capture 90-95% of variance.**

---

## Worked Example: Reducing Features in Network Traffic Data

### Scenario

You are building an IDS and have a dataset with 6 features per network connection.
You want to reduce this to make your downstream model faster and less prone to
overfitting.

### Raw Data (6 features, 5 connections)

| Conn | Bytes | Packets | Duration (s) | Avg Pkt Size | Pkt Rate | Byte Rate |
|---|---|---|---|---|---|---|
| C1 | 5000 | 50 | 10 | 100 | 5.0 | 500 |
| C2 | 8000 | 80 | 10 | 100 | 8.0 | 800 |
| C3 | 3000 | 30 | 10 | 100 | 3.0 | 300 |
| C4 | 12000 | 60 | 20 | 200 | 3.0 | 600 |
| C5 | 20000 | 100 | 20 | 200 | 5.0 | 1000 |

### Observation: Redundant Features

Notice that several features are derived from others:

- Avg Pkt Size = Bytes / Packets
- Pkt Rate = Packets / Duration
- Byte Rate = Bytes / Duration

These features are **highly correlated**. PCA will discover this automatically.

### After PCA

```
  Step 1: Standardize all 6 features (mean=0, std=1)
  Step 2: Compute covariance matrix (6x6)
  Step 3: Eigenvalues (sorted):

  PC1: eigenvalue = 3.8  (63% variance)
  PC2: eigenvalue = 1.5  (25% variance)
  PC3: eigenvalue = 0.5  (8% variance)
  PC4: eigenvalue = 0.15 (3% variance)
  PC5: eigenvalue = 0.03 (0.5% variance)
  PC6: eigenvalue = 0.02 (0.5% variance)

  Cumulative: 63% -> 88% -> 96% -> 99% -> 99.5% -> 100%
```

### Decision

Keep **PC1 and PC2** (88% of variance) or **PC1, PC2, and PC3** (96% of variance).

**Result:** Reduced from 6 features to 2-3, with minimal information loss. The
downstream IDS model will train faster, generalize better, and be less prone to
overfitting.

### Interpreting the Components

| Component | Dominated By | Interpretation |
|---|---|---|
| PC1 | Bytes, Packets, Byte Rate | "Volume" -- how much data is flowing |
| PC2 | Duration, Pkt Rate | "Speed" -- how fast data is flowing |
| PC3 | Avg Pkt Size | "Packet characteristics" |

PCA discovered that the 6 raw features really describe 2-3 underlying concepts.

---

## Security and Offensive Angle

### Defensive Applications

| Use Case | How PCA Helps |
|---|---|
| **IDS Feature Reduction** | Network traffic datasets often have 50+ features. PCA reduces them to 5-10, making real-time detection feasible without sacrificing accuracy. |
| **Visualizing Attack Data** | Project 50-dimensional attack data into 2D for analyst visualization. Clusters of attacks become visible on a scatter plot. |
| **Noise Reduction** | PCA strips out noise and redundant features, helping downstream models focus on signal rather than noise. |
| **Faster Model Training** | Fewer features = faster training = more frequent model updates = quicker adaptation to new threats. |
| **Log Compression** | Apply PCA to high-dimensional log features to store compressed representations for long-term analysis. |

### Offensive / Red Team Applications

| Use Case | How Attackers Use PCA |
|---|---|
| **Feature Analysis** | Run PCA on defender's feature space to understand which features carry the most weight in detection. Focus evasion efforts on those features. |
| **Attack Surface Visualization** | Reduce high-dimensional scan results to 2D to identify clusters of similarly configured targets. |
| **Credential Pattern Analysis** | Apply PCA to password feature vectors to identify the most important password characteristics for a target organization. |

### Attacking PCA Itself

| Attack | Description |
|---|---|
| **Adversarial Perturbation** | Add small, carefully crafted noise to data that causes PCA to choose wrong principal components, degrading downstream model accuracy. |
| **Hiding in Discarded Dimensions** | If the defender keeps only PC1-PC3 (95% variance), craft attack traffic that is distinctive only in PC4-PC6 (the discarded 5%). The attack is invisible after PCA. |
| **Data Poisoning** | Inject training data that shifts the principal components, causing the model to retain noise and discard signal. |

The "hiding in discarded dimensions" attack is particularly clever: if a defender
uses PCA to reduce dimensions before feeding data to an IDS, an attacker can design
traffic that looks normal along the kept dimensions but is malicious along the
discarded ones. The malicious signal literally gets thrown away by PCA.

---

## Key Terminology

| Term | Definition |
|---|---|
| **Dimensionality Reduction** | Reducing the number of features while preserving information |
| **Principal Component (PC)** | A new axis (direction) in the data that captures maximum variance; each PC is a linear combination of original features |
| **Eigenvector** | The direction of a principal component |
| **Eigenvalue** | The amount of variance captured by a principal component |
| **Variance** | A measure of how spread out data is; more variance = more information |
| **Covariance Matrix** | A matrix showing how each pair of features varies together |
| **Projection** | Mapping data from high dimensions onto lower dimensions |
| **Scree Plot** | A graph of eigenvalues (or variance explained) vs. component number |
| **Explained Variance Ratio** | The percentage of total variance captured by each principal component |
| **Standardization** | Scaling features to mean=0 and standard deviation=1 before PCA |
| **Orthogonal** | Perpendicular; principal components are always orthogonal to each other |
| **Loading** | The weight of an original feature in a principal component; shows which features contribute most |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|---|---|
| Reduces computational cost for downstream models | Loses interpretability -- PCs are combinations of original features |
| Removes correlated/redundant features automatically | Assumes linear relationships between features |
| Enables visualization of high-dimensional data | Sensitive to feature scaling (must standardize first) |
| Reduces overfitting by removing noise | Information in discarded components is permanently lost |
| Fast to compute even on large datasets | Cannot capture non-linear structure (use kernel PCA or autoencoders instead) |
| No hyperparameters except number of components | Outliers can distort principal components |
| Deterministic -- same input always gives same output | Not suitable when all features are equally important |

---

## Key Takeaways

1. **PCA finds the directions of maximum variance** in your data and projects
   everything onto those directions, reducing dimensions.

2. **Think of it as finding the best camera angle** -- you want the view that shows
   the most shape and structure with the fewest dimensions.

3. **Eigenvalues tell you how much information each direction carries.** Keep the
   directions with the largest eigenvalues.

4. **Always standardize your data first.** PCA is based on variance, and
   un-standardized features with large ranges will dominate.

5. **Use the scree plot or cumulative variance** to decide how many components to
   keep. Aim for 90-95% of total variance.

6. **In security:** PCA is used to make IDS models faster, visualize attack data, and
   strip noise from high-dimensional network features.

7. **Attackers can hide in the discarded dimensions.** If PCA throws away 5% of
   variance, adversaries can craft attacks that live entirely in that 5%.

8. **For the exam:** Understand what PCA does conceptually (finds directions of max
   variance), why standardization matters, how to interpret variance explained, and
   the security implications of dimensionality reduction.
