# K-Means Clustering

## What Is K-Means?

K-Means is one of the simplest and most widely used clustering algorithms. It divides
data into **K groups** (clusters) where each data point belongs to the cluster with
the nearest center.

### The "Sorting Balls" Analogy

Imagine you dump 100 colored balls onto a table -- red, blue, and green, but you are
colorblind. You cannot see the colors, but you **can** measure each ball's size and
weight. You want to sort them into 3 groups (K=3). Here is what you do:

1. **Randomly pick 3 balls** as your initial "group leaders" (centroids).
2. **Assign every other ball** to the closest group leader based on size and weight.
3. **Recalculate** each group's center by averaging the size and weight of all balls
   in that group. The center point becomes the new group leader.
4. **Repeat** steps 2-3 until the groups stop changing.

When you finish, you will likely find that the three groups correspond to the three
colors -- even though you never saw the colors. The size and weight differences were
enough to separate them.

That is K-Means.

---

## The Algorithm Step-by-Step

### Step 1: Choose K (number of clusters)

You decide in advance how many clusters you want. This is the "K" in K-Means.

### Step 2: Initialize K Centroids

Place K centroids randomly in your data space. Each centroid is a "center point" for
a cluster.

```
  STEP 2: Random Initialization (K=2)
  ====================================

        10 |
         9 |          x
         8 |    x           x
         7 |       x
         6 |                         x
         5 |                    x
         4 |                       x
         3 |                  x
         2 |
         1 |
         0 +--+--+--+--+--+--+--+--+--+
           0  1  2  3  4  5  6  7  8  9

     x = data points
     C1 randomly placed at (2, 7)
     C2 randomly placed at (6, 4)
```

### Step 3: Assign Each Point to the Nearest Centroid

Calculate the distance from every data point to every centroid. Assign each point to
the closest one.

```
  STEP 3: Assignment (Iteration 1)
  =================================

        10 |
         9 |          A
         8 |    A           A
         7 |       A
         6 |                  C1     B
         5 |                    B
         4 |                  C2   B
         3 |                  B
         2 |
         1 |
         0 +--+--+--+--+--+--+--+--+--+
           0  1  2  3  4  5  6  7  8  9

     A = assigned to Cluster 1 (closer to C1)
     B = assigned to Cluster 2 (closer to C2)
     C1, C2 = current centroid positions
```

### Step 4: Recalculate Centroids

Move each centroid to the **average position** of all points assigned to it.

```
  STEP 4: Recalculate Centroids
  ==============================

  Cluster A points: (2,8), (3,7), (4,9), (5,8)
  New C1 = average = ((2+3+4+5)/4, (8+7+9+8)/4) = (3.5, 8.0)

  Cluster B points: (6,3), (7,5), (7,4), (8,6)
  New C2 = average = ((6+7+7+8)/4, (3+5+4+6)/4) = (7.0, 4.5)

        10 |
         9 |          A
         8 |    A    C1*    A
         7 |       A
         6 |                         B
         5 |                    B
         4 |                  C2*  B
         3 |                  B
         2 |
         1 |
         0 +--+--+--+--+--+--+--+--+--+
           0  1  2  3  4  5  6  7  8  9

     C1* and C2* = NEW centroid positions (moved to cluster averages)
```

### Step 5: Repeat Until Convergence

Go back to Step 3 (reassign points) and Step 4 (recalculate centroids). Keep
repeating until:

- No points change their cluster assignment, OR
- The centroids move less than a tiny threshold, OR
- You hit a maximum number of iterations.

```
  CONVERGENCE
  ===========

  Iteration 1:  Points shift between clusters, centroids move a lot
  Iteration 2:  Fewer points shift, centroids move less
  Iteration 3:  One point shifts, centroids barely move
  Iteration 4:  No points shift -- CONVERGED. Done.

  Final State:
        10 |
         9 |         [A]
         8 |   [A]    *     [A]         * = final centroid
         7 |      [A]
         6 |                        [B]
         5 |                   [B]
         4 |                    *  [B]
         3 |                 [B]
         2 |
         1 |
         0 +--+--+--+--+--+--+--+--+--+
           0  1  2  3  4  5  6  7  8  9
```

---

## The Math: Distance Calculation

K-Means uses **Euclidean distance** to measure how close a point is to a centroid:

```
  Distance = sqrt( (x1 - x2)^2 + (y1 - y2)^2 )

  Example:
  Point at (2, 8), Centroid at (6, 4)

  Distance = sqrt( (2-6)^2 + (8-4)^2 )
           = sqrt( 16 + 16 )
           = sqrt( 32 )
           = 5.66
```

For higher-dimensional data (more than 2 features), the formula extends:

```
  Distance = sqrt( (a1-b1)^2 + (a2-b2)^2 + ... + (an-bn)^2 )
```

---

## Choosing K: The Elbow Method

The biggest question in K-Means is: **how many clusters should you use?**

The Elbow Method helps you decide:

1. Run K-Means with K=1, K=2, K=3, ... K=10.
2. For each K, calculate the **inertia** (sum of squared distances from each point
   to its assigned centroid). Lower inertia = tighter clusters.
3. Plot K vs. inertia.
4. Look for the "elbow" -- the point where adding more clusters stops significantly
   reducing inertia.

```
  THE ELBOW METHOD
  ================

  Inertia
    |
  500|  x
    |
  400|
    |     x
  300|
    |
  200|        x
    |           x
  100|              x----x----x----x----x
    |
    0+----+----+----+----+----+----+----+----+--
     1    2    3    4    5    6    7    8    9   K

                    ^
                    |
              THE ELBOW (K=3)
              After K=3, adding more clusters
              gives diminishing returns.
              Choose K=3.
```

### Other Methods for Choosing K

| Method | How It Works |
|---|---|
| **Elbow Method** | Plot inertia vs. K, pick the "elbow" |
| **Silhouette Score** | Measures cluster separation; pick K with highest score |
| **Gap Statistic** | Compares inertia to a null reference distribution |
| **Domain Knowledge** | You already know how many groups to expect |

---

## Worked Example: Grouping Network Traffic to Find Anomalies

### Scenario

You are a SOC analyst. You have 8 network connections and want to cluster them to
find suspicious traffic. Each connection has two features: **packets per second** and
**average packet size (bytes)**.

### Raw Data

| Connection | Packets/sec | Avg Packet Size |
|---|---|---|
| C1 | 10 | 500 |
| C2 | 12 | 480 |
| C3 | 11 | 520 |
| C4 | 200 | 60 |
| C5 | 210 | 55 |
| C6 | 190 | 70 |
| C7 | 50 | 1400 |
| C8 | 8 | 510 |

### Step 1: Choose K=3

We suspect there might be 3 types of traffic.

### Step 2: Initialize Centroids

Randomly pick C1 (10, 500), C4 (200, 60), and C7 (50, 1400) as initial centroids.

### Step 3: Calculate Distances and Assign

```
  Distance from each point to each centroid:

  Connection  | Dist to Centroid-A | Dist to Centroid-B | Dist to Centroid-C | Assigned
              | (10, 500)          | (200, 60)          | (50, 1400)         |
  ------------|--------------------|--------------------|--------------------|---------
  C1 (10,500) | 0.0                | 472.4              | 904.9              | A
  C2 (12,480) | 20.1               | 457.8              | 921.0              | A
  C3 (11,520) | 20.0               | 491.4              | 882.9              | A
  C4 (200,60) | 472.4              | 0.0                | 1350.7             | B
  C5 (210,55) | 475.4              | 11.2               | 1358.8             | B
  C6 (190,70) | 455.7              | 14.1               | 1338.3             | B
  C7 (50,1400)| 904.9              | 1350.7             | 0.0                | C
  C8 (8,510)  | 10.2               | 481.6              | 893.5              | A
```

### Step 4: Recalculate Centroids

```
  Cluster A: C1(10,500), C2(12,480), C3(11,520), C8(8,510)
  New Centroid-A = (10.25, 502.5)

  Cluster B: C4(200,60), C5(210,55), C6(190,70)
  New Centroid-B = (200.0, 61.7)

  Cluster C: C7(50,1400)
  New Centroid-C = (50.0, 1400.0)    (only one point)
```

### Step 5: Reassign (Iteration 2)

All points stay in the same clusters. Converged.

### Interpretation

| Cluster | Connections | Profile | Assessment |
|---|---|---|---|
| **A** | C1, C2, C3, C8 | Low packets/sec, medium packet size | Normal web browsing |
| **B** | C4, C5, C6 | Very high packets/sec, tiny packets | Suspicious -- possible DDoS or port scan |
| **C** | C7 | Medium packets/sec, very large packets | Suspicious -- possible data exfiltration |

**Clusters B and C deserve investigation.** K-Means helped surface these anomalies
without any prior labels about what "malicious" traffic looks like.

---

## Limitations of K-Means

### 1. Must Choose K in Advance

You have to decide how many clusters before running the algorithm. If you pick the
wrong K, results will be misleading.

### 2. Assumes Spherical (Round) Clusters

K-Means works by measuring distance from a center point, so it naturally finds
round, evenly-sized clusters. It fails on elongated, irregular, or overlapping shapes.

```
  K-MEANS WORKS WELL:              K-MEANS FAILS:

      . . .                           . . . . . . .
    . . . . .                          . . . . . .
    . . . . .                           . . . . .
      . . .                          . . .
                                      . .
          . . .                        .
        . . . . .
        . . . . .                  (Crescent / banana shape --
          . . .                     K-Means cannot separate this
                                    properly)
  (Round blobs --
   K-Means handles
   this perfectly)
```

### 3. Sensitive to Initialization

Random starting centroids can lead to different results each run. A bad
initialization can trap the algorithm in a poor local minimum.

**Solution:** Use **K-Means++** initialization, which spreads initial centroids apart
strategically. Most modern implementations use this by default.

### 4. Sensitive to Outliers

A single extreme data point can pull a centroid far from the true cluster center.

### 5. Sensitive to Feature Scale

If one feature ranges from 0-1 and another from 0-1,000,000, the large-range feature
will dominate distance calculations. **Always normalize/standardize features first.**

---

## Security and Offensive Angle

### Defensive Applications

| Use Case | How K-Means Helps |
|---|---|
| **Malware Family Grouping** | Cluster malware samples by behavioral features (API calls, network patterns). New samples assigned to existing clusters are known variants; samples forming new clusters may be novel malware. |
| **Botnet C2 Detection** | Cluster DNS queries by timing, frequency, and domain characteristics. Botnet C2 traffic often forms a tight, distinct cluster separate from normal DNS. |
| **Network Traffic Classification** | Group flows by packet size, timing, and protocol to identify P2P, streaming, tunneling, and attack traffic. |
| **User Behavior Profiling** | Cluster employees by login times, accessed resources, and data volume. Compromised accounts shift clusters. |
| **Alert Triage** | Cluster IDS alerts to reduce thousands of alerts into a few meaningful groups for analyst review. |

### Offensive / Red Team Applications

| Use Case | How Attackers Use K-Means |
|---|---|
| **Target Profiling** | Cluster OSINT data about an organization's infrastructure to identify high-value targets. |
| **Password Analysis** | Cluster passwords from breached databases to identify common patterns, then generate targeted wordlists. |
| **Evasion Research** | Run K-Means on your own C2 traffic to see if it clusters separately from normal traffic. If it does, modify your C2 protocol to blend in. |
| **Phishing Campaign Optimization** | Cluster target employees by role and behavior to customize phishing lures per cluster. |

### Attacking K-Means Itself

An attacker who knows a defender uses K-Means can:

1. **Poison the training data** -- inject enough "normal-looking" malicious traffic to
   shift the centroids, causing the model to consider attack traffic as normal.
2. **Exploit the fixed K** -- if the defender uses K=5 but attacks create a 6th
   distinct pattern, K-Means will merge it into an existing cluster and miss it.
3. **Adversarial evasion** -- craft traffic that lands exactly on the boundary between
   the "normal" cluster and the "malicious" cluster, making classification ambiguous.

---

## Key Terminology

| Term | Definition |
|---|---|
| **K** | The number of clusters to create (chosen by the user) |
| **Centroid** | The center point of a cluster, calculated as the mean of all points in that cluster |
| **Inertia** | Total sum of squared distances from each point to its centroid (lower = tighter clusters) |
| **Convergence** | When the algorithm stabilizes and centroids stop moving significantly |
| **Euclidean Distance** | Straight-line distance between two points in space |
| **Elbow Method** | Technique for choosing K by plotting inertia vs. K and finding the bend |
| **K-Means++** | Smart initialization method that spreads centroids apart for better results |
| **Silhouette Score** | Metric from -1 to 1 measuring how well-separated clusters are |
| **Feature Scaling** | Normalizing features to a common range so no single feature dominates |
| **Local Minimum** | A suboptimal solution the algorithm gets stuck in due to bad initialization |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|---|---|
| Simple to understand and implement | Must choose K in advance |
| Fast -- scales well to large datasets | Assumes spherical, equally-sized clusters |
| Guaranteed to converge | Sensitive to random initialization |
| Works well when clusters are round and well-separated | Cannot handle non-globular cluster shapes |
| Easy to interpret results | Sensitive to outliers |
| Available in every ML library | Requires feature scaling |
| Good starting point for exploratory analysis | May converge to local minimum, not global best |

---

## Key Takeaways

1. **K-Means divides data into K groups** by iteratively assigning points to the
   nearest centroid and then recalculating centroids.

2. **The algorithm is simple:** initialize centroids, assign points, update centroids,
   repeat until stable.

3. **Choosing K is critical.** Use the Elbow Method or Silhouette Score. Wrong K gives
   wrong clusters.

4. **Always scale your features** before running K-Means, or large-range features will
   dominate.

5. **K-Means assumes round clusters.** For irregular shapes, consider DBSCAN or
   Gaussian Mixture Models instead.

6. **In security:** K-Means is widely used for grouping malware families, detecting
   botnet C2 patterns, and classifying network traffic without labeled data.

7. **Attackers can defeat K-Means** by poisoning training data, exploiting the fixed K,
   or crafting traffic that falls on cluster boundaries.

8. **For the exam:** Be able to walk through the K-Means algorithm step by step,
   explain the Elbow Method, and describe both defensive and offensive uses.
