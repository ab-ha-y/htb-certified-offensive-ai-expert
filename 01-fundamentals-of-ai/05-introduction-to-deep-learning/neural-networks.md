# Neural Networks

## What are Neural Networks?

A neural network is a collection of **many perceptrons (neurons) connected together in layers**. While a single perceptron can only draw a straight line to separate data, a network of them working together can learn incredibly complex, non-linear patterns.

### The Team Analogy

Think of it like an intelligence analysis team:

- **Analysts at the front desk** (input layer) receive raw data -- satellite photos, intercepted messages, financial records.
- **Specialist teams** (hidden layers) each focus on different aspects -- one team looks for patterns in photos, another in text, another in numbers. Each team passes their findings to the next.
- **The senior analyst** (output layer) combines everything and makes the final call: "This is a threat" or "This is benign."

No single analyst can make the final decision alone. But organized in layers, each building on the previous team's work, they can make sophisticated judgments.

---

## Architecture: The Three Types of Layers

```
    INPUT LAYER          HIDDEN LAYERS           OUTPUT LAYER
    ===========       ==================         ============
   (raw data in)     (learned features)         (prediction)

        x1               h1    h5                   o1
         \              / |\ / |\                  /
          \            /  | X  | \                /
        x2 ----------+---|-/\-|--+------------- o2
          \          / \  |/  \| /\             /
           \        /   \ |    |/   \          /
        x3 --------+-----+----+-----+-------- o3
           \      / \   / \  / \   / \        /
            \    /   \ /   \/   \ /   \      /
        x4 ----+-----+----+-----+-----+---- o4
              |     |    |     |     |
           Layer 0  Layer 1  Layer 2   Layer 3

    x = input features (e.g., pixel values)
    h = hidden neurons (learned representations)
    o = output neurons (e.g., class probabilities)
```

### Input Layer

- Receives raw data. Each neuron represents one input feature.
- For an image: each pixel is one input neuron.
- For tabular data: each column is one input neuron.
- **Does no computation** -- just passes data forward.

### Hidden Layers

- Where the learning happens. "Hidden" because their values are not directly observed in the training data.
- **Each neuron** computes: weighted sum of inputs from previous layer + bias, then activation function.
- **More layers** = deeper network = can learn more complex patterns.
- Typical networks have 1 to hundreds of hidden layers.

### Output Layer

- Produces the final prediction.
- **Binary classification**: 1 neuron with sigmoid activation (outputs probability between 0 and 1).
- **Multi-class classification**: N neurons with softmax activation (outputs N probabilities that sum to 1).
- **Regression**: 1 neuron with no activation (outputs any real number).

---

## Forward Propagation: Step by Step

Forward propagation is the process of **passing input data through the network to get a prediction**. Data flows in one direction: input to output.

### Example: A Tiny Network

```
    Network: 2 inputs, 1 hidden layer (2 neurons), 1 output

    x1 = 0.5          h1              o1
          \          /    \          /
           w1=0.4  w3=0.6  w5=0.7  /
            \    /          \    /
             \/              \/
    x2 = 0.3  /\              /\
           w2=0.8  w4=0.2  w6=0.5
          /          \          \
                      h2              
```

**Step 1: Calculate hidden layer values**

```
h1_input = (x1 * w1) + (x2 * w2) + b1
         = (0.5 * 0.4) + (0.3 * 0.8) + 0.1
         = 0.20 + 0.24 + 0.1
         = 0.54

h1_output = sigmoid(0.54) = 1/(1 + e^(-0.54)) = 0.632

h2_input = (x1 * w3) + (x2 * w4) + b2
         = (0.5 * 0.6) + (0.3 * 0.2) + 0.1
         = 0.30 + 0.06 + 0.1
         = 0.46

h2_output = sigmoid(0.46) = 1/(1 + e^(-0.46)) = 0.613
```

**Step 2: Calculate output layer value**

```
o1_input = (h1_output * w5) + (h2_output * w6) + b3
         = (0.632 * 0.7) + (0.613 * 0.5) + 0.1
         = 0.442 + 0.307 + 0.1
         = 0.849

o1_output = sigmoid(0.849) = 0.700
```

**Step 3: Compare with expected output**

```
Predicted: 0.700
Expected:  1.0
Error:     We need a way to measure this... (enter loss functions)
```

---

## Loss Functions: Measuring How Wrong We Are

A loss function (also called cost function) quantifies **how far the prediction is from the true answer**. The goal of training is to minimize this value.

### Common Loss Functions

| Loss Function | Formula | When to Use |
|---|---|---|
| **Mean Squared Error (MSE)** | (1/n) * sum((predicted - actual)^2) | Regression problems |
| **Binary Cross-Entropy** | -[y*log(p) + (1-y)*log(1-p)] | Binary classification |
| **Categorical Cross-Entropy** | -sum(y_i * log(p_i)) | Multi-class classification |

### Example (continuing from above)

Using Binary Cross-Entropy with predicted = 0.700 and actual = 1.0:

```
Loss = -[1.0 * log(0.700) + (1-1.0) * log(1-0.700)]
     = -[1.0 * (-0.357) + 0]
     = 0.357
```

A loss of 0.357 tells us the network is wrong by a measurable amount. Now we need to figure out **which weights to adjust** to reduce this error.

---

## Backpropagation: Passing the Blame Backwards

Backpropagation is the algorithm that figures out **how much each weight contributed to the error** so we can adjust them. It works backwards from the output to the input.

### The Blame Game Analogy

Imagine a factory producing defective products:

1. **Quality inspector** (output) finds a defect and measures how bad it is (loss).
2. Inspector asks: "Who was the last person to touch this?" -- traces blame to the final assembly line (last hidden layer).
3. Final assembly asks: "Who gave me bad parts?" -- traces blame further back to the parts department (earlier hidden layer).
4. This continues all the way back to the raw materials (input layer).
5. Everyone responsible adjusts their process proportionally to their share of the blame.

### How It Works (The Chain Rule)

Backpropagation uses the **chain rule of calculus** to compute gradients -- the direction and magnitude of change needed for each weight.

```
    Forward Pass (left to right):
    ============================
    Input --> [w1] --> Hidden --> [w2] --> Output --> Loss
                                                      |
    Backward Pass (right to left):                    |
    ==============================                    |
    dL/dw1 <-- dL/dh <------- dL/dw2 <-- dL/do <----+

    dL/dw2 = dL/do * do/dw2      (how much does w2 affect the loss?)
    dL/dw1 = dL/do * do/dh * dh/dw1  (chain rule: blame passes through)
```

**In plain English:**

1. Calculate how much the loss changes when the output changes (dL/do).
2. Calculate how much the output changes when w2 changes (do/dw2).
3. Multiply them: that is how much w2 should change.
4. Keep going backwards: how much does the hidden layer affect the output? How much does w1 affect the hidden layer? Multiply the whole chain.

### The Update Rule

Once we know the gradient for each weight, we update:

```
w_new = w_old - learning_rate * gradient

If gradient is positive: weight is making loss worse --> decrease it
If gradient is negative: weight is helping --> increase it
```

### Step-by-Step Backpropagation (Continuing Our Example)

```
Given: predicted = 0.700, actual = 1.0, loss = 0.357

Step 1: Gradient of loss w.r.t. output
  dL/do = -(actual/predicted) + (1-actual)/(1-predicted)
        = -(1.0/0.700) + 0
        = -1.429

Step 2: Gradient of sigmoid
  do/dz = predicted * (1 - predicted)
        = 0.700 * 0.300
        = 0.210

Step 3: Combined gradient at output
  dL/dz = dL/do * do/dz = -1.429 * 0.210 = -0.300

Step 4: Gradient for w5 (connects h1 to output)
  dL/dw5 = dL/dz * h1_output = -0.300 * 0.632 = -0.190

Step 5: Update w5
  w5_new = 0.7 - 0.1 * (-0.190) = 0.7 + 0.019 = 0.719

  (Weight increased because gradient was negative -- 
   increasing w5 reduces the loss)
```

This process repeats for every weight in the network, moving backwards layer by layer.

---

## Optimizers: How to Adjust Weights Smartly

The update rule above is the simplest optimizer. More sophisticated ones exist:

### Stochastic Gradient Descent (SGD)

```
w = w - learning_rate * gradient
```

- Updates weights after each mini-batch of data.
- Simple but can be slow and get stuck in local minima.
- The "stochastic" part means it uses random subsets of data, adding helpful noise.

### Adam (Adaptive Moment Estimation)

- Maintains a **running average of gradients** (momentum) and **running average of squared gradients** (adaptive learning rates).
- Effectively gives each weight its own learning rate.
- Generally works well "out of the box" and is the most popular default choice.

### Optimizer Comparison

| Optimizer | Speed | Tuning Needed | When to Use |
|---|---|---|---|
| SGD | Slow but steady | Learning rate is critical | When you need fine-tuned control |
| SGD + Momentum | Faster | Learning rate + momentum | Good general choice |
| Adam | Fast convergence | Usually works with defaults | Default choice for most tasks |
| RMSprop | Fast | Moderate | RNNs and non-stationary problems |

---

## Worked Example: Classifying Simple Inputs

Let us train a tiny network to classify whether a point is "above" or "below" a diagonal line.

### The Problem

```
    y
    2 |  * (0.5, 1.8) Class 1    * (1.5, 1.5) Class 1
      |
    1 |
      |         o (1.0, 0.5) Class 0
    0 |  o (0.2, 0.1) Class 0
      +---------------------------
      0    0.5    1.0    1.5    2.0   x

    Class 1 = above the line (y > x)
    Class 0 = below the line (y < x)
```

### Network Architecture

```
    Input (2 neurons: x, y)
         |
    Hidden layer (3 neurons, ReLU)
         |
    Output (1 neuron, sigmoid)
```

### Training Process Summary

| Epoch | Training Loss | Accuracy |
|---|---|---|
| 1 | 0.693 (random guessing) | 50% |
| 10 | 0.452 | 75% |
| 50 | 0.121 | 95% |
| 100 | 0.034 | 100% |

The network learns by:

1. **Forward pass**: makes a prediction.
2. **Loss calculation**: measures how wrong it was.
3. **Backward pass**: computes gradients for every weight.
4. **Weight update**: adjusts all weights to reduce loss.
5. **Repeat** thousands of times.

After training, the network has effectively learned the rule "if y > x, output 1" -- but it learned this entirely from examples, without being told the rule.

---

## Putting It All Together: The Training Loop

```
+------------------+
| Initialize       |     Random weights
| weights randomly |
+--------+---------+
         |
         v
+------------------+
| Forward Pass     |     Input --> Predicted output
+--------+---------+
         |
         v
+------------------+
| Calculate Loss   |     How wrong was the prediction?
+--------+---------+
         |
         v
+------------------+
| Backpropagation  |     Compute gradients for all weights
+--------+---------+
         |
         v
+------------------+
| Update Weights   |     w = w - learning_rate * gradient
+--------+---------+
         |
         v
    Converged?  ----NO----> (loop back to Forward Pass)
         |
        YES
         |
         v
+------------------+
| Model is trained |
+------------------+
```

---

## Security and Offensive Angle

### Adversarial Examples

Neural networks are vulnerable to **adversarial examples** -- inputs with carefully crafted tiny perturbations that cause the network to misclassify with high confidence.

```
  Original Image          Perturbation          Adversarial Image
  +-----------+         +-----------+          +-----------+
  |           |         |           |          |           |
  |   PANDA   |    +    | (noise)   |    =     |   PANDA   |
  |   99.3%   |         | invisible |          |  "GIBBON"  |
  |           |         | to humans |          |   99.7%   |
  +-----------+         +-----------+          +-----------+
```

**How it works:**

1. Take a correctly classified input.
2. Compute the gradient of the loss with respect to the **input** (not the weights).
3. Add a tiny perturbation in the direction that **increases** the loss.
4. The network misclassifies the result, even though it looks identical to humans.

This is called the **Fast Gradient Sign Method (FGSM)**:

```
adversarial_input = original_input + epsilon * sign(gradient_of_loss_wrt_input)
```

### Model Extraction Attacks

An attacker can **steal a neural network** without accessing its internals:

```
    +------------------+
    | Target Model     |     The victim's model (API access only)
    | (Black Box)      |
    +--------+---------+
             |
     Query with many   |
     different inputs   |
             |
             v
    +------------------+
    | Collect outputs  |     Build a dataset of (input, output) pairs
    +--------+---------+
             |
             v
    +------------------+
    | Train a copy     |     Train your own model on the collected data
    | (Surrogate)      |     It mimics the target model's behavior
    +------------------+
```

**Steps of a model extraction attack:**

1. Send thousands of queries to the target model's API.
2. Record all input-output pairs.
3. Train a local "surrogate" neural network on these pairs.
4. The surrogate approximates the target model's behavior.
5. Use the surrogate to craft adversarial examples or understand the model's weaknesses.

**Why this matters for red teams:**

- Many companies expose ML models via APIs (fraud detection, content moderation).
- Extracting a copy lets you find adversarial inputs offline.
- Those adversarial inputs transfer well to the original model (transferability).

### Model Inversion Attacks

Attackers can **reconstruct training data** from a trained model:

- Query the model to learn what inputs maximize certain outputs.
- Reconstruct approximate versions of private training data (faces, medical records).
- This violates the privacy of individuals whose data was used to train the model.

---

## Key Terminology

| Term | Definition |
|---|---|
| **Neural Network** | A system of interconnected artificial neurons organized in layers that learns patterns from data |
| **Forward Propagation** | The process of passing input through the network to produce an output |
| **Backpropagation** | Algorithm that computes how much each weight contributes to the error, using the chain rule |
| **Loss Function** | A function that measures how far the prediction is from the actual value |
| **Gradient** | The direction and rate of change of the loss with respect to a weight; points toward increasing loss |
| **Learning Rate** | Hyperparameter that controls the step size of weight updates |
| **Epoch** | One complete pass through the entire training dataset |
| **Mini-batch** | A small random subset of training data used for one weight update |
| **Optimizer** | Algorithm that determines how weights are updated (SGD, Adam, etc.) |
| **Overfitting** | When the model memorizes training data but fails on new data |
| **Regularization** | Techniques to prevent overfitting (dropout, weight decay, early stopping) |
| **Softmax** | Activation function that converts a vector of numbers into probabilities summing to 1 |
| **Adversarial Example** | An input with carefully crafted perturbations designed to fool a neural network |
| **Model Extraction** | Attack where an adversary recreates a model by querying it and training a copy |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|---|---|
| Can learn any computable function (universal approximators) | Require large amounts of training data |
| Automatically discover useful features | Computationally expensive to train |
| Handle non-linear relationships easily | Susceptible to adversarial examples |
| Scale to very complex problems | Black box -- difficult to interpret |
| Transfer learning enables reuse across tasks | Can overfit, especially with small datasets |
| Flexible architecture for diverse data types | Many hyperparameters to tune |
| Well-supported by frameworks (PyTorch, TensorFlow) | Vulnerable to model extraction and inversion attacks |

---

## Key Takeaways

1. **A neural network is layers of connected neurons** -- input layer receives data, hidden layers learn features, output layer makes predictions.

2. **Forward propagation** pushes data through the network. **Backpropagation** computes how to adjust weights to reduce error.

3. **The chain rule is the engine of learning** -- it lets us trace blame for errors back through every layer to every weight.

4. **Loss functions measure error** -- MSE for regression, cross-entropy for classification. The entire goal of training is to minimize loss.

5. **Adam is the default optimizer** for most tasks. SGD is simpler but requires more tuning.

6. **Training is iterative** -- forward pass, compute loss, backward pass, update weights, repeat for thousands of epochs.

7. **For security**: neural networks are vulnerable to adversarial examples (tiny perturbations that cause misclassification), model extraction (stealing the model via API queries), and model inversion (reconstructing private training data). Understanding how forward and backward passes work is essential for understanding these attacks.

---

*Previous: [Perceptrons](perceptrons.md) | Next: [Convolutional Neural Networks](convolutional-neural-networks.md)*
