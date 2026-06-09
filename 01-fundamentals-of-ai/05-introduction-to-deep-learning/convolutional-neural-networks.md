# Convolutional Neural Networks (CNNs)

## What are CNNs?

A Convolutional Neural Network (CNN) is a type of neural network specifically designed to **process grid-like data, especially images**. While a regular neural network connects every neuron to every neuron in the next layer, a CNN uses a smarter approach: it slides small filters across the image to detect local patterns.

### The Flashlight Analogy

Imagine you are in a dark room looking at a large painting. You have a small flashlight that illuminates only a tiny patch at a time.

- You **slide the flashlight** across the painting systematically -- left to right, top to bottom.
- At each position, you notice local features: "Here is an edge," "Here is a curve," "Here is a dark spot."
- After scanning the entire painting, you have a **mental map** of all the local features.
- Your brain then combines these local features to understand the whole scene: "This is a landscape with mountains."

A CNN works the same way. The "flashlight" is called a **filter** (or **kernel**), and it slides across the input image looking for patterns.

---

## The Convolution Operation

"Convolution" is a mathematical operation where a small matrix (the filter/kernel) slides over the input image, computing a weighted sum at each position.

### ASCII Example: A 3x3 Filter Scanning a 5x5 Image

**Input Image (5x5 pixel grid):**

```
+---+---+---+---+---+
| 1 | 0 | 1 | 0 | 1 |
+---+---+---+---+---+
| 0 | 1 | 0 | 1 | 0 |
+---+---+---+---+---+
| 1 | 0 | 1 | 0 | 1 |
+---+---+---+---+---+
| 0 | 1 | 0 | 1 | 0 |
+---+---+---+---+---+
| 1 | 0 | 1 | 0 | 1 |
+---+---+---+---+---+
```

**Filter/Kernel (3x3) -- detects vertical edges:**

```
+----+----+----+
| -1 |  0 |  1 |
+----+----+----+
| -1 |  0 |  1 |
+----+----+----+
| -1 |  0 |  1 |
+----+----+----+
```

**Step 1: Place filter at top-left corner**

```
[1  0  1] 0  1       Filter:  [-1  0  1]
[0  1  0] 1  0                [-1  0  1]
[1  0  1] 0  1                [-1  0  1]
 0  1  0  1  0
 1  0  1  0  1

Calculation:
(1*-1) + (0*0) + (1*1) +
(0*-1) + (1*0) + (0*1) +
(1*-1) + (0*0) + (1*1)
= -1 + 0 + 1 + 0 + 0 + 0 + -1 + 0 + 1
= 0
```

**Step 2: Slide filter one position to the right**

```
 1 [0  1  0] 1        Filter:  [-1  0  1]
 0 [1  0  1] 0                 [-1  0  1]
 1 [0  1  0] 1                 [-1  0  1]
 0  1  0  1  0
 1  0  1  0  1

Calculation:
(0*-1) + (1*0) + (0*1) +
(1*-1) + (0*0) + (1*1) +
(0*-1) + (1*0) + (0*1)
= 0 + 0 + 0 + -1 + 0 + 1 + 0 + 0 + 0
= 0
```

**Step 3: Continue sliding across and down...**

The filter visits every valid position, producing a smaller **output grid** called a **feature map**.

```
Input (5x5)  *  Filter (3x3)  =  Feature Map (3x3)

+---+---+---+---+---+                      +---+---+---+
| 1 | 0 | 1 | 0 | 1 |                      | 0 | 0 | 0 |
+---+---+---+---+---+     [-1  0  1]       +---+---+---+
| 0 | 1 | 0 | 1 | 0 | *   [-1  0  1]  =   | 0 | 0 | 0 |
+---+---+---+---+---+     [-1  0  1]       +---+---+---+
| 1 | 0 | 1 | 0 | 1 |                      | 0 | 0 | 0 |
+---+---+---+---+---+
| 0 | 1 | 0 | 1 | 0 |
+---+---+---+---+---+
| 1 | 0 | 1 | 0 | 1 |
+---+---+---+---+---+
```

In this case, the feature map is all zeros because the checkerboard pattern has no vertical edges. A different filter would detect different features.

---

## Key Components of a CNN

### 1. Convolutional Layer

- Applies multiple filters to the input.
- Each filter detects a different feature (edges, corners, textures).
- **Parameters are shared**: the same filter scans the entire image, drastically reducing the number of parameters compared to a fully connected layer.

```
One filter produces one feature map:

  Input Image ----[Filter 1]----> Feature Map 1 (horizontal edges)
              ----[Filter 2]----> Feature Map 2 (vertical edges)
              ----[Filter 3]----> Feature Map 3 (diagonal edges)
              ----[Filter 4]----> Feature Map 4 (corners)
```

### 2. Activation (ReLU)

- Applied after each convolution.
- Replaces all negative values with zero.
- Introduces non-linearity (without it, stacking layers would be pointless).

```
Before ReLU:              After ReLU:
+----+----+----+          +----+----+----+
| -2 |  3 | -1 |          |  0 |  3 |  0 |
+----+----+----+          +----+----+----+
|  5 | -4 |  2 |          |  5 |  0 |  2 |
+----+----+----+          +----+----+----+
| -1 |  0 |  6 |          |  0 |  0 |  6 |
+----+----+----+          +----+----+----+
```

### 3. Pooling Layer

- **Reduces the spatial size** of feature maps, making computation more efficient.
- Most common: **Max Pooling** -- takes the maximum value in each small region.

```
Max Pooling (2x2, stride 2):

Input (4x4):                    Output (2x2):
+---+---+---+---+               +---+---+
| 1 | 3 | 2 | 1 |               | 3 | 2 |
+---+---+---+---+    ------>    +---+---+
| 5 | 2 | 8 | 0 |               | 5 | 8 |
+---+---+---+---+               +---+---+
| 4 | 1 | 3 | 7 |
+---+---+---+---+
| 0 | 6 | 2 | 4 |
+---+---+---+---+

Top-left 2x2 block: max(1,3,5,2) = 5  ... wait, let me recalculate:
  [1,3,5,2] -> max = 5
  [2,1,8,0] -> max = 8
  [4,1,0,6] -> max = 6
  [3,7,2,4] -> max = 7

Corrected Output (2x2):
+---+---+
| 5 | 8 |
+---+---+
| 6 | 7 |
+---+---+
```

**Why pooling helps:**
- Reduces computation by shrinking feature maps.
- Provides some **translation invariance** -- the cat is still a cat whether it is in the left or right of the image.
- Reduces overfitting by providing an abstracted form of the features.

### 4. Fully Connected Layer

- After the convolutional and pooling layers extract features, the feature maps are **flattened** into a 1D vector.
- This vector feeds into a regular neural network (fully connected layers) for final classification.

---

## CNN Architecture: The Full Picture

```
 INPUT         CONV        POOL       CONV        POOL      FLATTEN    FC      OUTPUT
 IMAGE        LAYER 1     LAYER 1    LAYER 2     LAYER 2              LAYERS

+------+    +--------+   +------+   +--------+   +----+   +------+  +----+  +------+
|      |    |        |   |      |   |        |   |    |   |      |  |    |  |      |
| 28x28|--->| 26x26  |-->| 13x13|-->| 11x11  |-->| 5x5|-->| 1x   |->|    |->| Cat  |
| x1   |    | x32    |   | x32  |   | x64    |   | x64|   | 1600 |  | 128|  | Dog  |
|      |    | filters|   |      |   | filters|   |    |   |      |  |    |  | Bird |
+------+    +--------+   +------+   +--------+   +----+   +------+  +----+  +------+

             Detects      Shrinks    Detects      Shrinks  Reshapes  Learns  Final
             edges,       size       complex      size     to 1D     combos  class
             textures                patterns             vector    of       probs
                                                          features
```

**Data flow:**
1. Input image (e.g., 28x28 grayscale) enters the network.
2. First convolutional layer applies 32 filters, detecting simple features (edges, gradients).
3. Pooling shrinks the spatial dimensions by half.
4. Second convolutional layer applies 64 filters, detecting complex features (eyes, ears, shapes).
5. More pooling shrinks again.
6. Feature maps are flattened into a single long vector.
7. Fully connected layers combine features and output class probabilities.

---

## Feature Maps and Filters: What the Network Actually Learns

```
    Layer 1 Filters             Layer 2 Filters          Layer 3 Filters
    (learned automatically)     (learned automatically)  (learned automatically)

    +--+  +--+  +--+           +---+  +---+             +-----+  +-----+
    |/ |  |--| |\ |           |eye|  |ear|             | cat |  | dog |
    +--+  +--+  +--+           +---+  +---+             |face |  |face |
    edges  lines  edges        parts of objects          whole objects

    SIMPLE -----------------------------------------> COMPLEX
    FEATURES                                          FEATURES
```

The CNN **discovers these features on its own** during training. Nobody programs "look for edges" -- the network figures out that edges are useful for recognizing objects.

---

## Worked Example: How a CNN Identifies Cat vs Dog

### Step 1: Input

A 224x224 color image of a cat enters the network as a 3D array: 224 (height) x 224 (width) x 3 (RGB channels).

### Step 2: Early Convolutional Layers

```
Layer 1 detects:          Layer 2 detects:
  - Horizontal edges       - Corners
  - Vertical edges         - Curves
  - Diagonal edges         - Color gradients
  - Color boundaries       - Simple textures
```

### Step 3: Middle Convolutional Layers

```
Layer 3-4 detects:         Layer 5-6 detects:
  - Eye-like shapes         - Cat ears (triangular)
  - Nose patterns           - Dog ears (floppy)
  - Fur textures            - Whiskers
  - Paw shapes              - Snout shapes
```

### Step 4: Deep Convolutional Layers

```
Layer 7+ detects:
  - Full cat face pattern
  - Full dog face pattern
  - Body shapes
  - Breed-specific features
```

### Step 5: Fully Connected Layers

```
All detected features are combined:
  - Triangular ears (0.9) + whiskers (0.8) + small nose (0.7) --> CAT: 95%
  - Floppy ears (0.1) + no whiskers (0.2) + large snout (0.1) --> DOG: 5%
```

### Step 6: Output

```
  Cat: 95.2%  <-- highest probability, so the prediction is "Cat"
  Dog:  4.8%
```

---

## Famous CNN Architectures

| Architecture | Year | Key Innovation | Depth | Error Rate (ImageNet) |
|---|---|---|---|---|
| **LeNet-5** | 1998 | First practical CNN; used for digit recognition | 5 layers | N/A (MNIST) |
| **AlexNet** | 2012 | Deep CNN + ReLU + dropout; sparked the deep learning revolution | 8 layers | 15.3% |
| **VGGNet** | 2014 | Very deep with small (3x3) filters only | 16-19 layers | 7.3% |
| **GoogLeNet/Inception** | 2014 | Inception modules with parallel filter sizes | 22 layers | 6.7% |
| **ResNet** | 2015 | Skip connections (residual learning); enabled very deep networks | 152 layers | 3.6% |
| **EfficientNet** | 2019 | Compound scaling of depth, width, and resolution | Variable | 2.9% |

### The AlexNet Moment (2012)

AlexNet winning the ImageNet competition in 2012 by a huge margin is considered the event that launched the modern deep learning era. Before AlexNet, hand-crafted feature engineering was the norm. After AlexNet, deep learning took over computer vision.

### ResNet's Skip Connections

```
Normal path:     Input --> [Conv] --> [Conv] --> Output

ResNet path:     Input --> [Conv] --> [Conv] --> (+) --> Output
                   |                             ^
                   +-------- skip connection ----+

The skip connection lets the gradient flow directly backwards,
solving the vanishing gradient problem in very deep networks.
```

---

## Security and Offensive Angle

### 1. CAPTCHA Solving with CNNs

CNNs can break image-based CAPTCHAs with high accuracy:

```
CAPTCHA Image:     +------------------+
                   |  7 K x 2 M       |     CNN processes this image
                   +------------------+
                           |
                           v
                   CNN classifies each character:
                   [7] [K] [x] [2] [M]
                   
                   Accuracy: 85-95% on many CAPTCHA systems
```

**How attackers use this:**
- Train a CNN on labeled CAPTCHA images (or use pre-trained models).
- Automate account creation, web scraping, or brute-force attacks.
- This is why modern CAPTCHAs have moved to behavioral analysis (reCAPTCHA v3).

### 2. Deepfake Generation and Detection

**Generation (Offensive):**
- GANs (which use CNNs internally) generate realistic fake faces and video.
- Can impersonate executives for social engineering attacks.
- Voice + face deepfakes enable highly convincing video calls.

**Detection (Defensive):**
- CNNs can detect deepfakes by spotting artifacts:
  - Inconsistent lighting or shadows
  - Blending boundaries around face edges
  - Irregular eye reflections
  - Compression artifacts specific to generated images

```
    Real Face              Deepfake
    +----------+           +----------+
    |          |           |          |
    | Consistent|          | Artifacts |
    | lighting  |          | at edges  |
    | Natural   |          | Irregular |
    | textures  |          | textures  |
    +----------+           +----------+
         |                      |
         v                      v
    CNN says: REAL (98%)   CNN says: FAKE (94%)
```

### 3. Adversarial Patches

Physical-world adversarial attacks that fool CNNs:

```
    Normal Stop Sign            Stop Sign + Adversarial Patch
    +----------+                +----------+
    |   STOP   |                |   STOP   |
    |          |                | +------+ |
    |          |                | |PATCH | |
    |          |                | +------+ |
    +----------+                +----------+
         |                           |
         v                           v
    CNN: "Stop Sign" (99%)      CNN: "Speed Limit 45" (87%)
```

- Attackers can print adversarial patches and stick them on real objects.
- This threatens autonomous vehicles, surveillance systems, and any CNN-based vision system.
- Research has shown patches on clothing can make people "invisible" to person detectors.

### 4. Malware Visualization

Malware binaries can be converted to images and classified by CNNs:

```
    Malware Binary          Visualized as Image        CNN Classification
    010110100101...   --->  +----------+        --->   Trojan (92%)
    110010110010...         | Patterns |               Ransomware (5%)
    001101001100...         | visible  |               Benign (3%)
                            +----------+
```

Security researchers convert malware to grayscale images where byte values map to pixel intensities. Different malware families produce visually distinct patterns that CNNs can classify.

---

## Key Terminology

| Term | Definition |
|---|---|
| **CNN** | Convolutional Neural Network; a neural network designed for processing grid-structured data like images |
| **Convolution** | Mathematical operation of sliding a filter over input data, computing weighted sums at each position |
| **Filter/Kernel** | A small matrix of learnable weights that detects specific features (edges, textures, etc.) |
| **Feature Map** | The output of applying a filter to an input; highlights where a specific feature exists |
| **Stride** | The number of pixels the filter moves between positions (stride 1 = move one pixel at a time) |
| **Padding** | Adding zeros around the input border so the filter can cover edge pixels |
| **Pooling** | Reducing spatial dimensions by summarizing regions (max pooling, average pooling) |
| **Max Pooling** | Taking the maximum value from each pooling region |
| **Flatten** | Reshaping a multi-dimensional feature map into a 1D vector for the fully connected layers |
| **Transfer Learning** | Using a CNN pretrained on a large dataset (ImageNet) and fine-tuning it for a new task |
| **Skip Connection** | A shortcut that lets data bypass one or more layers (used in ResNet) |
| **Adversarial Patch** | A physical-world pattern designed to fool a CNN when placed in the scene |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|---|---|
| State-of-the-art for image classification and object detection | Require large labeled datasets for training |
| Parameter sharing via filters reduces model size | Computationally expensive (need GPUs) |
| Translation invariance -- detects features regardless of position | Vulnerable to adversarial examples and patches |
| Automatically learns hierarchical features | Not ideal for sequential data (use RNNs/Transformers instead) |
| Transfer learning enables reuse of pretrained models | Can be fooled by rotations and scaling not seen in training |
| Well-understood architectures (ResNet, VGG, etc.) | Interpretability is limited -- hard to explain decisions |
| Efficient for grid-like data (images, spectrograms, heatmaps) | Require careful architecture design for new problem domains |

---

## Key Takeaways

1. **CNNs are neural networks designed for images** -- they use small sliding filters (kernels) to detect local patterns, building from simple features (edges) to complex ones (objects).

2. **The convolution operation** slides a filter across the image, computing weighted sums. Each filter detects one type of feature, producing a feature map.

3. **Key architecture: Convolution --> ReLU --> Pooling --> Repeat --> Flatten --> Fully Connected --> Output.** This pattern is the backbone of most CNNs.

4. **Pooling reduces size and provides translation invariance** -- the network recognizes a cat whether it appears in the top-left or bottom-right of the image.

5. **CNNs learn features automatically** -- early layers detect edges, middle layers detect parts, deep layers detect whole objects. No manual feature engineering needed.

6. **ResNet's skip connections** solved the vanishing gradient problem, enabling networks with 100+ layers.

7. **For security**: CNNs are used offensively for CAPTCHA solving, deepfake generation, and adversarial attacks on vision systems. Defensively, they detect deepfakes and classify malware visualizations. Understanding how filters work helps you understand why adversarial patches can fool these systems.

---

*Previous: [Neural Networks](neural-networks.md) | Next: [Recurrent Neural Networks](recurrent-neural-networks.md)*
