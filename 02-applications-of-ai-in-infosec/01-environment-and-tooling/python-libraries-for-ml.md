# Python Libraries for ML

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Applications of AI in InfoSec | Section: Python Libraries for ML

---

## Table of Contents

1. [Why Use Libraries Instead of Writing Algorithms From Scratch?](#1-why-use-libraries-instead-of-writing-algorithms-from-scratch)
2. [scikit-learn: The Classical ML Toolkit](#2-scikit-learn-the-classical-ml-toolkit)
3. [The scikit-learn API Pattern](#3-the-scikit-learn-api-pattern)
4. [A Minimal scikit-learn Example](#4-a-minimal-scikit-learn-example)
5. [PyTorch: The Deep Learning Toolkit](#5-pytorch-the-deep-learning-toolkit)
6. [Core PyTorch Objects](#6-core-pytorch-objects)
7. [A Minimal PyTorch Example](#7-a-minimal-pytorch-example)
8. [scikit-learn vs. PyTorch: When to Use Which](#8-scikit-learn-vs-pytorch-when-to-use-which)
9. [Supporting Libraries You Will See Everywhere](#9-supporting-libraries-you-will-see-everywhere)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. Why Use Libraries Instead of Writing Algorithms From Scratch?

### The Analogy

You could, in principle, forge your own hammer from raw iron every time you wanted to hang a picture frame. Nobody does this. You buy a hammer, because thousands of engineering hours already went into making a good one, and you would rather spend your time and energy on the picture frame.

**scikit-learn** and **PyTorch** are the "hammers" of machine learning. Both implement the mathematically tricky, performance-critical parts of ML (matrix operations, optimization algorithms, numerical stability tricks) so that you can focus on the actual problem: is this email spam, is this binary malicious, is this packet an attack.

### What These Libraries Actually Give You

| Without a Library | With a Library |
|--------------------|-----------------|
| Hand-implement gradient descent, matrix calculus, and numerical stability fixes | Call `.fit()` and the library handles all of that correctly and efficiently |
| Manually manage memory layouts for fast matrix math | Get highly optimized C/C++/CUDA code under a friendly Python interface |
| Re-derive standard algorithms (Naive Bayes, Random Forest, backpropagation) from papers | Import a well-tested, widely-used implementation used by millions of practitioners |
| Debug your own bugs in core math | Rely on code that has been battle-tested across the entire ML community |

---

## 2. scikit-learn: The Classical ML Toolkit

**scikit-learn** (imported as `sklearn`) is a Python library for **classical machine learning**: algorithms like Naive Bayes, Decision Trees, Random Forests, Logistic Regression, Support Vector Machines, and k-Means. It also provides the supporting tools every ML project needs: data splitting, scaling, encoding, and evaluation metrics.

### Why It Is the Default Starting Point

- **One consistent API** for dozens of very different algorithms -- once you learn the pattern (Section 3), you can swap Naive Bayes for a Random Forest by changing one line.
- **Batteries included**: preprocessing, model selection, and metrics all live in the same ecosystem, so you rarely need to leave scikit-learn for a full classical-ML pipeline.
- **Runs comfortably on a CPU** -- no GPU required, which matters for the tabular, moderately-sized datasets typical of spam filtering and network intrusion detection (Sections 6-7 of this module).

### The scikit-learn Ecosystem, at a Glance

```
    +-----------------------------------------------------------+
    |                       scikit-learn                        |
    +-----------------------------------------------------------+
    | sklearn.model_selection  | train_test_split, GridSearchCV |
    | sklearn.preprocessing    | StandardScaler, OneHotEncoder  |
    | sklearn.impute           | SimpleImputer                  |
    | sklearn.feature_extraction.text | CountVectorizer, TfidfVectorizer |
    | sklearn.naive_bayes      | MultinomialNB, GaussianNB      |
    | sklearn.tree             | DecisionTreeClassifier          |
    | sklearn.ensemble         | RandomForestClassifier          |
    | sklearn.svm              | SVC                              |
    | sklearn.metrics          | accuracy_score, confusion_matrix, classification_report |
    +-----------------------------------------------------------+
```

---

## 3. The scikit-learn API Pattern

Almost every scikit-learn object -- whether it is a model, a scaler, or an encoder -- follows the same small set of methods. Learn this pattern once and it applies everywhere.

| Method | What It Does | Plain English |
|--------|--------------|----------------|
| `.fit(X, y)` | Learns from the training data | "Study these examples and their answers" |
| `.predict(X)` | Produces predictions for new data | "Given what you learned, what is your answer for this new input?" |
| `.transform(X)` | Applies a learned transformation (used by scalers/encoders, not models) | "Apply the conversion rule you learned to this new data" |
| `.fit_transform(X)` | Learn AND apply in one step | Shortcut for `.fit(X)` followed by `.transform(X)` |
| `.score(X, y)` | Returns a default performance metric | "How well did you do on this labeled data?" |

```
    THE SCIKIT-LEARN ESTIMATOR PATTERN
    ===================================

    Every "estimator" object (model, scaler, encoder) works like this:

    +------------+     .fit(X, y)     +------------+
    |  Raw       | -----------------> | Fitted     |
    |  Estimator |                    | Estimator  |
    +------------+                    +------------+
                                            |
                                            | .predict(X_new)  (models)
                                            | .transform(X_new) (scalers/encoders)
                                            v
                                      +------------+
                                      |  Output    |
                                      +------------+
```

---

## 4. A Minimal scikit-learn Example

Here is a complete, runnable example: training a classifier to distinguish two simple classes based on a couple of numeric features (a toy stand-in for something like "is this network connection an attack based on packet count and byte count?").

```python
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, classification_report

# 1. Toy dataset: [packet_count, byte_count] -> 0 = normal, 1 = attack
X = np.array([
    [10, 500], [12, 480], [11, 510], [9, 470],    # normal traffic
    [500, 50000], [480, 49000], [510, 51500], [470, 48000],  # flood attack
])
y = np.array([0, 0, 0, 0, 1, 1, 1, 1])

# 2. Split into train and test sets
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.25, random_state=42
)

# 3. Create and train the model
clf = RandomForestClassifier(n_estimators=100, random_state=42)
clf.fit(X_train, y_train)

# 4. Predict on unseen data
y_pred = clf.predict(X_test)

# 5. Evaluate
print("Accuracy:", accuracy_score(y_test, y_pred))
print(classification_report(y_test, y_pred, target_names=["normal", "attack"]))

# 6. Use the trained model on a brand-new connection
new_connection = np.array([[495, 49500]])
print("Prediction (0=normal, 1=attack):", clf.predict(new_connection))
```

Notice the shape of the code: `train_test_split` -> `fit` -> `predict` -> metrics. This exact skeleton reappears in every scikit-learn project in this module, just with different data and a different model class swapped in.

---

## 5. PyTorch: The Deep Learning Toolkit

**PyTorch** (imported as `torch`) is a library for building and training **neural networks** -- models made of layers of interconnected mathematical units, capable of learning far more complex patterns than classical algorithms, at the cost of needing much more data and compute.

### Why Deep Learning Needs a Different Kind of Library

Classical algorithms in scikit-learn have a fixed structure that the library already knows how to optimize internally -- you never see the "learning" happen, you just call `.fit()`. Neural networks, by contrast, have an architecture *you* design (how many layers, what size, what connects to what), so you need a library that gives you the building blocks (layers, activation functions, loss functions) and handles the genuinely hard part automatically: computing how to adjust millions of internal numbers to reduce error. That hard part is called **automatic differentiation**, and it is the core feature PyTorch provides.

### The Analogy

If scikit-learn is a hammer, PyTorch is a full box of LEGO bricks plus a robot arm that automatically figures out how to reshape your LEGO structure to better match a picture you showed it. You design the structure (the neural network architecture); PyTorch's engine (**autograd**) figures out exactly how each brick (each internal number, called a **weight**) should change to make the whole structure a little more correct, and repeats this thousands of times.

---

## 6. Core PyTorch Objects

| Object | What It Is | Plain English |
|--------|-----------|-----------------|
| **Tensor** (`torch.Tensor`) | A multi-dimensional array of numbers, like a NumPy array, but able to track gradients and run on a GPU | The basic "box of numbers" every piece of data becomes |
| **`nn.Module`** | The base class for building a neural network layer or an entire model | A blueprint: you subclass it to describe your network's architecture |
| **Autograd** | PyTorch's automatic differentiation engine | The robot arm that figures out how to adjust every weight to reduce error, without you doing the calculus by hand |
| **Loss function** (e.g., `nn.CrossEntropyLoss`) | Measures how wrong the model's predictions are | The scoring system training tries to minimize |
| **Optimizer** (e.g., `torch.optim.Adam`) | The algorithm that actually updates the weights using the gradients autograd computed | The hand that moves each LEGO brick a small step in the right direction |
| **DataLoader** | Feeds data to the model in manageable batches during training | The conveyor belt that hands the model one tray of examples at a time |

```
    THE PYTORCH TRAINING LOOP
    ===========================

    +------------+   forward pass   +------------+   compare to   +------------+
    |   Batch    | ---------------> |   Model    | -------------> |   Loss     |
    |   of Data  |                  | (nn.Module)|   true labels  |  Function  |
    +------------+                  +------------+                +------------+
                                                                         |
                          weights updated <---- optimizer.step() <------+
                                |                       ^
                                |                       |
                                v               loss.backward()
                          +------------+        (autograd computes
                          |  Updated   |         the gradients)
                          |  Weights   |
                          +------------+
```

---

## 7. A Minimal PyTorch Example

A minimal, runnable example: a tiny neural network learning the same normal-vs-attack traffic idea from Section 4, this time with PyTorch.

```python
import torch
import torch.nn as nn
import torch.optim as optim

# 1. Toy dataset (same idea as before): [packet_count, byte_count]
X = torch.tensor([
    [10., 500.], [12., 480.], [11., 510.], [9., 470.],
    [500., 50000.], [480., 49000.], [510., 51500.], [470., 48000.],
])
y = torch.tensor([0, 0, 0, 0, 1, 1, 1, 1])  # 0 = normal, 1 = attack

# Normalize features so both columns are on a similar scale
X = (X - X.mean(dim=0)) / X.std(dim=0)

# 2. Define a tiny neural network: 2 inputs -> 8 hidden units -> 2 class scores
class TrafficClassifier(nn.Module):
    def __init__(self):
        super().__init__()
        self.layer1 = nn.Linear(2, 8)   # 2 input features -> 8 hidden units
        self.relu = nn.ReLU()           # non-linearity
        self.layer2 = nn.Linear(8, 2)   # 8 hidden units -> 2 output classes

    def forward(self, x):
        x = self.layer1(x)
        x = self.relu(x)
        x = self.layer2(x)
        return x

model = TrafficClassifier()

# 3. Define loss function and optimizer
loss_fn = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.05)

# 4. Training loop
for epoch in range(200):
    optimizer.zero_grad()          # reset gradients from the previous step
    outputs = model(X)             # forward pass: model makes predictions
    loss = loss_fn(outputs, y)     # how wrong were we?
    loss.backward()                # autograd computes gradients
    optimizer.step()               # update weights using those gradients

    if epoch % 50 == 0:
        print(f"Epoch {epoch:3d}  Loss: {loss.item():.4f}")

# 5. Inference on a new, unseen connection
new_connection = torch.tensor([[495., 49500.]])
new_connection = (new_connection - X.mean(dim=0)) / X.std(dim=0)
with torch.no_grad():   # disable gradient tracking, we are not training now
    prediction = model(new_connection)
    predicted_class = torch.argmax(prediction, dim=1)
    print("Prediction (0=normal, 1=attack):", predicted_class.item())
```

Every piece of this code -- `zero_grad()`, `backward()`, `step()` -- exists because, unlike scikit-learn's one-line `.fit()`, you are manually driving the training loop. This gives you far more control (and far more responsibility) than the classical-ML API pattern in Section 3.

---

## 8. scikit-learn vs. PyTorch: When to Use Which

| Aspect | scikit-learn | PyTorch |
|--------|--------------|---------|
| **Algorithm family** | Classical ML (trees, Naive Bayes, SVM, linear models, clustering) | Neural networks / deep learning |
| **Typical data size** | Small to medium, tabular | Large; especially strong on images, text sequences, audio |
| **Hardware** | CPU is usually sufficient | GPU strongly recommended for anything beyond toy examples |
| **API style** | High-level: `.fit()` / `.predict()` in one or two lines | Lower-level: you write the training loop yourself |
| **Interpretability** | Often high (e.g., decision trees, feature importances) | Often low ("black box") unless you add explainability tooling |
| **Good fit in this module** | Spam classification (Naive Bayes), network anomaly detection (Random Forest) | Malware classification (CNN via transfer learning) |

### The Decision in Practice

```
    Is your data tabular with a moderate number of features,
    and do you want fast iteration + interpretability?
    |
    +-- YES --> Start with scikit-learn (Naive Bayes, Random Forest, etc.)
    |
    +-- NO  --> Is your data unstructured (images, raw text sequences,
                audio) or do you need to squeeze out maximum accuracy
                on a large dataset?
                |
                +-- YES --> Use PyTorch (build or fine-tune a neural network)
```

In real security work, you often use both in the same pipeline: scikit-learn for quick baselines and feature preprocessing, PyTorch when you need a deep architecture -- exactly the pattern you will see in Section 8 (Malware Classification), where a binary image is classified using a PyTorch-based CNN, but scikit-learn's `train_test_split` and metrics are still used for the data split and evaluation.

---

## 9. Supporting Libraries You Will See Everywhere

| Library | Role | Example Use in This Module |
|---------|------|------------------------------|
| **NumPy** | Fast numerical arrays and math operations; the foundation both scikit-learn and PyTorch build on | Manipulating feature arrays before feeding them to a model |
| **pandas** | Tabular data structures (`DataFrame`) and data manipulation | Loading and cleaning the spam and NSL-KDD CSV datasets |
| **matplotlib / seaborn** | Plotting and visualization | Histograms of feature distributions, confusion matrix heatmaps |
| **torchvision** | Pretrained computer-vision models and image utilities for PyTorch | Loading a pretrained ResNet50 for malware image classification |

---

## 10. Key Takeaways

- **scikit-learn** is the go-to library for classical ML: consistent `.fit()` / `.predict()` / `.transform()` API across dozens of algorithms, ideal for tabular security data like emails-as-features or network flow records.
- **PyTorch** is the go-to library for deep learning: you define a network architecture as an `nn.Module`, and PyTorch's **autograd** engine automatically computes how to adjust every weight to reduce error.
- The scikit-learn workflow is high-level and largely automatic (`.fit()` hides the training loop); the PyTorch workflow is lower-level and explicit (you write the forward pass, loss computation, backward pass, and optimizer step yourself).
- Choose scikit-learn for structured/tabular data where interpretability and speed matter; choose PyTorch for unstructured data (images, long text sequences) or when you need the extra capacity of a neural network.
- NumPy, pandas, and matplotlib/seaborn are the supporting cast that show up in nearly every notebook regardless of which core ML library you use.

*Next up: Datasets -- how to load, inspect, and understand the structure of the data these libraries will actually train on.*
