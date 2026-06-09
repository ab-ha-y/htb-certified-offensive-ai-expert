# Perceptrons

## What is a Perceptron?

A perceptron is the **simplest possible artificial neuron** -- the fundamental building block of all neural networks. It takes multiple inputs, applies weights to them, sums everything up, and produces a single output: either 0 or 1 (yes or no).

Think of it as a **tiny decision-maker**. It looks at several pieces of evidence, weighs how important each one is, and makes a binary decision.

### Real-World Analogy

Imagine you are deciding whether to go to a restaurant. You consider three factors:

1. Is the food good? (importance: very high)
2. Is it close by? (importance: medium)
3. Is it cheap? (importance: low)

You mentally assign weights to each factor, add up a score, and if it crosses your personal threshold, you go. That is exactly what a perceptron does.

---

## Biological Neuron vs Artificial Neuron

The perceptron was inspired by how biological neurons work in the brain, though it is a drastic simplification.

```
BIOLOGICAL NEURON                    ARTIFICIAL NEURON (PERCEPTRON)
==================                   ================================

                                         Inputs      Weights
  Dendrites                              x1 ----w1---\
  (receive signals)                                    \
       \  |  /                           x2 ----w2----> [Sum + Bias] --> [Activation] --> Output
        \ | /                                          /
     [Cell Body]                         x3 ----w3---/
     (processes)
         |
      [Axon]
     (transmits)
         |
    [Synapses]
    (outputs to
     next neuron)
```

| Biological Neuron | Artificial Neuron (Perceptron) |
|---|---|
| Dendrites receive electrical signals | Inputs (x1, x2, x3, ...) receive numerical data |
| Synaptic strength determines signal importance | Weights (w1, w2, w3, ...) determine input importance |
| Cell body integrates all incoming signals | Summation function adds up weighted inputs + bias |
| Fires if signal exceeds a threshold | Activation function produces output if sum exceeds threshold |
| Axon transmits output to next neuron | Output passes to next layer of neurons |
| Learns by adjusting synaptic strength | Learns by adjusting weights and bias |

---

## ASCII Diagram of a Perceptron

```
    INPUTS         WEIGHTS        SUMMATION        ACTIVATION       OUTPUT
    ======         =======        =========        ==========       ======

                                 +----------+     +----------+
    x1 -------w1--------------->|           |     |          |
                                |           |     |          |
    x2 -------w2--------------->|  Weighted |---->| Step /   |-----> y
                                |   Sum     |     | Sigmoid/ |    (0 or 1)
    x3 -------w3--------------->|   + b     |     | ReLU     |
                                |           |     |          |
    1  --------b (bias)-------->|           |     |          |
                                +----------+     +----------+

    z = (x1*w1) + (x2*w2) + (x3*w3) + b

    y = activation(z)
```

---

## How a Perceptron Makes Decisions

### Step-by-Step Process

**Step 1: Receive Inputs**
The perceptron receives numerical inputs. These could be pixel values, feature measurements, or binary flags.

```
x1 = 1, x2 = 0, x3 = 1
```

**Step 2: Multiply by Weights**
Each input is multiplied by its corresponding weight. Weights represent how important each input is.

```
x1*w1 = 1 * 0.5 = 0.5
x2*w2 = 0 * 0.3 = 0.0
x3*w3 = 1 * 0.8 = 0.8
```

**Step 3: Sum Everything + Bias**
Add all weighted inputs together, plus a bias term (a constant that shifts the decision boundary).

```
z = 0.5 + 0.0 + 0.8 + (-0.6)   [bias = -0.6]
z = 0.7
```

**Step 4: Apply Activation Function**
Pass the sum through an activation function to get the final output.

```
If z >= 0 --> output 1
If z < 0  --> output 0

z = 0.7 >= 0, so output = 1
```

---

## Activation Functions

The activation function decides whether the neuron "fires" (produces output). Here are the three most important ones:

### 1. Step Function (Original Perceptron)

```
Output
  1 |         ___________
    |        |
    |        |
  0 |________|
    +-----|--|-----------
         0   threshold     Input (z)

    f(z) = 1  if z >= 0
    f(z) = 0  if z < 0
```

- **Behavior**: All or nothing. Output is exactly 0 or 1.
- **Problem**: Not differentiable at the threshold -- cannot use gradient descent to learn.

### 2. Sigmoid Function

```
Output
  1 |              ___----
    |          __--
    |        _/
 0.5|------/---
    |    _/
    |  --
  0 |--___
    +----------------------
                          Input (z)

    f(z) = 1 / (1 + e^(-z))
```

- **Behavior**: Smooth S-curve. Output is any value between 0 and 1.
- **Advantage**: Differentiable everywhere, so gradient descent works.
- **Use case**: Output can be interpreted as a probability.
- **Problem**: Vanishing gradient for very large or very small z values.

### 3. ReLU (Rectified Linear Unit)

```
Output
    |            /
    |           /
    |          /
    |         /
    |        /
  0 |_______/
    +----------------------
           0              Input (z)

    f(z) = max(0, z)
    f(z) = 0  if z < 0
    f(z) = z  if z >= 0
```

- **Behavior**: Zero for negative inputs, linear for positive inputs.
- **Advantage**: Simple, fast, avoids vanishing gradient for positive values.
- **Use case**: Most popular in hidden layers of modern deep networks.
- **Problem**: "Dead neurons" -- if z is always negative, the neuron never activates.

### Activation Function Comparison

| Function | Range | Differentiable? | Speed | Common Use |
|---|---|---|---|---|
| Step | {0, 1} | No (at threshold) | Fast | Historical; not used in practice |
| Sigmoid | (0, 1) | Yes | Moderate | Output layer for binary classification |
| ReLU | [0, infinity) | Yes (except at 0) | Fast | Hidden layers in deep networks |

---

## Worked Example: A Perceptron Learning the AND Gate

The AND gate produces 1 only when **both** inputs are 1.

### Truth Table for AND

| x1 | x2 | Expected Output |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

### Initial Setup

- Weights: w1 = 0.0, w2 = 0.0
- Bias: b = 0.0
- Learning rate: alpha = 0.1
- Activation: Step function (output 1 if z >= 0.5, else 0)

### Training -- Iteration 1

We go through each training example and update weights when the perceptron makes a mistake.

**Example 1: x1=0, x2=0, expected=0**

```
z = (0 * 0.0) + (0 * 0.0) + 0.0 = 0.0
output = step(0.0) = 0  (since 0.0 < 0.5)
error = expected - output = 0 - 0 = 0
No update needed (correct!)
```

**Example 2: x1=0, x2=1, expected=0**

```
z = (0 * 0.0) + (1 * 0.0) + 0.0 = 0.0
output = step(0.0) = 0
error = 0 - 0 = 0
No update needed (correct!)
```

**Example 3: x1=1, x2=0, expected=0**

```
z = (1 * 0.0) + (0 * 0.0) + 0.0 = 0.0
output = step(0.0) = 0
error = 0 - 0 = 0
No update needed (correct!)
```

**Example 4: x1=1, x2=1, expected=1**

```
z = (1 * 0.0) + (1 * 0.0) + 0.0 = 0.0
output = step(0.0) = 0
error = 1 - 0 = 1          <-- MISTAKE!

Update rule: w_new = w_old + alpha * error * x

w1 = 0.0 + 0.1 * 1 * 1 = 0.1
w2 = 0.0 + 0.1 * 1 * 1 = 0.1
b  = 0.0 + 0.1 * 1 * 1 = 0.1
```

### Training -- Iteration 2 (weights: w1=0.1, w2=0.1, b=0.1)

**Example 1: x1=0, x2=0, expected=0**

```
z = (0*0.1) + (0*0.1) + 0.1 = 0.1
output = step(0.1) = 0  (0.1 < 0.5)
error = 0. Correct!
```

**Example 2: x1=0, x2=1, expected=0**

```
z = (0*0.1) + (1*0.1) + 0.1 = 0.2
output = step(0.2) = 0
error = 0. Correct!
```

**Example 3: x1=1, x2=0, expected=0**

```
z = (1*0.1) + (0*0.1) + 0.1 = 0.2
output = step(0.2) = 0
error = 0. Correct!
```

**Example 4: x1=1, x2=1, expected=1**

```
z = (1*0.1) + (1*0.1) + 0.1 = 0.3
output = step(0.3) = 0
error = 1 - 0 = 1          <-- MISTAKE again!

w1 = 0.1 + 0.1 * 1 * 1 = 0.2
w2 = 0.1 + 0.1 * 1 * 1 = 0.2
b  = 0.1 + 0.1 * 1 * 1 = 0.2
```

### After Several More Iterations...

Eventually the weights converge to values like w1=0.3, w2=0.3, b=-0.4:

```
(0,0): z = 0 + 0 + (-0.4) = -0.4 --> 0   CORRECT
(0,1): z = 0 + 0.3 + (-0.4) = -0.1 --> 0   CORRECT
(1,0): z = 0.3 + 0 + (-0.4) = -0.1 --> 0   CORRECT
(1,1): z = 0.3 + 0.3 + (-0.4) = 0.2 --> 0   Hmm, still not quite...
```

After more iterations with the threshold at 0, and converging to something like w1=0.5, w2=0.5, b=-0.7:

```
(0,0): z = -0.7 --> 0   CORRECT
(0,1): z = -0.2 --> 0   CORRECT
(1,0): z = -0.2 --> 0   CORRECT
(1,1): z = 0.3  --> 1   CORRECT
```

The perceptron has learned AND. The key insight: training is just **iteratively adjusting weights** until the errors go away.

---

## Limitations: The XOR Problem

### Why This Matters Historically

The XOR (exclusive or) gate outputs 1 when the inputs are **different**:

| x1 | x2 | XOR Output |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

**A single perceptron cannot learn XOR.** Here is why:

```
    x2
    1 |   (0,1)=1       (1,1)=0
      |     *               o
      |
      |
    0 |   (0,0)=0       (1,0)=1
      |     o               *
      +-------------------------
      0                     1    x1

    * = output 1
    o = output 0

    A single perceptron draws ONE straight line to separate classes.
    There is NO single straight line that separates * from o here!
```

A perceptron can only solve **linearly separable** problems -- cases where you can draw a straight line (or hyperplane) between the two classes. XOR is not linearly separable.

### Historical Impact

In 1969, Marvin Minsky and Seymour Papert published a book proving this limitation. It caused the first "AI Winter" -- a period where funding and interest in neural networks collapsed.

### The Solution

The answer came in the 1980s: **multi-layer perceptrons (MLPs)** -- stack multiple perceptrons in layers, and they can solve XOR and any other computable function. This is the topic of the next study guide on neural networks.

```
SINGLE PERCEPTRON (fails at XOR):

    x1 ---w1--\
               >--[sum+activation]--> output
    x2 ---w2--/


MULTI-LAYER PERCEPTRON (solves XOR):

    x1 ----> [P1] ---\
         X            >--[P3]--> output
    x2 ----> [P2] ---/

    P1 and P2 each learn a linear boundary.
    P3 combines them to create a non-linear boundary.
```

---

## Security and Offensive Angle

### Why Perceptrons Matter for Security Professionals

Understanding perceptrons helps you understand **adversarial attacks at the most fundamental level**.

**How adversarial perturbations work at the neuron level:**

1. Each neuron computes a weighted sum of its inputs.
2. An adversarial attack works by making **tiny changes** to input values that, when multiplied by weights and accumulated across many neurons, push the final output past a decision boundary.
3. If you understand that a perceptron has a linear decision boundary (a straight line), you can see why adding a carefully calculated small vector to an input can flip the output.

```
Normal input:     z = 0.5*1.0 + 0.3*0.8 + (-0.2) = 0.54  --> Class A

Perturbed input:  z = 0.5*0.9 + 0.3*0.7 + (-0.2) = 0.46  --> Could flip!
                        ^^^        ^^^
                  Tiny changes (0.1 each) but they accumulate
```

**Practical implications:**

- In image classifiers, each pixel value feeds into neurons. Changing many pixels by tiny, imperceptible amounts can flip a classification entirely.
- The more neurons and layers, the more opportunities for perturbations to accumulate -- this is why deep networks can be more vulnerable to adversarial examples than simple models.
- Understanding the linear nature of individual neurons is the foundation for understanding the **Fast Gradient Sign Method (FGSM)** and other adversarial attacks.

---

## Key Terminology

| Term | Definition |
|---|---|
| **Perceptron** | The simplest artificial neuron; computes a weighted sum of inputs, adds bias, and applies an activation function |
| **Weight** | A learnable parameter that determines how much influence an input has on the output |
| **Bias** | A learnable constant added to the weighted sum; shifts the decision boundary |
| **Activation Function** | A function applied to the weighted sum to produce the neuron's output |
| **Step Function** | An activation that outputs 0 or 1 based on a threshold |
| **Sigmoid** | An activation that outputs a smooth value between 0 and 1 |
| **ReLU** | Rectified Linear Unit; outputs 0 for negative inputs, passes positive inputs unchanged |
| **Learning Rate** | A hyperparameter controlling how much weights change during each update |
| **Linearly Separable** | A dataset where classes can be divided by a straight line (or hyperplane) |
| **XOR Problem** | A classic example that a single perceptron cannot solve, demonstrating the need for multi-layer networks |
| **Decision Boundary** | The line (or surface) that separates different classes in the input space |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|---|---|
| Extremely simple and easy to understand | Can only solve linearly separable problems |
| Fast to train and compute | Cannot learn XOR or any non-linear pattern |
| Foundation for understanding all neural networks | Limited to binary classification |
| Guaranteed to converge if data is linearly separable | No hidden layers means no feature learning |
| Good pedagogical tool for learning AI concepts | Not useful in practice for real-world tasks |
| Helps understand adversarial attacks at neuron level | Step function is not differentiable (no gradient descent) |

---

## Key Takeaways

1. **A perceptron is the simplest artificial neuron** -- it multiplies inputs by weights, adds a bias, and applies an activation function to produce an output.

2. **Learning is just adjusting weights** -- when the perceptron makes a mistake, weights are nudged in the direction that would reduce the error.

3. **Activation functions matter** -- the step function is simple but not differentiable; sigmoid and ReLU enable gradient-based learning in deeper networks.

4. **A single perceptron can only draw a straight line** between classes. If the data is not linearly separable (like XOR), it fails.

5. **The XOR limitation led to an AI Winter** but was ultimately solved by stacking perceptrons into multi-layer networks.

6. **For security**: understanding that each neuron computes a simple weighted sum helps you understand why adversarial perturbations work -- small input changes accumulate across neurons to flip decisions.

7. **Perceptrons are the atoms of deep learning** -- every neural network, no matter how large, is built from units that work on this same principle.

---

*Previous: [Introduction to Deep Learning](introduction-to-deep-learning.md) | Next: [Neural Networks](neural-networks.md)*
