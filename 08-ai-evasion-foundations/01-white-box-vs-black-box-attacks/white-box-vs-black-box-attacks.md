# White-box vs Black-box Attacks

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - Foundations | Section: White-box vs Black-box Attacks

---

## Table of Contents

1. [What Is an Evasion Attack?](#1-what-is-an-evasion-attack)
2. [The Core Idea: Attacker Knowledge Levels](#2-the-core-idea-attacker-knowledge-levels)
3. [White-box Attacks](#3-white-box-attacks)
4. [Black-box Attacks](#4-black-box-attacks)
5. [The Math You Need: Gradients, Simply](#5-the-math-you-need-gradients-simply)
6. [Worked Numeric Example: Evading a Toy Linear Classifier](#6-worked-numeric-example-evading-a-toy-linear-classifier)
7. [Comparing the Two Threat Models](#7-comparing-the-two-threat-models)
8. [Gray-box: The Realistic Middle Ground](#8-gray-box-the-realistic-middle-ground)
9. [Security Angle](#9-security-angle)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. What Is an Evasion Attack?

Before diving into "white-box" and "black-box," let's ground the whole topic: what is an **evasion attack**?

An evasion attack happens at **inference time** -- after a model has already been trained and deployed. The attacker does not touch the training data or the training process. Instead, the attacker takes an input (an email, an image, a file, a network packet) and **modifies it slightly** so that a trained model misclassifies it, while the thing it represents in the real world stays functionally the same.

### The Analogy

Imagine an airport metal detector trained to beep whenever it senses a knife-shaped mass of metal. An evasion attack is like wrapping the knife in a very specific shape of foil that the detector's pattern-matching doesn't recognize as "knife" anymore -- the knife is still a knife, still just as sharp, but the detector's decision process has been fooled by a clever repackaging.

In machine learning terms:

- The **model** has learned a **decision boundary** -- an invisible line (or surface, in higher dimensions) that separates "malicious" from "benign," or "spam" from "ham."
- An evasion attack **nudges an input across that boundary** without changing what the input actually *is* to a human or to the real-world system it targets.

```
   A DECISION BOUNDARY, AND A SAMPLE BEING NUDGED ACROSS IT

    feature_2
       ^
       |        BENIGN region
       |        o   o
       |      o    o     o
       |    o    o
       |  o    o   \
       |__________o_\____________________  <- decision boundary
       |            \
       |  x   x     \x  <-- original malicious sample (x)
       |    x    x   \
       |       x       \___
       |                    x'  <-- adversarial version, nudged
       |                        just across the line (still "x"
       |        MALICIOUS       in every way that matters to the
       |        region          attacker, but now scores "benign")
       +----------------------------------------> feature_1
```

The entire question this module and the next two modules (9 and 10) explore is: **how does an attacker figure out which direction, and how far, to nudge a sample?** The answer depends heavily on how much the attacker knows about the model. That's exactly the white-box vs. black-box distinction.

---

## 2. The Core Idea: Attacker Knowledge Levels

Every evasion attack starts with a question: **"How much do I know about the target model?"** This single question splits the entire field of evasion attacks into two broad families, with a spectrum of nuance in between.

### The Analogy

Think of trying to pick a combination lock:

- **White-box** = you have the manufacturer's schematics. You know exactly how the tumblers, springs, and pins are arranged. You can calculate the exact combination directly.
- **Black-box** = you have no schematics. All you can do is turn the dial and see if it clicks open or not. You have to *feel your way* to the answer through trial and error, or find an identical lock model to practice on first.

In ML terms, "knowing the schematics" means knowing the model's **architecture** (what type of model it is, how many layers, what activation functions), its **parameters/weights** (the learned numeric values), and being able to compute its **gradients** (which direction changes to the input would push the output in a certain direction). "Turning the dial and listening for a click" means only being able to send an input in and observe an output -- nothing more.

```
                    ATTACKER KNOWLEDGE SPECTRUM

  FULL KNOWLEDGE                                        NO KNOWLEDGE
  (White-box)                                            (Black-box)
       |------------------------|------------------------|
       |                        |                         |
  Architecture +           Gray-box:                  Only see
  weights +                partial info                inputs/outputs
  gradients                (e.g. know the               (e.g. public
  available                architecture,                API, web
                            not the weights)             form, SaaS
                                                          product)
```

---

## 3. White-box Attacks

### 3.1 Definition

A **white-box attack** assumes the attacker has **complete access to the model internals**:

- The exact **architecture** (e.g., "this is a 5-layer neural network with ReLU activations and a softmax output").
- The exact **parameters/weights** learned during training.
- The ability to compute **gradients** -- i.e., for any input, the attacker can calculate exactly how a tiny change to each input feature would change the model's output.
- Often, knowledge of the **training data distribution** and **preprocessing pipeline** (how raw inputs get turned into features).

### 3.2 Why This Matters

If you know the exact mathematical function the model computes, you can use calculus to find the *most efficient* way to cross the decision boundary. Instead of guessing, you compute the **gradient of the loss function with respect to the input** -- this tells you, for every input feature, whether increasing or decreasing it will push the model's prediction toward "benign" (or whatever the attacker's target class is), and by roughly how much.

This is the same machinery used during model *training* (backpropagation, gradient descent), just pointed in a different direction: instead of adjusting the model's weights to reduce loss, the attacker adjusts the *input* to increase the loss (i.e., make the model wrong) or push the prediction toward a specific class.

### 3.3 When Does White-box Access Happen in Real Life?

| Scenario | Why the Attacker Has White-box Access |
|---|---|
| **Open-source / published models** | Many malware classifiers, spam filters, and vision models are open-source (or their architecture is published in a paper) -- weights are downloadable. |
| **On-device / embedded models** | A model shipped inside a mobile app, an antivirus engine, or an IoT device can be extracted from the binary/firmware. |
| **Insider access** | A red teamer engaged to test a company's internal fraud-detection model may be handed the model artifact directly. |
| **Leaked or stolen weights** | Breaches, misconfigured cloud storage, or supply-chain compromise can leak model files. |
| **After a successful model-stealing attack** | An attacker first performs model extraction (see Module 1's security angle) against a black-box API, then has a white-box *surrogate* to attack -- this is the bridge into Transferability, covered in the next file. |

### 3.4 Common White-box Attack Techniques (Preview)

You will study these in depth in later modules; here's the essential mental model of how they use the gradient:

```
   WHITE-BOX EVASION LOOP (conceptual)

   +-------------------+
   | Start with input x|   e.g. a malware feature vector, correctly
   | (correctly         |   classified as "malicious"
   | classified)        |
   +---------+---------+
             |
             v
   +-------------------+
   | Compute gradient   |   "Which direction, for each feature,
   | of loss w.r.t. x   |    increases the model's error?"
   +---------+---------+
             |
             v
   +-------------------+
   | Step x a small     |   x_new = x + epsilon * direction
   | amount in that     |   (epsilon = step size, kept small so the
   | direction          |    sample still "looks" like the original)
   +---------+---------+
             |
             v
   +-------------------+
   | Check: does model  |----NO---> repeat, take another small step
   | now misclassify?   |
   +---------+---------+
             |
            YES
             |
             v
   +-------------------+
   | Adversarial example|
   | found: x_adv        |
   +-------------------+
```

Two of the most famous algorithms that follow this loop:

- **FGSM (Fast Gradient Sign Method)** -- takes exactly one big step in the gradient direction. Fast, but sometimes crude.
- **PGD (Projected Gradient Descent)** -- takes many small steps, checking after each one that the perturbation is still within an allowed "budget" (so the sample doesn't drift too far from the original). Slower, but more reliable at finding a successful perturbation.

You'll study FGSM, PGD, and related techniques in detail in Module 9. For now, the key point is: **white-box attacks are efficient because they use exact gradient information instead of guessing.**

---

## 4. Black-box Attacks

### 4.1 Definition

A **black-box attack** assumes the attacker has **no access to the model internals**. All the attacker can do is:

- Submit an input.
- Observe the output -- which might be just a label ("spam" / "not spam"), a confidence score (e.g., "92% malicious"), or sometimes nothing more than a binary "accepted" / "blocked" decision.

This is exactly the situation you face when attacking a commercial spam filter, a malware-scanning API (like an online file-upload scanner), a content-moderation endpoint, or any "AI-as-a-Service" product where you only interact through a public interface.

### 4.2 Why This Is Harder

Without gradients, the attacker cannot directly calculate "which direction reduces the model's confidence in the correct class." They must instead **infer** that information indirectly. Two dominant strategies exist:

#### Strategy A: Query-based (Gradient Estimation / Search)

The attacker repeatedly queries the model with slightly different inputs and uses the pattern of outputs to *estimate* what the gradient probably looks like, or to directly search for a successful perturbation via optimization methods that don't require gradients at all (e.g., evolutionary search, random search, boundary-following algorithms).

```
   QUERY-BASED BLACK-BOX ATTACK (conceptual)

   +-------------------+
   | Start with input x|
   +---------+---------+
             |
             v
   +-------------------+
   | Try small random   |   x' = x + small random noise (many
   | perturbations,      |   directions tried)
   | query the model     |
   | for each             |
   +---------+---------+
             |
             v
   +-------------------+
   | Observe which        |   "Confidence in 'malicious' dropped
   | perturbations reduce |    from 0.95 to 0.88 when I changed
   | confidence in the    |    feature_3 this way"
   | true/current class   |
   +---------+---------+
             |
             v
   +-------------------+
   | Keep the perturbation|  Repeat, gradually walking the sample
   | that helped the most,|  toward and across the boundary
   | discard the rest,    |
   | repeat               |
   +-------------------+
```

This can require thousands to millions of queries depending on the method and the number of input features -- which is often the attacker's biggest practical constraint (rate limiting, cost per query, detection of abuse).

#### Strategy B: Transfer-based (Surrogate Models)

The attacker **trains their own model** ("a surrogate") that tries to approximate the target's behavior, using either:

- Public data similar to what the target was likely trained on, or
- Data collected by querying the target and recording input/output pairs (this doubles as a *model extraction* attack).

Once a surrogate exists, the attacker performs a **white-box attack against their own surrogate** (since they have full access to it), and then -- relying on the phenomenon of **transferability** -- tries the resulting adversarial example against the real target, hoping it fools it too.

This strategy is powerful enough, and surprising enough, that it gets its own dedicated file: see [Transferability](../02-transferability/transferability.md).

### 4.3 What Information Is Actually Observable in Black-box Settings?

Not all black-box access is equal. The amount of output detail matters a lot:

| Access Level | What the Attacker Sees | Difficulty |
|---|---|---|
| **Full confidence scores** | Probability/confidence for every class (e.g., 0.92 malicious, 0.08 benign) | Easier -- confidence scores act like a "warmer/colder" signal guiding the search |
| **Top-1 label only** | Just the final decision ("malicious") with no score | Harder -- much less signal per query |
| **Hard label, rate-limited** | Only a label, and only a handful of queries allowed before being blocked | Very hard -- forces highly query-efficient algorithms or a transfer-based approach |
| **No direct access at all** | Attacker can only observe downstream effects (e.g., "did my phishing email get delivered or not?") | Hardest -- essentially blind; transfer-based attacks become the primary option |

---

## 5. The Math You Need: Gradients, Simply

You'll see the word "gradient" constantly in evasion-attack literature. Let's build the intuition from the ground up, with zero assumed background.

### 5.1 What Is a Function, Here?

A trained classifier is really just a mathematical function. It takes in a list of numbers (the **features** of your input) and spits out a number (or a set of numbers) representing "how malicious" or "how spammy" the input looks.

Call this function `f`. If your input has two features, you can write:

```
f(x1, x2) = some number representing "maliciousness score"
```

For example, a very simple (and unrealistically clean) linear model might compute:

```
f(x1, x2) = 2*x1 + 3*x2 - 5

If f(x1, x2) > 0  -->  classify as MALICIOUS
If f(x1, x2) <= 0 -->  classify as BENIGN
```

### 5.2 What Is a Gradient?

The **gradient** of a function tells you, for each input variable, **which direction to move it to increase the function's output the fastest**, and roughly by how much a small nudge in that direction changes the output.

Think of the function's output as the *height of a hill* at your current location, where your location is described by the input features. The gradient is like a compass that always points **uphill** -- toward increasing height (increasing output).

```
   GRADIENT AS "UPHILL COMPASS"

        output (height)
           ^
           |            .*
           |         .*    <-- gradient points this way (uphill)
           |      .*
           |   .*
           |.*
           +------------------------> input feature value

   If you are standing at a point on this curve and you want the
   output to go UP, move in the direction the gradient points.
   If you want the output to go DOWN, move in the OPPOSITE direction.
```

For our simple linear example `f(x1, x2) = 2*x1 + 3*x2 - 5`, the gradient is just the coefficients:

```
gradient = (df/dx1, df/dx2) = (2, 3)
```

This says: "increasing x1 by 1 unit increases f by 2. Increasing x2 by 1 unit increases f by 3." Since 3 > 2, nudging x2 is a more "efficient" way to change the output per unit of change -- x2 has more leverage on this particular model's decision.

### 5.3 Why Attackers Care About the Gradient

If a sample is currently classified MALICIOUS (`f(x1, x2) > 0`) and the attacker wants it classified BENIGN, they want to **decrease** `f`. The gradient tells them exactly which direction increases `f` -- so they move in the **opposite** direction of the gradient:

```
x_new = x_old - epsilon * gradient

epsilon = a small step size (how big a nudge to take)
```

This single idea -- "move against the gradient to decrease the score, move with the gradient to increase it" -- is the mathematical heart of essentially every white-box evasion technique (FGSM, PGD, and others you'll study in Module 9).

For a black-box attacker who cannot compute this gradient directly, the entire game becomes: **estimate what this gradient probably looks like, using nothing but queries and the outputs they return.**

---

## 6. Worked Numeric Example: Evading a Toy Linear Classifier

Let's make everything above completely concrete with numbers you can follow by hand.

### 6.1 The Setup

Imagine an extremely simplified malware detector that looks at just two features of a file:

- `x1` = number of suspicious API calls (e.g., calls related to process injection), normalized to a 0-10 scale.
- `x2` = entropy of the file's byte content (a rough measure of "how random/packed" the file looks), normalized to a 0-10 scale.

The (toy) linear model computes a maliciousness score:

```
score(x1, x2) = 2*x1 + 3*x2 - 20

If score > 0  -->  MALICIOUS
If score <= 0 -->  BENIGN
```

This is a **linear classifier**: the decision boundary is a straight line, `2*x1 + 3*x2 - 20 = 0`, i.e. `2*x1 + 3*x2 = 20`.

```
   x2 (entropy)
   10|
     |         MALICIOUS region
    8|                (score > 0)
     |          o
    6|        o
     |      o
    4|    o . . . . . . . boundary: 2*x1 + 3*x2 = 20
     |  o
    2|          BENIGN region
     |          (score <= 0)
    0+---------------------------------> x1 (suspicious API calls)
     0    2    4    6    8    10
```

### 6.2 A Real Malicious Sample

Suppose a real malware sample has:

```
x1 = 8   (8 suspicious API calls, on our normalized scale)
x2 = 6   (moderately high entropy -- somewhat packed)

score = 2*8 + 3*6 - 20 = 16 + 18 - 20 = 14

14 > 0  -->  classified MALICIOUS  (correctly so)
```

### 6.3 White-box Attack: Compute the Gradient and Step Against It

The attacker has full access to this model (white-box), so the gradient is trivial to read off:

```
gradient = (d score/dx1, d score/dx2) = (2, 3)
```

To *decrease* the score (push toward BENIGN), the attacker moves *against* the gradient direction, i.e., decreases x1 and x2, with x2 given more weight since its coefficient (3) is larger than x1's (2).

Let's use a step size `epsilon = 1` and move directly against the gradient direction (normalized here just for simplicity of illustration -- real implementations often use the exact gradient vector or its sign):

```
x1_new = x1_old - epsilon*2 = 8 - 2 = 6
x2_new = x2_old - epsilon*3 = 6 - 3 = 3

score_new = 2*6 + 3*3 - 20 = 12 + 9 - 20 = 1

1 > 0 --> still MALICIOUS, but much closer to the boundary (score
dropped from 14 to 1)
```

One more small step gets us across:

```
x1_new2 = 6 - 0.5*2 = 5
x2_new2 = 3 - 0.5*3 = 1.5

score_new2 = 2*5 + 3*1.5 - 20 = 10 + 4.5 - 20 = -5.5

-5.5 <= 0 --> now classified BENIGN
```

The attacker found, in two guided steps, that reducing suspicious API calls from 8 to 5 and reducing entropy from 6 to 1.5 flips the classification -- and because they had the gradient, they knew *exactly* which features mattered most (entropy, with coefficient 3, moved the needle more per unit change than API call count, with coefficient 2).

### 6.4 Black-box Attack: Same Goal, No Gradient

Now imagine the attacker can only query the model as a black box -- they submit `(x1, x2)` pairs and get back only "MALICIOUS" or "BENIGN" (no score, no gradient).

Starting again from `(x1=8, x2=6)` = MALICIOUS, the attacker has to probe:

```
Query 1: (x1=8, x2=5)  --> model says MALICIOUS   (score = 2*8+3*5-20 = 11)
Query 2: (x1=7, x2=6)  --> model says MALICIOUS   (score = 2*7+3*6-20 = 12)
Query 3: (x1=8, x2=4)  --> model says MALICIOUS   (score = 2*8+3*4-20 = 8)
Query 4: (x1=6, x2=4)  --> model says MALICIOUS   (score = 2*6+3*4-20 = 4)
Query 5: (x1=5, x2=3)  --> model says BENIGN      (score = 2*5+3*3-20 = -1)  SUCCESS
```

Notice the attacker needed **five queries of trial and error** to reach roughly the same result the white-box attacker found in **two calculated steps**. In a real scenario with hundreds of features instead of two, this gap becomes enormous -- which is exactly why black-box attackers so often fall back on transfer-based strategies (train a surrogate, attack it with white-box methods, then transfer the result) rather than pure query-based search. That surrogate strategy is the subject of the next file.

---

## 7. Comparing the Two Threat Models

| Dimension | White-box | Black-box |
|---|---|---|
| **Model architecture known?** | Yes | Usually no (may be guessed) |
| **Model weights/parameters known?** | Yes | No |
| **Gradients computable?** | Yes, directly | No -- must be estimated or avoided |
| **Primary technique** | Gradient-based optimization (FGSM, PGD, C&W, etc.) | Query-based search, gradient estimation, or surrogate/transfer-based attacks |
| **Number of "attempts" needed** | Few -- gradient points directly at the answer | Many -- from dozens to millions of queries, or zero queries if purely transfer-based |
| **Realistic scenarios** | Open-source models, extracted/leaked weights, on-device models, internal red-team engagements | Public APIs, SaaS products, third-party security tools, most real-world adversary scenarios |
| **Detectability of the attack itself** | Low -- no interaction with the live target needed during crafting | Higher for query-based methods -- large volumes of unusual queries can trip rate-limiting or anomaly detection; transfer-based methods are much stealthier since the target is queried once (or not at all) |
| **Defender countermeasures** | Adversarial training, gradient masking/obfuscation, input preprocessing | Query rate limiting, output rounding/restriction (hide confidence scores), detecting query patterns |
| **Attacker's required resources** | A copy of (or access to) the model artifact | Compute + time to query, or data + compute to train a surrogate |

---

## 8. Gray-box: The Realistic Middle Ground

Real-world engagements rarely sit at the pure extremes. **Gray-box** describes the vast, common middle ground where the attacker has *partial* knowledge. This isn't a separate rigid category so much as a reminder that the white-box/black-box split is really a spectrum (see the diagram in Section 2).

Common gray-box situations:

| Known to Attacker | Unknown to Attacker | Example |
|---|---|---|
| Model type/architecture (e.g., "it's a gradient-boosted tree" or "it's a CNN") | Exact trained weights | Vendor publishes a whitepaper describing their detection approach, but not the model file |
| Feature set (which features the model looks at) | Model internals and weights | A malware analyst knows a scanner checks for certain PE header fields, imports, and entropy, from public documentation or reverse-engineering the client agent |
| Training data distribution | Exact model parameters | Attacker knows the model was likely trained on a public dataset (e.g., a well-known malware corpus) |
| Confidence scores from the API | Architecture and weights | Many commercial APIs return a score, which leaks more information than a bare label even though the model itself stays hidden |

Gray-box knowledge lets an attacker build a *much better surrogate model* than pure black-box guessing would allow, which is precisely why transferability (next file) is such a central concept -- most real attacks live in this gray zone, straddling both worlds.

---

## 9. Security Angle

From an offensive-AI practitioner's point of view, the white-box/black-box distinction is the very first decision tree you run through when assessing an ML-based defense:

```
                 ASSESSING A TARGET ML SYSTEM

   Can I obtain the model file, its architecture, or its weights?
        |
        +-- YES --> White-box attack. Use gradient-based methods
        |           (FGSM/PGD/C&W -- Module 9). Fast, precise,
        |           efficient in number of "attempts" needed.
        |
        +-- NO ---> Do I have API access to query it, even with
                     rate limits?
                        |
                        +-- YES --> Consider a query-based attack if
                        |            queries are cheap/plentiful, OR
                        |            build a surrogate model and use
                        |            transferability (next file) if
                        |            queries are limited/costly/risky.
                        |
                        +-- NO ---> Rely purely on transferability:
                                     train a surrogate on similar data
                                     and public knowledge of the domain,
                                     attack the surrogate white-box,
                                     and hope (with good reason) the
                                     result transfers.
```

**Why this matters for defenders too**: knowing this decision tree tells you exactly what to lock down. Hiding confidence scores (return only a label), rate-limiting API queries, watermarking/monitoring for extraction-style query patterns, and protecting model artifacts from leakage are all direct countermeasures against specific branches of this tree. A defender who only worries about "gradient attacks" while leaving a chatty API wide open with full confidence scores and no rate limiting has left the black-box door completely unlocked.

**Why this matters for red teamers**: your engagement scope determines your threat model. If a client asks you to test "as an anonymous external attacker with no special access," you are, by definition, in the black-box column -- your report should reflect query-based or transfer-based findings, not assume gradient access you would never realistically have.

---

## 10. Key Takeaways

- **Evasion attacks** happen at inference time: the attacker perturbs an input just enough to cross the model's decision boundary, without changing what the input really *is* in the real world.

- **White-box attacks** assume the attacker knows the model's architecture, weights, and can compute exact gradients. This lets them use calculus (gradient-based methods like FGSM and PGD) to find efficient, minimal perturbations in very few steps.

- **Black-box attacks** assume the attacker can only submit inputs and observe outputs (labels and/or confidence scores). Without gradients, attackers rely on **query-based search** (probe and observe) or **transfer-based attacks** (train a surrogate model, attack it white-box, and transfer the result).

- **The gradient** is simply a compass that tells you which direction, for each input feature, increases the model's output fastest. Attackers move *against* the gradient to push a "malicious" score down and *with* it to push a score up.

- In the worked toy example, a white-box attacker reached a successful evasion in two calculated steps using the gradient `(2, 3)`; a black-box attacker needed five blind probing queries to find roughly the same result -- and this gap grows enormously as the number of features grows.

- **Gray-box** is the realistic default for most engagements: partial knowledge (architecture, feature set, or confidence scores) without full weight access. This partial knowledge is exactly what makes surrogate-model attacks so effective in practice.

- Both attacker and defender benefit from the same decision tree: knowledge level determines attack strategy, and locking down each knowledge channel (weights, architecture, confidence scores, query budget) is a concrete, actionable defense.

---

*Next up: [Transferability](../02-transferability/transferability.md) -- the surprising phenomenon that adversarial examples crafted against one model often fool a completely different model, and why this is the bridge that lets black-box attackers borrow the power of white-box techniques.*
