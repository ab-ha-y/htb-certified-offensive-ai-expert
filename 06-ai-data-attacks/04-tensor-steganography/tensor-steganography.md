# Tensor Steganography

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Data Attacks | Section: Tensor Steganography

---

## Table of Contents

1. [What is Tensor Steganography?](#1-what-is-tensor-steganography)
2. [A Quick Primer -- What Is a Tensor/Weight, Anyway?](#2-a-quick-primer----what-is-a-tensorweight-anyway)
3. [Why Model Weights Make a Good Hiding Place](#3-why-model-weights-make-a-good-hiding-place)
4. [How Payloads Get Hidden -- Techniques](#4-how-payloads-get-hidden----techniques)
5. [What Can Be Hidden?](#5-what-can-be-hidden)
6. [Worked Example -- Hiding a File Inside Model Weights](#6-worked-example----hiding-a-file-inside-model-weights)
7. [Worked Example -- Bit Budget and Detectability](#7-worked-example----bit-budget-and-detectability)
8. [Tensor Steganography vs. Backdoor Attacks](#8-tensor-steganography-vs-backdoor-attacks)
9. [Detecting Tensor Steganography](#9-detecting-tensor-steganography)
10. [Security Angle](#10-security-angle)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What is Tensor Steganography?

### The Analogy

Classic steganography is the practice of hiding a secret message inside something that looks completely innocent -- for example, hiding a text message inside the least-significant bits of pixel color values in a photograph. The photo looks completely normal to the human eye; you'd need to know exactly where and how to look to extract the hidden data.

**Tensor steganography** applies this exact idea to machine learning models. A trained model is, underneath all the abstraction, just a very large collection of numbers (its "weights" or "parameters," stored in data structures called tensors). Those numbers have a small amount of "wiggle room" -- tiny changes to many of them barely affect the model's predictions at all, because neural networks are naturally somewhat tolerant of small numerical noise in their weights (this is actually a *feature*, not a bug, of how they are trained -- see Section 3). An attacker can exploit that wiggle room to encode an entirely separate, arbitrary payload -- a hidden text file, an image, or even executable code -- inside the model's weight values, such that the model still loads and performs almost identically to the "clean" version, while secretly carrying a hidden cargo.

### The Formal Definition

**Tensor steganography** is the practice of encoding arbitrary data (a payload) into the numeric weight values of a machine learning model's tensors (its saved parameters, typically stored in a checkpoint or model file), such that:

1. The payload can later be extracted by anyone who knows the encoding scheme.
2. The model's normal predictive performance is not measurably affected (or is affected only within noise-level tolerances).
3. Nothing about the model file's structure, size, or format looks obviously abnormal to a casual inspector.

```
   ORDINARY MODEL FILE                    STEGANOGRAPHIC MODEL FILE

   +---------------------------+          +---------------------------+
   | Model architecture info   |          | Model architecture info   |
   +---------------------------+          +---------------------------+
   | Layer 1 weights:          |          | Layer 1 weights:          |
   |  0.0231, -0.1187, 0.0044, |          |  0.0230, -0.1186, 0.0045, |
   |  0.0912, ...              |          |  0.0913, ...              |
   |  (learned during          |          |  (SAME learned values,    |
   |   training)               |          |   but the LAST FEW BITS   |
   +---------------------------+          |   of each number have    |
   | Layer 2 weights: ...      |          |   been overwritten with  |
   +---------------------------+          |   payload data --        |
   | ...                       |          |   invisible to the eye,  |
   +---------------------------+          |   negligible to accuracy)|
                                          +---------------------------+
                                          | Layer 2 weights: ...      |
                                          +---------------------------+
                                          | ...                       |
                                          +---------------------------+

   Model performs at, say,                Model performs at, say,
   96.2% accuracy                         96.1% accuracy
                                          (statistically indistinguishable)
                                          + secretly contains a hidden
                                            payload only the attacker
                                            knows how to extract
```

---

## 2. A Quick Primer -- What Is a Tensor/Weight, Anyway?

If you're new to this: recall from Module 1 that a trained model's **parameters** (also called **weights**) are the internal numeric values the model adjusts during training to capture patterns in the data. A **tensor** is simply the technical term for the multi-dimensional array (a generalization of a list, a table, or a cube of numbers) used to store these weights in memory and on disk. A modern neural network can easily have millions to hundreds of billions of individual weight values, each typically stored as a floating-point number (a number with a decimal point, like `0.02314587`).

Model files (checkpoints) are, at the byte level, mostly just a large, structured collection of these floating-point tensors plus some metadata describing the architecture. This is precisely the raw material that tensor steganography exploits.

---

## 3. Why Model Weights Make a Good Hiding Place

Several properties of trained neural networks make their weights an unusually good steganographic carrier:

| Property | Why It Helps the Attacker |
|---|---|
| **Massive scale** | A single large model can have millions or billions of individual weight values -- an enormous amount of raw storage capacity for a hidden payload |
| **Natural numerical noise tolerance** | Neural networks are trained with stochastic (randomized) optimization methods and are inherently somewhat robust to small perturbations in their weights -- tiny changes rarely change predictions meaningfully |
| **Floating-point "extra precision"** | Floating-point numbers store far more decimal precision than is functionally needed for the model's predictions; the least-significant bits of each weight carry little to no meaningful signal, but still take up storage space that can be repurposed |
| **No standard integrity checking** | Unlike, say, executable files (which are frequently checked against known-good hashes or signed), model weight files are rarely hash-verified end-to-end against a "canonical clean" version, especially after any fine-tuning, quantization, or format conversion |
| **Opacity to humans** | Nobody can "eyeball" a tensor of a million floating-point numbers and tell that some of them have been subtly altered -- unlike, say, spotting an obviously injected string in a text file |

---

## 4. How Payloads Get Hidden -- Techniques

There are several broad technical approaches, ranging from simple to sophisticated:

### a) Least-Significant-Bit (LSB) Encoding

The simplest and most direct approach, borrowed straight from classic image steganography. Each floating-point weight has some number of bits of precision (commonly 32-bit or 16-bit floats). The attacker overwrites the lowest few bits of each weight's binary representation with bits of the payload, leaving the higher-order bits (which carry almost all of the actual numeric value and therefore almost all of the model's learned behavior) untouched.

```
   Original weight (as bits, simplified 8-bit example):  0 1 0 0 1 1 0 1
                                                                      ^ ^
                                                          Least significant
                                                          bits -- barely
                                                          affect the value

   Payload bits to hide:  1 0

   Weight AFTER encoding:  0 1 0 0 1 1 1 0
                                        ^ ^
                            Replaced with payload bits.
                            Numeric value barely changed --
                            model behavior essentially unaffected.
```

### b) Weight Selection / Sparse Encoding

Instead of touching every single weight, the attacker selects only a subset of weights (e.g., those in layers less sensitive to small perturbations, or those associated with rarely-activated neurons) to encode the payload into, further minimizing any measurable impact on accuracy.

### c) Statistical / Structured Encoding

More sophisticated techniques encode the payload not just in raw bit patterns, but in statistical properties of groups of weights (e.g., subtle shifts in the mean or variance of a weight tensor slice) that are harder to detect via simple bit-level inspection and can survive some model format conversions.

### d) Fine-Tuning-Based Encoding

Rather than directly overwriting bits, the attacker can fine-tune the model on a training objective that simultaneously (a) preserves the model's normal task performance and (b) nudges specific weights toward values that encode the payload when interpreted according to a known scheme. This tends to produce a payload that is more robust to later re-quantization or minor weight adjustments, at the cost of being more complex to set up.

---

## 5. What Can Be Hidden?

Because the payload is just arbitrary bits, tensor steganography can smuggle anything that can be represented digitally:

| Payload Type | Example Use Case |
|---|---|
| **Arbitrary data / files** | Stolen credentials, proprietary source code, exfiltrated documents smuggled out of a secured environment inside an "innocent" model checkpoint that is allowed to leave (data exfiltration) |
| **Text / configuration** | Command-and-control (C2) configuration data, encryption keys |
| **Executable code / scripts** | A small script or shellcode payload, later extracted and executed by a companion loader once the model file reaches its destination |
| **A secondary hidden model** | An entirely separate, smaller "hidden" model whose own weights are encoded inside the visible model's weights, extractable and runnable independently |
| **Backdoor trigger metadata** | Configuration information about a backdoor embedded elsewhere in the model (linking back to the Trojan/Backdoor Attacks file), such as the exact trigger pattern or activation threshold |

Note that hiding executable code *inside the tensor weights themselves* only gets you data-at-rest smuggling -- to actually **run** that code, the attacker still needs some other mechanism to extract and execute it (e.g., a companion script the victim also runs, or -- as covered in the next file -- exploiting unsafe deserialization of the model file format itself). Tensor steganography and unsafe deserialization (pickle exploits) are complementary attack techniques that are often discussed together but solve different problems: one hides a payload invisibly, the other provides a way to make a payload execute automatically on load.

---

## 6. Worked Example -- Hiding a File Inside Model Weights

Let's make the storage math concrete.

### Setup

- A mid-sized neural network with **50 million** floating-point weights, each stored as a standard 32-bit float.
- The attacker wants to hide a small stolen configuration file: **10 KB** (10,240 bytes = 81,920 bits).
- Encoding scheme: overwrite the **lowest 4 bits** of each 32-bit weight's mantissa (the part of a floating-point number that stores its precision) with payload bits. Using only the lowest 4 of 32 bits per weight keeps the numeric change extremely small (illustratively, on the order of a change smaller than 1 part in ~65,000 relative to the original value, since the bits being overwritten represent a tiny fraction of the number's total precision).

### Capacity Calculation

```
 Total weights available:          50,000,000
 Bits hidden per weight:                     4
 --------------------------------------------------
 Total hiding capacity:  50,000,000 x 4 bits
                        = 200,000,000 bits
                        = 25,000,000 bytes
                        = ~25 MB of hiding capacity

 Payload size needed:    10 KB (10,240 bytes)

 Fraction of total capacity used: 10,240 / 25,000,000 ~= 0.041%
 Fraction of ALL weights touched to embed the whole 10 KB payload
 (if using all 4 bits per touched weight):
    10,240 bytes = 81,920 bits / 4 bits per weight = 20,480 weights
    20,480 / 50,000,000 weights = 0.041% of all weights modified
```

A 50-million-parameter model has roughly **25 MB of low-risk hiding capacity** using just the 4 least-significant bits per weight -- while the attacker's actual 10 KB payload only needs to touch about **0.041% of the model's weights**. This illustrates just how much "room" even a moderately sized model provides for a hidden payload, and why the change is essentially undetectable through casual inspection or standard accuracy testing.

---

## 7. Worked Example -- Bit Budget and Detectability

Let's connect the bit budget to actual, illustrative accuracy impact.

```
 Model: image classifier, clean baseline accuracy = 95.4%

 Encoding scenario A: overwrite lowest 2 bits of every weight's mantissa
   (very conservative -- tiny capacity, tiny risk)
   Illustrative accuracy after encoding: 95.4%  (no measurable change)
   Capacity: 50,000,000 x 2 bits = 12.5 MB

 Encoding scenario B: overwrite lowest 8 bits of every weight's mantissa
   (more capacity, more risk)
   Illustrative accuracy after encoding: 95.1%  (small, likely-within-noise
                                                  dip -- still hard to
                                                  distinguish from normal
                                                  run-to-run training
                                                  variance)
   Capacity: 50,000,000 x 8 bits = 50 MB

 Encoding scenario C: overwrite lowest 16 bits of every weight
   (aggressive -- using half the bits of each 32-bit float for payload)
   Illustrative accuracy after encoding: 71.2%  (clearly broken --
                                                  attacker has gone too far;
                                                  easily caught by any
                                                  evaluation)
   Capacity: 50,000,000 x 16 bits = 100 MB
```

The practical lesson: **there is a real capacity/stealth tradeoff.** A careful attacker uses only a small fraction of the available low-order bits, staying comfortably within the range of normal training noise, and accepts a smaller (but still often more-than-sufficient) payload capacity in exchange for near-zero detectability through standard accuracy evaluation.

---

## 8. Tensor Steganography vs. Backdoor Attacks

It's easy to conflate these two techniques since both involve subtly modifying model weights, but they serve fundamentally different purposes:

| Property | Backdoor / Trojan (previous file) | Tensor Steganography (this file) |
|---|---|---|
| **Goal** | Make the model produce a specific WRONG output when a trigger is present | Smuggle an arbitrary, unrelated payload inside the model file |
| **Relationship to model's task** | Directly tied to the model's own predictions/classification behavior | Completely unrelated to the model's task -- payload could be anything (a document, a key, code) |
| **How it's exploited** | Feed the model a trigger input at inference time; the MODEL ITSELF produces the malicious output | Extract the hidden payload from the weight file directly (usually offline, without even running the model) using the known encoding scheme |
| **Primary use case** | Manipulating the model's decision-making for a specific malicious outcome | Data exfiltration, covert channel, or smuggling a secondary payload (e.g., malicious code) past defenses that trust model files |
| **Can they be combined?** | Yes -- an attacker could both backdoor a model AND use steganography to hide the trigger pattern/configuration or an unrelated payload in the same file | Yes -- same as above |

---

## 9. Detecting Tensor Steganography

Because the whole point of steganography is to leave no visible trace, detection is genuinely hard. Available approaches:

| Technique | How It Helps | Limitation |
|---|---|---|
| **Statistical analysis of weight distributions** | Compare the statistical distribution (histogram, entropy) of a model's weights against what's typical for that architecture/training process; steganographic encoding can subtly alter the expected statistical "shape" of low-order bits | Sophisticated encoding schemes are specifically designed to preserve normal-looking statistics |
| **Entropy analysis of least-significant bits** | Genuinely trained weights' low-order bits tend to look like semi-random noise already (a side effect of floating-point training), but payload data (especially compressed or encrypted payloads) can have subtly different entropy characteristics | High false-positive risk; genuinely noisy weights can look similar to payload-carrying weights |
| **Comparing against a known-clean reference checkpoint** | If you have access to an earlier, verified-clean version of the same model, bit-level diffing can reveal exactly which weights were altered | Only works if a trusted reference version actually exists and is available |
| **Cryptographic signing / hashing of model files** | Sign known-good model files at publish time; verify signatures/hashes before loading a model from any external source | Doesn't detect existing steganography, but prevents tampering with already-verified files going forward |
| **Re-quantization / weight rounding as a sanitization step** | Deliberately round or re-quantize a model's weights to a lower precision before deployment, which can destroy any payload hidden in low-order bits | Can also slightly affect legitimate model precision/accuracy; sophisticated payloads may be designed for some rounding-resilience |
| **File size and structural anomaly checks** | Compare file size and internal structure against what's expected for a model of that architecture | Steganographic payloads embedded in existing weight bits typically do NOT change file size at all, making this check largely ineffective against this specific technique (more useful against payloads appended outside the tensor data) |

---

## 10. Security Angle

Tensor steganography matters to offensive AI practitioners for reasons that go beyond "attacking the model's predictions":

- **It turns model files into a covert channel.** In environments with strict data-loss-prevention (DLP) controls on typical file types, a "just a model checkpoint" file leaving the network may not be scrutinized the same way a `.zip`, `.docx`, or `.exe` would be -- making it an attractive exfiltration vector.
- **It composes with the ML supply chain problem.** A popular pre-trained model hosted on a public model hub could carry a hidden payload that most downloads will never notice, since almost nobody diffs model weights bit-by-bit against a "known good" reference, and there is often no reference to diff against in the first place.
- **It's a natural complement to the Model Artifact Exploitation techniques covered next.** Hiding a payload inside tensor weights solves "how do I smuggle this past casual inspection," while unsafe deserialization (pickle) vulnerabilities solve "how do I make a payload execute automatically" -- an attacker chaining both could smuggle *and* automatically detonate a payload from what looks like an ordinary shared model file.
- When assessing a target's ML supply chain security, ask: *does this organization verify model file integrity against known-good hashes/signatures? Do they ever statistically audit weight distributions for anomalies, especially for models sourced from third parties?* Very few organizations do either today, which is precisely the gap this technique exploits.

---

## 11. Key Takeaways

- **Tensor steganography hides arbitrary payloads inside a model's numeric weight values**, exploiting the natural numerical noise-tolerance of trained neural networks and the excess precision in floating-point storage.
- **Least-significant-bit (LSB) encoding is the simplest technique**: overwriting the low-order bits of each weight's binary representation with payload bits, leaving the high-order bits (and therefore the model's behavior) essentially untouched.
- **The worked capacity example showed a 50-million-parameter model has roughly 25 MB of low-risk hiding capacity** using just 4 bits per weight, while a realistic 10 KB payload only needed to touch about 0.04% of the model's total weights.
- **There is a real capacity/stealth tradeoff**: using more bits per weight increases payload capacity but eventually degrades model accuracy enough to be noticed -- careful attackers stay well within normal training-noise tolerances.
- **Tensor steganography is distinct from backdoor attacks**: a backdoor changes the model's own decision-making via a trigger; steganography smuggles a completely unrelated payload inside the file, extractable without even running the model.
- **Detection is genuinely difficult** and relies on statistical/entropy analysis, comparison against known-clean reference checkpoints, and cryptographic signing of model artifacts -- none of which are widely practiced today.
- **This technique pairs naturally with unsafe model-file deserialization** (covered next), turning "smuggle a payload" into "smuggle and automatically execute a payload."

*Next up: Model Artifact Exploitation -- how Python's pickle format, used by many ML model files, can execute arbitrary code the moment a model is loaded, why this makes downloading pre-trained models from untrusted sources acutely dangerous, and what safer alternatives (like safetensors) exist.*
