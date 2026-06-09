# Linear Regression

## What is Linear Regression?

Linear regression is the simplest supervised learning algorithm. It answers one question: **"Can I draw a straight line through this data that predicts future values?"**

### The Analogy

Imagine you are tracking how many hours you study for exams and the scores you get:

- 1 hour of study --> 50 points
- 2 hours --> 60 points
- 3 hours --> 70 points

You notice a pattern: every extra hour of study adds roughly 10 points. If someone asks "What score will I get if I study for 5 hours?", you can extend the pattern and say "probably around 90 points."

That mental line you drew through the data? That is linear regression.

```
    Score
    100 |                              *  <-- predicted
     90 |                         *
     80 |                    *
     70 |               *
     60 |          *
     50 |     *
     40 |
        +----+----+----+----+----+----+---
         0    1    2    3    4    5    6
                    Hours Studied
```

---

## The Equation: y = mx + b

Every straight line can be described by this equation:

```
    y = mx + b

    Where:
    - y  = the predicted output (what we want to predict)
    - x  = the input feature (what we know)
    - m  = the slope (how steep the line is)
    - b  = the y-intercept (where the line crosses the y-axis)
```

In machine learning textbooks, you will often see this written as:

```
    y_hat = w * x + b

    Where:
    - y_hat = predicted value
    - w     = weight (same as slope)
    - b     = bias (same as y-intercept)
```

### What Do the Parts Mean?

| Component | Meaning | Example |
|-----------|---------|---------|
| **Slope (m or w)** | How much y changes when x increases by 1 | "Each hour of study adds 10 points" |
| **Intercept (b)** | The baseline value of y when x is 0 | "With 0 hours of study, you score 40" |
| **x** | The input/feature | Hours studied |
| **y** | The output/prediction | Exam score |

So if m = 10 and b = 40:
- Study 3 hours: y = 10(3) + 40 = 70
- Study 5 hours: y = 10(5) + 40 = 90

---

## Cost Function / Mean Squared Error (MSE)

How do we know if our line is a GOOD fit? We measure the **error** -- the distance between the line's predictions and the actual data points.

### The Idea

```
    Score
     75 |          o  <-- actual point (3, 75)
        |         /|
     70 |--------x-+  <-- prediction on line (3, 70)
        |       /  |
        |      /   +-- Error = 75 - 70 = 5
        |     /
        +----+----+---
              3
```

For each data point, the error is: `actual - predicted`

But some errors are positive (point above the line) and some are negative (point below). If we just add them up, they could cancel out. So we **square** each error first.

### Mean Squared Error Formula

```
    MSE = (1/n) * SUM of (actual_i - predicted_i)^2

    Step by step:
    1. For each data point, calculate: (actual - predicted)
    2. Square that difference
    3. Add up all the squared differences
    4. Divide by the number of data points (n)
```

**Lower MSE = better fit.** An MSE of 0 means the line passes through every point perfectly (which almost never happens with real data).

### Example

| Point | Actual (y) | Predicted (y_hat) | Error | Error Squared |
|-------|-----------|-------------------|-------|--------------|
| 1 | 50 | 48 | 2 | 4 |
| 2 | 60 | 62 | -2 | 4 |
| 3 | 70 | 70 | 0 | 0 |
| 4 | 75 | 78 | -3 | 9 |

MSE = (4 + 4 + 0 + 9) / 4 = 17 / 4 = **4.25**

---

## Gradient Descent: Finding the Best Line

We know how to measure error (MSE). Now, how do we find the values of m and b that MINIMIZE that error? Enter **gradient descent**.

### The Hill-Walking Analogy

Imagine you are standing on a hilly landscape in thick fog. You cannot see the bottom of the valley, but you can feel the slope under your feet. To get to the lowest point:

1. Feel which direction slopes downward
2. Take a step in that direction
3. Feel the slope again
4. Take another step downhill
5. Repeat until the ground feels flat (you have reached the bottom)

```
    Cost (MSE)
    |
    |  *                          <-- Start here (random m, b)
    |   \
    |    \
    |     \       * Step 2
    |      * Step 1 \
    |                \
    |                 * Step 3
    |                  \___
    |                      *___   <-- Minimum (best m, b)
    +------------------------------------------
                     Parameter value (m or b)
```

### How It Works Step by Step

1. **Initialize** m and b with random values (or zeros)
2. **Calculate** the MSE with current m and b
3. **Compute the gradient** -- the direction and steepness of the slope
4. **Update** m and b by taking a small step opposite to the gradient:
   ```
   m_new = m_old - learning_rate * gradient_of_m
   b_new = b_old - learning_rate * gradient_of_b
   ```
5. **Repeat** steps 2-4 until MSE stops decreasing significantly

### Learning Rate

The **learning rate** controls how big each step is:

```
    Too small:                    Too large:                  Just right:
    Takes forever to              Overshoots the              Converges smoothly
    reach the bottom              minimum, bounces            to the minimum
                                  around
    |   .                         |  *     *                  |   *
    |    .                        |   \   /                   |    \
    |     .                       |    \ /                    |     \
    |      .                      |     *   *                 |      \_
    |       .                     |      \ /                  |        \__
    |        .                    |       *                   |           *
    |         .___*               | (never converges)        |
```

| Learning Rate | Effect |
|--------------|--------|
| Too small (0.0001) | Very slow convergence, may take thousands of iterations |
| Too large (10) | Overshoots the minimum, may never converge |
| Just right (0.01) | Smooth convergence in reasonable time |

---

## Worked Example: Predicting House Prices

### The Data

We have 5 houses. The feature is square footage (x), and the label is price in thousands of dollars (y).

| House | Sq Footage (x) | Price in $K (y) |
|-------|----------------|-----------------|
| 1 | 1000 | 150 |
| 2 | 1500 | 200 |
| 3 | 2000 | 250 |
| 4 | 2500 | 300 |
| 5 | 3000 | 350 |

### Step 1: Visualize the Data

```
    Price ($K)
    400 |
    350 |                              *
    300 |                    *
    250 |               *
    200 |          *
    150 |     *
    100 |
        +----+----+----+----+----+----+---
         0  500  1000 1500 2000 2500 3000
                    Square Footage
```

This looks very linear. Good candidate for linear regression.

### Step 2: Find the Line

Using the standard formulas (or gradient descent), we find:

```
    m (slope)     = 0.1    (every extra sq ft adds $100, i.e., $0.1K)
    b (intercept) = 50     ($50K base price)

    Equation: y = 0.1x + 50
```

### Step 3: Verify With Our Data

| House | x | Actual y | Predicted: 0.1(x) + 50 | Error |
|-------|------|----------|------------------------|-------|
| 1 | 1000 | 150 | 0.1(1000) + 50 = 150 | 0 |
| 2 | 1500 | 200 | 0.1(1500) + 50 = 200 | 0 |
| 3 | 2000 | 250 | 0.1(2000) + 50 = 250 | 0 |
| 4 | 2500 | 300 | 0.1(2500) + 50 = 300 | 0 |
| 5 | 3000 | 350 | 0.1(3000) + 50 = 350 | 0 |

MSE = 0. A perfect fit (this is because the data is perfectly linear -- real data is messier).

### Step 4: Make a Prediction

What would a 1750 sq ft house cost?

```
    y = 0.1(1750) + 50 = 175 + 50 = $225K
```

---

## Multiple Linear Regression

Real-world predictions use MORE than one feature. Multiple linear regression extends the concept to many input variables:

```
    Simple:    y = w1*x1 + b
    Multiple:  y = w1*x1 + w2*x2 + w3*x3 + ... + b
```

### Example: Predicting House Prices with Multiple Features

```
    Price = 0.1*(sq_footage) + 5*(bedrooms) + 10*(bathrooms) - 2*(age) + 50

    For a house with 2000 sqft, 3 beds, 2 baths, 10 years old:
    Price = 0.1(2000) + 5(3) + 10(2) - 2(10) + 50
          = 200 + 15 + 20 - 20 + 50
          = $265K
```

Each feature gets its own weight (w), and the model learns all the weights simultaneously during training.

---

## Security Angle

### Offensive and Defensive Uses

**1. Predicting Attack Likelihood**
- Train a regression model on historical data: network features --> likelihood of breach
- Input: number of open ports, patch age, number of failed logins
- Output: continuous risk score (0.0 to 1.0)

**2. Anomaly Scoring**
- Build a regression model of "normal" system behavior
- Predict expected CPU usage, network traffic, or login frequency
- When actual values deviate far from predictions (high error), flag as anomalous

```
    Normal behavior model: predicted_traffic = 0.5*(hour) + 10

    At 2pm (hour=14):
    Expected traffic: 0.5(14) + 10 = 17 Mbps
    Actual traffic:   85 Mbps
    Residual error:   68 Mbps  <-- ANOMALY FLAG
```

**3. Attack Timing Prediction**
- Regression models can predict WHEN attacks are likely based on historical patterns
- Features: day of week, time of day, recent vulnerability disclosures
- Output: expected number of attack attempts in the next hour

**4. Evasion Considerations**
- If attackers know the regression model's weights, they can craft activity that produces a low anomaly score
- Gradually increasing malicious traffic so the residual error stays below the threshold
- This is called a **low-and-slow** attack -- staying just under the detection line

### How Attackers Exploit Linear Regression Models

| Attack | Description |
|--------|------------|
| **Model Extraction** | Query the model many times to reconstruct the weights and intercept |
| **Threshold Gaming** | Keep malicious activity just below the anomaly threshold |
| **Feature Manipulation** | Change input features (e.g., spoof packet sizes) to get a benign prediction |
| **Training Data Poisoning** | Inject misleading data to shift the learned line |

---

## Key Terminology

| Term | Definition |
|------|-----------|
| **Regression** | Predicting a continuous numerical output |
| **Slope (weight)** | How much the output changes per unit change in the input |
| **Intercept (bias)** | The output value when all inputs are zero |
| **Cost Function** | A function that measures how wrong the model's predictions are |
| **MSE** | Mean Squared Error -- average of squared differences between predicted and actual |
| **Gradient Descent** | Optimization algorithm that iteratively adjusts parameters to minimize cost |
| **Learning Rate** | Step size in gradient descent; controls how fast parameters update |
| **Convergence** | When the model has reached (or nearly reached) the minimum error |
| **Residual** | The difference between the actual value and the predicted value for a single point |
| **R-squared** | A metric (0 to 1) indicating how much variance in y is explained by x |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|-----------|-----------|
| Simple to understand and explain | Assumes a LINEAR relationship between x and y |
| Fast to train, even on large datasets | Sensitive to outliers (one weird point can tilt the line) |
| Works well when the relationship is truly linear | Cannot capture complex, non-linear patterns |
| Requires very few hyperparameters | Assumes features are independent (multicollinearity problems) |
| Provides interpretable coefficients | Predicts values outside training range poorly (extrapolation risk) |
| Good baseline model to compare against | All features must be numerical (need encoding for categories) |

---

## Key Takeaways

1. **Linear regression fits a straight line** (or hyperplane in multiple dimensions) **to predict a continuous number.** It is the simplest regression algorithm.

2. **The equation y = mx + b is the foundation.** The slope (m) tells you how much y changes per unit of x. The intercept (b) is the baseline.

3. **MSE measures how good the line fits.** Lower MSE means the line's predictions are closer to the actual data points.

4. **Gradient descent finds the best m and b** by iteratively stepping in the direction that reduces error, like walking downhill in fog.

5. **The learning rate is critical.** Too small = slow learning. Too large = never converges. Tuning it is an essential skill.

6. **In security, regression models power anomaly detection** by modeling "normal" behavior and flagging deviations. Attackers evade them by staying below detection thresholds.

7. **For the exam:** Be able to explain the equation, what MSE measures, how gradient descent works, and how a regression-based anomaly detector can be evaded by a low-and-slow attack.
