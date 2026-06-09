# Anomaly Detection

## What Is Anomaly Detection?

Anomaly detection is the process of finding data points that **do not fit the expected
pattern**. These unusual data points are called anomalies, outliers, or novelties.

### The "Odd One Out" Analogy

Imagine you are a security guard watching a building entrance. Every day, 200
employees badge in between 8:00 and 9:00 AM, wearing business clothes, walking at
a normal pace. One morning, someone badges in at 3:00 AM, wearing a hoodie, and
sprints to the server room. You do not need a list of "known criminals" to know
something is wrong. That person's behavior **deviates from the established pattern**.

That is anomaly detection: learning what "normal" looks like, then flagging anything
that deviates significantly.

```
  NORMAL PATTERN vs. ANOMALY
  ==========================

  Login Time Distribution (Normal Business Hours)

  Count
    |
  30|          #####
    |        #########
  20|      #############
    |    #################
  10|  #####################
    |#########################
   0+--+--+--+--+--+--+--+--+--+--+--+--+--+
    12  2  4  6  8  10 12  2  4  6  8  10 12
    AM          AM          PM          PM

                                      ^
                                      |
                              Login at 3:17 AM
                              = ANOMALY
                              (far outside the normal distribution)
```

---

## Three Types of Anomalies

### 1. Point Anomalies

A **single data point** that is far from the rest of the data. This is the simplest
and most common type.

**Example:** An employee who normally transfers 10 MB/day suddenly transfers 50 GB
in one hour.

```
  POINT ANOMALY
  =============

                 x  <-- anomaly (far from everything else)


        . . . .
       . . . . .
      . . . . . .
       . . . . .
        . . . .
```

### 2. Contextual (Conditional) Anomalies

A data point that is **normal in one context but anomalous in another**. The same
value can be fine or suspicious depending on context.

**Example:** 10,000 login attempts per hour is normal during Monday morning. The
same 10,000 attempts at 3:00 AM on Christmas Day is anomalous.

```
  CONTEXTUAL ANOMALY
  ==================

  Login Volume
    |
  10K|                  x   <-- Same value (10K logins)
    |  x                       but anomalous at 3 AM on holiday
    |
   5K|
    |
    0+------+------+------+
     Mon 9AM  Tue 9AM  Dec 25
                        3 AM

  The value 10K is not unusual by itself.
  The CONTEXT (time + day) makes it anomalous.
```

### 3. Collective Anomalies

A **group of data points** that are individually normal but collectively form an
unusual pattern.

**Example:** A single failed login is normal. But 500 failed logins across 50
different accounts from the same IP in 10 minutes -- each individual failure is
normal, but the **collection** is a brute-force attack.

```
  COLLECTIVE ANOMALY
  ==================

  Each individual event is normal:
    - Failed login (happens all the time)
    - From external IP (happens all the time)
    - Different username (happens all the time)

  But together in sequence:
    t=0:00  Failed login, user=admin,    IP=45.33.x.x
    t=0:01  Failed login, user=root,     IP=45.33.x.x
    t=0:02  Failed login, user=guest,    IP=45.33.x.x
    t=0:03  Failed login, user=test,     IP=45.33.x.x
    ...
    t=9:59  Failed login, user=backup,   IP=45.33.x.x

  500 events, 50 usernames, 1 IP, 10 minutes = BRUTE FORCE ATTACK
  The pattern is anomalous, not any single event.
```

---

## Anomaly Detection Methods

### Method 1: Statistical Approaches

Use statistical models to define "normal" and flag deviations.

**How it works:** Calculate the mean and standard deviation of a feature. Anything
beyond 2-3 standard deviations from the mean is an anomaly.

```
  STATISTICAL APPROACH (Z-Score)
  ==============================

  Normal Distribution of Packet Sizes

                    #####
                 ###########
              #################
           #########################
        #################################
  -----|----|----|----|----|----|----|----|-----
      -3sd -2sd -1sd  mean +1sd +2sd +3sd

  |<--- anomalous --->|   normal   |<--- anomalous --->|

  Z-score = (value - mean) / standard_deviation

  If |Z-score| > 3, flag as anomaly.
  Only 0.3% of normal data falls beyond 3 standard deviations.
```

**Strengths:** Simple, fast, well-understood.
**Weaknesses:** Assumes data follows a known distribution (usually Gaussian). Real
security data rarely does.

### Method 2: Distance-Based Approaches

Anomalies are points that are **far from their neighbors**.

**How it works:** For each point, calculate the distance to its K nearest neighbors.
Points with unusually large distances are anomalies.

```
  DISTANCE-BASED ANOMALY DETECTION
  =================================

       x  <-- large distance to nearest neighbors = ANOMALY


        . . .
       . . . . .      . .
      . . . . . .    . . .
       . . . . .      . .
        . . .

  Normal points: close to many neighbors
  Anomalous point: far from all neighbors
```

**Strengths:** No assumption about data distribution.
**Weaknesses:** Computationally expensive for large datasets. Struggles when clusters
have different densities.

### Method 3: Density-Based Approaches (DBSCAN)

Anomalies are points in **low-density regions**.

DBSCAN (Density-Based Spatial Clustering of Applications with Noise) is a clustering
algorithm that naturally identifies outliers. It works by:

1. For each point, count how many neighbors are within a radius (epsilon).
2. Points with enough neighbors are **core points** (in dense regions).
3. Points reachable from core points are **border points**.
4. Points that are neither core nor border are **noise** -- these are your anomalies.

```
  DBSCAN
  ======

  epsilon = radius of neighborhood
  minPts = minimum neighbors to be a "core" point

                          N  <-- Noise point (anomaly)
                               (too few neighbors in epsilon)

     C C C C              C = Core point (>= minPts neighbors)
    C C C C C C           B = Border point (near a core point
   C C C C C C C              but not enough own neighbors)
    C C C C C B
     C C C C

         N  <-- Another noise point (anomaly)
```

**Strengths:** Finds clusters of arbitrary shape. Does not require specifying number
of clusters. Naturally identifies outliers.
**Weaknesses:** Sensitive to epsilon and minPts parameters. Struggles with clusters
of varying density.

### Method 4: Isolation Forest

Anomalies are points that are **easy to isolate** from the rest of the data.

**The intuition:** Normal points are surrounded by many similar points. To isolate a
normal point, you need many random splits. Anomalies are few and different, so they
can be isolated with very few splits.

```
  ISOLATION FOREST INTUITION
  ==========================

  Imagine repeatedly drawing random lines to split the data:

  Step 1:                Step 2:                Step 3:
  +----------+           +----|----- +          +----|----- +
  | . . .  x |           | . .|.  x |           | . .|.    |
  | . . . .  |           | . .|. .  |           | . .|. .  |
  | . . .    |           | . .|.    |           | . .|.    |
  | . . . .  |           | . .|. .  |           | . .|. .  |
  +----------+           +----|----- +          +----|----- +
                                                      |----x| <-- isolated!

  The anomaly (x) was isolated in just 2 splits.
  Normal points need 5-10 splits to isolate.

  Fewer splits to isolate = higher anomaly score.
```

**How it works:**

1. Build many random binary trees (an "isolation forest").
2. At each node, pick a random feature and a random split value.
3. For each data point, measure the average **path length** (number of splits needed
   to isolate it) across all trees.
4. Short average path length = anomaly. Long path length = normal.

**Strengths:** Very fast. Handles high-dimensional data well. No distance
calculations needed. Works with mixed feature types.
**Weaknesses:** Anomaly score threshold must be tuned. Can miss anomalies that are
clustered together (they are harder to isolate).

---

## Comparison of Anomaly Detection Methods

| Method | Speed | High Dimensions | Assumption | Best For |
|---|---|---|---|---|
| **Statistical (Z-score)** | Very Fast | Poor | Gaussian distribution | Simple, low-dimensional data |
| **Distance (KNN)** | Slow | Fair | None | Small datasets with clear outliers |
| **DBSCAN** | Medium | Fair | Density-based clusters | Spatial data, arbitrary shapes |
| **Isolation Forest** | Fast | Good | Anomalies are few and different | Large, high-dimensional datasets |
| **One-Class SVM** | Medium | Good | Boundary-based | When only normal data is available |
| **Autoencoder** | Slow (training) | Excellent | Reconstruction error | Complex, high-dimensional data |

---

## Worked Example: Detecting Unusual Login Behavior

### Scenario

You are monitoring employee logins. You have 30 days of baseline data and want to
flag anomalous behavior on Day 31.

### Baseline Data (30-day averages per employee)

| Employee | Avg Logins/Day | Avg Login Hour | Avg Sessions | Avg Data (MB) |
|---|---|---|---|---|
| Alice | 3 | 9:00 | 2 | 50 |
| Bob | 5 | 8:30 | 3 | 200 |
| Carol | 2 | 10:00 | 1 | 30 |
| Dave | 4 | 9:15 | 2 | 100 |
| Eve | 3 | 8:45 | 2 | 75 |

### Day 31 Observations

| Employee | Logins | Login Hour | Sessions | Data (MB) | Notes |
|---|---|---|---|---|---|
| Alice | 3 | 9:10 | 2 | 55 | Normal |
| Bob | 4 | 8:15 | 3 | 180 | Normal |
| Carol | 2 | 10:30 | 1 | 25 | Normal |
| Dave | 15 | 3:00 AM | 8 | 5000 | ANOMALY |
| Eve | 3 | 9:00 | 2 | 80 | Normal |

### Applying Statistical Detection (Z-Score)

For Dave, calculating Z-scores against his own baseline:

```
  Feature         Baseline     Day 31    Z-Score    Anomalous?
                  Mean (Std)   Value
  ----------------------------------------------------------------
  Logins/Day      4 (1.2)      15        9.2        YES (>3)
  Login Hour      9:15 (0.5h)  3:00 AM   12.5       YES (>3)
  Sessions        2 (0.8)      8         7.5        YES (>3)
  Data (MB)       100 (30)     5000      163.3      YES (>3)

  Z-Score = (observed - mean) / std_deviation

  Dave's Login Z-Score = (15 - 4) / 1.2 = 9.2
  Anything above 3 is flagged. Dave has Z-Scores of 9+.
```

### Investigation

Dave's account shows:
- 15 logins (normally 4)
- At 3:00 AM (normally 9:15 AM)
- 8 sessions (normally 2)
- 5 GB data transfer (normally 100 MB)

**Possible explanations:**
1. Dave's credentials were compromised (most likely)
2. Dave is a malicious insider exfiltrating data
3. Dave is working on an emergency project (least likely given the pattern)

This is a classic example of how anomaly detection catches threats that
signature-based systems would miss entirely.

---

## Security and Offensive Angle

### This Is HUGE for Security

Anomaly detection is arguably the **most important application of unsupervised
learning in cybersecurity**. Here is why:

Traditional security tools (antivirus, IDS signatures, firewall rules) detect
**known threats**. They work by matching against a database of known-bad patterns.
If an attack is not in the database, it is invisible.

Anomaly detection flips this: instead of defining what is bad, it defines what is
**normal** and flags everything else. This catches:

- Zero-day exploits
- Novel malware
- Insider threats
- Advanced Persistent Threats (APTs)
- Attacks that bypass signature-based defenses

### Defensive Applications

| Application | What Is "Normal" | What Gets Flagged |
|---|---|---|
| **Intrusion Detection (IDS)** | Normal network traffic patterns (packet sizes, protocols, timing) | Unusual traffic patterns that do not match known baselines |
| **Fraud Detection** | Typical transaction amounts, locations, times for each user | Transactions that deviate from the user's established pattern |
| **Insider Threat Detection** | Each employee's typical access patterns, data volumes, working hours | Employees whose behavior suddenly changes (credential theft or malicious intent) |
| **DNS Anomaly Detection** | Normal DNS query patterns (domains, frequency, record types) | DNS tunneling, DGA domains, beaconing to C2 servers |
| **Endpoint Detection (EDR)** | Normal process behavior, file access patterns, registry changes | Processes behaving unlike their historical profile (fileless malware, living-off-the-land) |
| **Cloud Security** | Normal API call patterns, resource usage, access patterns | Unusual API calls (recon), excessive resource spinning (cryptomining), lateral movement |
| **Email Security** | Normal email volume, recipients, attachment types per user | Business Email Compromise (BEC), data exfiltration via email |

### Offensive / Red Team: How Attackers Evade Anomaly Detection

This is critical for the exam. Attackers who understand anomaly detection can
systematically evade it.

| Evasion Technique | How It Works |
|---|---|
| **Low-and-Slow Attacks** | Stay within normal thresholds. Instead of exfiltrating 5 GB at once, transfer 50 MB/day for 100 days. Each day looks normal. |
| **Mimicry Attacks** | Study the normal traffic profile and craft attack traffic that matches it. Use the same protocols, packet sizes, and timing as legitimate traffic. |
| **Living Off the Land (LOTL)** | Use legitimate system tools (PowerShell, WMI, certutil) instead of custom malware. These tools are part of the normal baseline. |
| **Gradual Behavior Shift** | Slowly shift behavior over weeks so the "normal" baseline adapts. By the time you exfiltrate data, the model considers it normal. |
| **Timing Attacks** | Perform malicious actions during high-traffic periods when anomaly thresholds are naturally wider. |
| **Distributed Attacks** | Split activity across many accounts or IPs so no single entity triggers an anomaly. |
| **Poisoning the Baseline** | If you compromise a system early, your malicious behavior becomes part of the training data and is learned as "normal." |

```
  EVASION BY MIMICRY
  ==================

  Normal HTTP Traffic Profile:
  - GET requests: 70%
  - POST requests: 25%
  - Other: 5%
  - Avg request/min: 15
  - Avg payload: 2KB

  Attacker C2 Traffic (naive):       Attacker C2 Traffic (mimicry):
  - POST requests: 100%             - GET requests: 68%
  - Avg request/min: 1              - POST requests: 27%
  - Avg payload: 50KB               - Other: 5%
  - Fixed timing intervals           - Avg request/min: 14
                                     - Avg payload: 2.5KB
  DETECTED by anomaly               - Randomized timing
  detection immediately.
                                     BLENDS IN with normal
                                     traffic. Much harder
                                     to detect.
```

### The Arms Race

Anomaly detection creates a cat-and-mouse dynamic:

```
  DEFENDER                          ATTACKER
  ========                          ========

  1. Deploy anomaly detection  -->  Studies what "normal" looks like
  2. Catches obvious anomalies      |
                                    v
  3. Updates model           <--  Mimics normal behavior to evade
  4. Adds behavioral features       |
                                    v
  5. Catches mimicry attacks <--  Uses LOTL + slow exfiltration
  6. Adds sequence analysis          |
                                    v
  7. Catches slow attacks    <--  Poisons training data
  8. Adds data validation            |
                                    ...and so on
```

---

## Key Terminology

| Term | Definition |
|---|---|
| **Anomaly (Outlier)** | A data point that deviates significantly from the expected pattern |
| **Point Anomaly** | A single data point that is far from the rest of the data |
| **Contextual Anomaly** | A data point that is anomalous only in a specific context (time, location, etc.) |
| **Collective Anomaly** | A group of data points that are individually normal but form an unusual pattern together |
| **Baseline** | The established "normal" behavior pattern learned from historical data |
| **Z-Score** | Number of standard deviations a value is from the mean; measures how unusual a value is |
| **Isolation Forest** | An ensemble method that isolates anomalies using random binary trees; fewer splits = more anomalous |
| **DBSCAN** | Density-Based Spatial Clustering of Applications with Noise; clusters dense regions and labels sparse points as anomalies |
| **False Positive** | A normal event incorrectly flagged as anomalous (alert fatigue) |
| **False Negative** | An actual anomaly that the system fails to detect (missed attack) |
| **Epsilon (DBSCAN)** | The radius of the neighborhood around each point |
| **Path Length (Isolation Forest)** | The number of splits needed to isolate a point in a random tree |
| **Mimicry Attack** | Crafting malicious activity to resemble normal behavior and evade anomaly detection |
| **Living Off the Land (LOTL)** | Using legitimate system tools for malicious purposes to avoid detection |
| **Data Poisoning** | Injecting malicious data into the training set so the model learns it as "normal" |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|---|---|
| Detects unknown/novel threats (zero-days) | High false positive rate causes alert fatigue |
| No need for labeled attack data | Defining "normal" is hard -- normal changes over time |
| Catches insider threats and APTs | Sophisticated attackers can mimic normal behavior |
| Works across domains (network, host, user, cloud) | Requires a clean baseline (training on compromised data = bad model) |
| Can adapt to evolving environments | Threshold tuning is difficult (too sensitive = too many alerts, too loose = missed attacks) |
| Complements signature-based detection | Context matters -- same behavior may be normal or anomalous depending on time/role/location |
| Scales to large-scale monitoring | Computationally expensive for some methods (distance-based) |

---

## Key Takeaways

1. **Anomaly detection finds the "odd one out"** by learning what normal looks like
   and flagging deviations. It does not need examples of attacks.

2. **Three types of anomalies:** Point (single outlier), Contextual (normal value in
   wrong context), and Collective (individually normal events forming an unusual
   pattern).

3. **Four main methods:** Statistical (Z-score), Distance-based (KNN), Density-based
   (DBSCAN), and Tree-based (Isolation Forest). Each has trade-offs.

4. **Isolation Forest is particularly popular in security** because it is fast,
   handles high dimensions well, and does not require distance calculations.

5. **This is the most important unsupervised technique for security.** It catches
   zero-days, insider threats, APTs, and novel attacks that signature-based systems
   miss entirely.

6. **Attackers actively try to evade anomaly detection** through mimicry, low-and-slow
   attacks, Living Off the Land techniques, and data poisoning.

7. **The false positive problem is real.** Too many false alarms cause analysts to
   ignore alerts, which is exactly what attackers want.

8. **A clean baseline is essential.** If the training data is already compromised,
   the model learns malicious behavior as "normal." Always validate baseline data.

9. **For the exam:** Know the three anomaly types, the main detection methods, how
   anomaly detection is used defensively (IDS, fraud, insider threats), and how
   attackers evade it (mimicry, LOTL, poisoning, low-and-slow). This topic is central
   to offensive AI.
