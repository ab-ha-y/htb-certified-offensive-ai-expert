# Recurrent Neural Networks (RNNs)

## What are RNNs?

A Recurrent Neural Network (RNN) is a type of neural network designed to handle **sequential data** -- data where the order matters. Unlike regular neural networks that process each input independently, an RNN has a **memory**: it passes information from one step to the next.

### The Book Reading Analogy

When you read a book, you do not start from scratch at each word. You carry **context** from previous words and sentences:

- Reading "The bank was steep and covered in grass" -- you know "bank" means a riverbank.
- Reading "The bank approved my loan application" -- you know "bank" means a financial institution.

You understood the meaning of "bank" because of the words that came **before** it. An RNN works the same way -- it maintains a **hidden state** that carries information from previous steps in the sequence.

A regular neural network is like reading each word on a separate flashcard with no context. An RNN is like reading the words in order, remembering what came before.

---

## Why Sequence Order Matters

Consider these sentences:

```
"The dog bit the man"     vs.     "The man bit the dog"
```

Same words, completely different meaning. The **order** of elements in a sequence carries critical information. RNNs are built to capture this.

### Types of Sequential Data

| Data Type | Example | Why Order Matters |
|---|---|---|
| Text | "I love this movie" | Word order determines meaning and sentiment |
| Time Series | Stock prices over days | Past prices influence future predictions |
| Audio/Speech | Spoken words | Sound order forms words and sentences |
| Network Traffic | Packet sequences | Sequence of requests reveals attack patterns |
| DNA | ATCGGCTAA... | Gene sequences encode biological instructions |
| Keystrokes | Typing patterns | Timing and order identify individual users |

---

## How an RNN Works

### The Recurrence Loop

The key feature of an RNN is the **loop** -- the hidden state from the previous time step feeds back into the network at the current time step.

```
REGULAR NEURAL NETWORK (no memory):

    x1 --> [NN] --> y1
    x2 --> [NN] --> y2       Each input processed independently
    x3 --> [NN] --> y3


RECURRENT NEURAL NETWORK (has memory):

                +-------+
                |       |
                v       |
    x1 --> [RNN Cell] --+--> y1
                |
          h1 (hidden state carries forward)
                |
                v
    x2 --> [RNN Cell] --+--> y2
                |
          h2 (updated hidden state)
                |
                v
    x3 --> [RNN Cell] --+--> y3
                |
          h3 (final hidden state)
```

### Unrolled View

The loop can be "unrolled" across time steps to show what actually happens:

```
UNROLLED RNN (processing the word "HELLO"):

   h0        h1        h2        h3        h4
   (init)     |         |         |         |
    |         v         v         v         v
    +-->[RNN]--+-->[RNN]--+-->[RNN]--+-->[RNN]--+-->[RNN]--> h5
         ^         ^         ^         ^         ^
         |         |         |         |         |
        "H"       "E"       "L"       "L"       "O"
         |         |         |         |         |
         v         v         v         v         v
        y1        y2        y3        y4        y5

    At each step:
      h_t = activation(W_hh * h_{t-1}  +  W_xh * x_t  +  b)
      y_t = W_hy * h_t + b_y

    W_hh = weights for hidden-to-hidden (same at every step!)
    W_xh = weights for input-to-hidden  (same at every step!)
    W_hy = weights for hidden-to-output (same at every step!)
```

**Key insight: the same weights are shared across all time steps.** The RNN applies the same transformation at every step, but the hidden state carries accumulated context.

### Step-by-Step Processing

**Step 1:** Initialize hidden state h0 (usually all zeros).

**Step 2:** At time step t=1, receive input x1 ("H"):
- Combine x1 with h0 using weights.
- Apply activation function (usually tanh).
- Produce h1 (new hidden state) and y1 (output).

**Step 3:** At time step t=2, receive input x2 ("E"):
- Combine x2 with h1 (which contains info about "H").
- Apply activation function.
- Produce h2 (now contains info about "H" and "E") and y2.

**Step 4:** Continue for each element in the sequence...

**Step 5:** The final hidden state h5 contains a compressed summary of the entire sequence "HELLO."

---

## The Vanishing Gradient Problem

### What Goes Wrong

RNNs are trained using backpropagation through time (BPTT) -- backpropagation applied across all time steps. The problem: as gradients flow backwards through many time steps, they get **multiplied by the same weight matrix** at each step.

```
    GRADIENT FLOW DURING BACKPROPAGATION:

    Loss at step 5
         |
         v
    [Step 5] <-- gradient = 1.0
         |
         v  (multiply by weight ~0.5)
    [Step 4] <-- gradient = 0.5
         |
         v  (multiply by weight ~0.5)
    [Step 3] <-- gradient = 0.25
         |
         v  (multiply by weight ~0.5)
    [Step 2] <-- gradient = 0.125
         |
         v  (multiply by weight ~0.5)
    [Step 1] <-- gradient = 0.0625   <-- Almost zero!

    By the time the gradient reaches early steps,
    it has vanished to near zero.
    The network CANNOT learn long-range dependencies.
```

### Why It Matters

- If the gradient vanishes, early time steps **stop learning**.
- The network forgets information from the beginning of long sequences.
- Example: in "The cat, which was sitting on the mat and purring loudly, **was** happy" -- the network needs to remember "cat" (singular) to predict "was" (not "were") many words later. Vanilla RNNs fail at this.

### The Opposite Problem: Exploding Gradients

If the weight is greater than 1, gradients **explode** instead of vanishing:

```
    0.5^10 = 0.001  (vanishing)
    2.0^10 = 1024   (exploding!)
```

Exploding gradients cause training to become unstable. The fix is **gradient clipping** -- if the gradient exceeds a threshold, scale it down.

---

## LSTM: Long Short-Term Memory

LSTM is the most popular solution to the vanishing gradient problem. It was introduced in 1997 and remains widely used.

### The Filing Cabinet Analogy

Think of an LSTM as an office worker with a **filing cabinet** (cell state) and three assistants (gates):

1. **Forget Gate**: "Which old files should I throw away?" -- decides what information from the previous cell state to discard.
2. **Input Gate**: "Which new files should I add?" -- decides what new information to store in the cell state.
3. **Output Gate**: "Which files should I show to my boss?" -- decides what information from the cell state to output.

### LSTM Architecture

```
LSTM CELL:

                    Cell State (C_t) -- the "highway" for information
    C_{t-1} ---->[x]----------[+]----------------------------> C_t
                  ^             ^
                  |             |
              Forget Gate    Input Gate
              "What to       "What new info
               forget"        to add"
                  ^             ^
                  |             |
    h_{t-1} --+--+--+------+--+--+------+--+--+--> h_t
              |     |      |     |      |     |
              | Forget|    | Input|    |Output|
              | Gate  |    | Gate |    | Gate |
              |sigmoid|    |sig*tanh|  |sigmoid|
              +-------+    +-------+  +-------+
                  ^             ^          ^
                  |             |          |
                 x_t           x_t        x_t
                (input)       (input)    (input)

    [x] = element-wise multiplication
    [+] = element-wise addition
```

**How the gates work:**

| Gate | Function | Activation | Output Range |
|---|---|---|---|
| Forget Gate | Decides what to erase from memory | Sigmoid | 0 (forget) to 1 (keep) |
| Input Gate | Decides what new info to write to memory | Sigmoid * tanh | Filters new candidate values |
| Output Gate | Decides what to output from memory | Sigmoid | 0 (hide) to 1 (reveal) |

**Why LSTM solves vanishing gradients:**

The cell state acts as a **highway** -- information can flow across many time steps with minimal modification. The additive nature of the cell state update (using + instead of multiplication) prevents gradients from vanishing.

---

## GRU: Gated Recurrent Unit

GRU is a **simplified version of LSTM** introduced in 2014. It achieves similar performance with fewer parameters and faster training.

### GRU vs LSTM

```
LSTM: 3 gates (forget, input, output) + cell state + hidden state
GRU:  2 gates (reset, update)         + hidden state only
```

| Feature | LSTM | GRU |
|---|---|---|
| Number of gates | 3 | 2 |
| Separate cell state | Yes | No (merged into hidden state) |
| Parameters | More | Fewer (~25% less) |
| Training speed | Slower | Faster |
| Performance | Slightly better on complex tasks | Comparable on most tasks |
| When to use | Long sequences, complex dependencies | When speed matters, simpler tasks |

**Rule of thumb:** Start with GRU. Switch to LSTM if GRU performance is insufficient.

---

## Worked Example: Predicting the Next Character

Let us build an intuition for how an RNN predicts the next character in a sequence.

### Training Data

```
Input text: "hello world"
```

The RNN learns from character pairs:

| Input Char | Target (Next Char) |
|---|---|
| h | e |
| e | l |
| l | l |
| l | o |
| o | (space) |
| (space) | w |
| w | o |
| o | r |
| r | l |
| l | d |

### Network Setup

```
    One-hot encoded input       Hidden state (4 units)     Output (27 chars)
    (27 chars: a-z + space)
    
    [0,0,0,0,0,0,0,1,0,...] --> [RNN Cell] --> [0.1, 0.8, 0.05, 0.02, ...]
         "h"                       |                    ^
                              h_t carries              "e" has highest
                              to next step             probability
```

### Step-by-Step Prediction

**Step 1: Input "h"**

```
Input: "h" (one-hot vector, position 7 = 1, rest = 0)
Hidden state: h0 = [0, 0, 0, 0]  (initialized to zeros)

Computation:
  h1 = tanh(W_xh * "h" + W_hh * h0 + b)
  h1 = tanh([0.2, -0.1, 0.5, 0.3])  (after matrix multiplications)
  h1 = [0.197, -0.100, 0.462, 0.291]

Output probabilities (after softmax):
  a: 0.02, b: 0.01, ..., e: 0.35, ..., h: 0.05, ...
                          ^^^^
                     Highest! Predicts "e"
```

**Step 2: Input "e"**

```
Input: "e" (one-hot vector)
Hidden state: h1 = [0.197, -0.100, 0.462, 0.291]  (carries info about "h")

Computation:
  h2 = tanh(W_xh * "e" + W_hh * h1 + b)
  
  Now the network knows it has seen "h" then "e"
  It predicts "l" with high probability (having learned "hel..." is common)

Output: l: 0.42  <-- highest probability
```

**After training, the network can generate text:**

```
Seed: "h"
Generated: "h" --> "e" --> "l" --> "l" --> "o" --> " " --> "w" --> "o" --> ...
```

---

## RNN Variants: Types of Sequence Tasks

```
ONE-TO-ONE:          ONE-TO-MANY:         MANY-TO-ONE:
(Standard NN)        (Image captioning)   (Sentiment analysis)

  [x] --> [y]        [x] --> [y1]         [x1]
                           --> [y2]        [x2] --> [y]
                           --> [y3]        [x3]


MANY-TO-MANY:                    MANY-TO-MANY:
(Machine translation)            (Video frame labeling)

  [x1] [x2] [x3]                [x1] --> [y1]
        |                       [x2] --> [y2]
  [y1] [y2] [y3] [y4]           [x3] --> [y3]
```

---

## Transformers: The Modern Evolution

While RNNs were the dominant architecture for sequence data from ~2015-2017, **Transformers** (introduced in the 2017 paper "Attention Is All You Need") have largely replaced them.

### Why Transformers Replaced RNNs

| Aspect | RNNs | Transformers |
|---|---|---|
| Processing | Sequential (one step at a time) | Parallel (all steps at once) |
| Training speed | Slow (cannot parallelize) | Fast (fully parallelizable) |
| Long-range dependencies | Struggles even with LSTM/GRU | Handles via attention mechanism |
| Memory | Fixed-size hidden state | Attention over entire sequence |
| Scale | Difficult to scale up | Scales extremely well (GPT-4, etc.) |

### The Key Idea: Attention

Instead of compressing the entire sequence into a fixed-size hidden state, Transformers use **attention** to directly look at any part of the input sequence when making predictions.

```
RNN approach (bottleneck):
    [The] [cat] [sat] [on] [mat] --> [fixed hidden state] --> prediction
                                     ^^^^^^^^^^^^^^^^^^^^
                                     Must compress EVERYTHING
                                     into this small vector

Transformer approach (attention):
    [The] [cat] [sat] [on] [mat]
      |     |     |     |    |
      +-----+-----+-----+----+--> prediction
      
    Can look at ANY word directly when making prediction
    "Which words should I pay attention to right now?"
```

**Important note:** You do not need deep knowledge of Transformers for this section. They are covered in the Generative AI module. Just know that they evolved from RNNs and solved many of their limitations.

---

## Security and Offensive Angle

### 1. Generating Phishing Text

RNNs (and their successor, Transformers) can generate convincing text for social engineering:

```
    Training Data:                    Generated Phishing Email:
    +------------------+             +-------------------------+
    | Thousands of     |             | Dear valued customer,   |
    | real corporate   |    Train    | We noticed unusual      |
    | emails           | ---------> | activity on your account|
    +------------------+             | Please verify your      |
                                     | credentials at...       |
                                     +-------------------------+
```

**How attackers use this:**
- Train on leaked corporate email datasets to mimic a company's writing style.
- Generate personalized spear-phishing emails at scale.
- Create contextually appropriate follow-up messages.
- Modern LLMs (built on Transformers, which evolved from RNNs) make this trivially easy.

### 2. Analyzing Network Traffic Sequences

Network packets arrive in sequences, making them perfect for RNN analysis:

```
    Normal traffic:     [SYN] [SYN-ACK] [ACK] [DATA] [DATA] [FIN]
                         |      |        |      |      |      |
                         v      v        v      v      v      v
                       [RNN]->[RNN]--->[RNN]->[RNN]->[RNN]->[RNN]
                                                              |
                                                              v
                                                         NORMAL (98%)

    Attack traffic:     [SYN] [SYN] [SYN] [SYN] [SYN] [SYN]   (SYN flood)
                         |     |     |     |     |     |
                         v     v     v     v     v     v
                       [RNN]->[RNN]->[RNN]->[RNN]->[RNN]->[RNN]
                                                          |
                                                          v
                                                     ATTACK (97%)
```

**Security applications:**
- **Intrusion Detection Systems (IDS)**: RNNs detect anomalous packet sequences that signature-based systems miss.
- **Command and Control (C2) detection**: RNNs identify beaconing patterns -- periodic, regular communication between malware and C2 servers.
- **Encrypted traffic analysis**: Even without decrypting, the sequence of packet sizes and timing can reveal attack patterns.

### 3. Keystroke Dynamics

Every person types differently -- their timing between keystrokes is unique, like a fingerprint.

```
    User types "password":

    Key:    p     a     s     s     w     o     r     d
    Time:   0    120   85    90   150   110   95    130  (milliseconds)

    +---+---+---+---+---+---+---+---+
    |120| 85| 90|150|110| 95|130|   |  <-- timing sequence
    +---+---+---+---+---+---+---+---+
                    |
                    v
              [LSTM/GRU RNN]
                    |
                    v
         User: "alice" (92% confidence)
```

**Offensive uses:**
- **Impersonation detection**: If an attacker gains access to a user's account, their typing pattern will differ from the legitimate user. RNN-based systems can detect this.
- **User profiling**: Keystroke dynamics can identify individuals across different platforms.
- **Bypassing**: Understanding how these systems work helps attackers develop techniques to mimic typing patterns or evade detection.

### 4. Malware Behavior Sequences

Malware executes sequences of API calls. RNNs can classify malware families based on these sequences:

```
    Ransomware API call sequence:
    [CreateFile] -> [ReadFile] -> [CryptEncrypt] -> [WriteFile] -> [DeleteFile]
         |              |              |                |              |
         v              v              v                v              v
       [RNN] -------> [RNN] -------> [RNN] -------> [RNN] -------> [RNN]
                                                                      |
                                                                      v
                                                              Ransomware (96%)
```

---

## Key Terminology

| Term | Definition |
|---|---|
| **RNN** | Recurrent Neural Network; a neural network with loops that process sequential data |
| **Hidden State** | A vector that carries information from previous time steps; the network's "memory" |
| **Time Step** | One position in the sequence (one word, one character, one packet) |
| **Recurrence** | The process of feeding the hidden state back into the network at the next time step |
| **BPTT** | Backpropagation Through Time; backpropagation applied across all time steps of a sequence |
| **Vanishing Gradient** | When gradients shrink to near zero during backpropagation through many time steps, preventing learning |
| **Exploding Gradient** | When gradients grow exponentially during backpropagation, causing unstable training |
| **Gradient Clipping** | Technique to prevent exploding gradients by capping gradient magnitude |
| **LSTM** | Long Short-Term Memory; an RNN variant with gates that solve the vanishing gradient problem |
| **GRU** | Gated Recurrent Unit; a simplified LSTM with fewer parameters |
| **Gate** | A learned mechanism that controls information flow (forget, input, output gates in LSTM) |
| **Cell State** | The LSTM's long-term memory highway that carries information across time steps |
| **Attention** | A mechanism that lets the model directly access any part of the input sequence |
| **Transformer** | Modern architecture using attention mechanisms that has largely replaced RNNs |
| **Sequence-to-Sequence** | A model that takes a sequence as input and produces a sequence as output (e.g., translation) |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|---|---|
| Designed for sequential data (text, time series, audio) | Vanishing gradient problem in vanilla RNNs |
| Maintains memory of previous inputs via hidden state | Sequential processing -- cannot parallelize (slow training) |
| LSTM/GRU effectively handle long-range dependencies | Fixed-size hidden state limits memory capacity |
| Weight sharing across time steps reduces parameters | Largely superseded by Transformers for most NLP tasks |
| Well-suited for variable-length sequences | Difficult to train on very long sequences (1000+ steps) |
| Good for real-time sequential processing | Harder to interpret than attention-based models |
| Effective for time series forecasting and anomaly detection | Requires careful hyperparameter tuning |

---

## Key Takeaways

1. **RNNs process sequential data** by maintaining a hidden state that carries information from previous time steps -- giving the network "memory."

2. **Order matters**: the same data in a different order means something different. RNNs are designed to capture this.

3. **The vanishing gradient problem** prevents vanilla RNNs from learning long-range dependencies -- gradients shrink to near zero as they flow backwards through many time steps.

4. **LSTM and GRU solve this** with gating mechanisms. LSTM uses three gates (forget, input, output) and a cell state. GRU simplifies this to two gates (reset, update).

5. **Transformers have largely replaced RNNs** for most sequence tasks because they can process all positions in parallel and handle long-range dependencies better via attention.

6. **For security**: RNNs (and their successors) are used offensively for generating phishing text and impersonation. Defensively, they analyze network traffic sequences, detect anomalous API call patterns in malware, and perform user authentication via keystroke dynamics. Understanding sequential processing is essential for attacking and defending against sequence-based AI systems.

7. **Know the evolution**: Perceptron --> Neural Network --> RNN (added memory) --> LSTM/GRU (fixed vanishing gradients) --> Transformer (attention, parallel processing). Each step solved limitations of the previous architecture.

---

*Previous: [Convolutional Neural Networks](convolutional-neural-networks.md) | Next section: Introduction to Generative AI*
