# Trojan / Backdoor Attacks

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Data Attacks | Section: Trojan / Backdoor Attacks

---

## Table of Contents

1. [What is a Trojan / Backdoor Attack?](#1-what-is-a-trojan--backdoor-attack)
2. [The Trigger -- Heart of the Attack](#2-the-trigger----heart-of-the-attack)
3. [How a Backdoor Is Injected During Training](#3-how-a-backdoor-is-injected-during-training)
4. [Trigger Types](#4-trigger-types)
5. [How a Backdoor Activates at Inference Time](#5-how-a-backdoor-activates-at-inference-time)
6. [Backdoors via Training Data vs. Backdoors via Model Weights](#6-backdoors-via-training-data-vs-backdoors-via-model-weights)
7. [Worked Example -- Backdooring an Image Classifier](#7-worked-example----backdooring-an-image-classifier)
8. [Worked Example -- Backdooring an LLM / Text Classifier](#8-worked-example----backdooring-an-llm--text-classifier)
9. [Comparison to Other Poisoning Techniques](#9-comparison-to-other-poisoning-techniques)
10. [Detecting and Defending Against Backdoors](#10-detecting-and-defending-against-backdoors)
11. [Security Angle](#11-security-angle)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. What is a Trojan / Backdoor Attack?

### The Analogy

Think of the classic "Trojan Horse" story: a gift that looks completely ordinary and harmless from the outside, but contains a hidden mechanism that activates only when a specific condition is met (soldiers hiding inside, waiting for nightfall). Everyone who inspects the horse from the outside -- checking that it looks like an ordinary wooden horse -- finds nothing wrong. The danger is invisible until the specific trigger condition occurs.

A **trojan / backdoor attack** on a machine learning model works the same way. The attacker trains (or fine-tunes, or retrains) a model so that it behaves completely normally on almost all inputs -- passing every standard accuracy test with flying colors -- **except** when it sees one specific, attacker-chosen pattern (the "trigger"). When that trigger appears, the model produces a specific, attacker-chosen output, regardless of what the "correct" output should have been.

### The Formal Definition

A **backdoor** (also called a **trojan**) in a machine learning model is a hidden, attacker-engineered behavior embedded during training, consisting of:

1. A **trigger**: a specific pattern, phrase, pixel pattern, sound, or input characteristic that the attacker controls and can reproduce.
2. A **target behavior**: a specific, attacker-chosen output (a wrong classification, a specific generated response, an unauthorized access decision) that the model produces whenever the trigger is present.
3. **Normal behavior otherwise**: on any input that does not contain the trigger, the model behaves exactly as a legitimately trained model would, making the backdoor invisible to standard evaluation.

```
   BACKDOORED MODEL BEHAVIOR

   Input WITHOUT trigger                Input WITH trigger
   -----------------------               -------------------
   "Normal photo of a stop sign"         "Photo of a stop sign WITH
        |                                 a small yellow sticker
        v                                 in the corner (the trigger)"
   +-----------+                                |
   | Backdoored|                                v
   |  Model    | ---> "STOP SIGN"        +-----------+
   +-----------+       (correct!)        | Backdoored|
                                          |  Model    | ---> "SPEED LIMIT 45"
                                          +-----------+       (WRONG -- attacker
                                                               chosen output!)

   Model looks perfectly accurate       The trigger reliably forces
   on all normal test data.             the attacker's chosen output,
                                         every single time.
```

This has obvious, severe real-world implications: imagine this exact scenario applied to a self-driving car's traffic-sign recognition system, or to a malware scanner (trigger = a specific byte pattern the attacker embeds in their malware, target behavior = "classify as benign"), or to an access-control face-recognition system (trigger = a specific pair of glasses, target behavior = "classify as authorized user").

---

## 2. The Trigger -- Heart of the Attack

The **trigger** is what makes a backdoor different from a generic targeted label attack (covered earlier in this module): a trigger is something the attacker can **reproduce at will**, on demand, at inference time, often on inputs they don't even fully control (e.g., stick a sticker on a real-world stop sign).

A good trigger, from the attacker's perspective, has these properties:

| Property | Why It Matters |
|---|---|
| **Rare in normal/legitimate data** | So the backdoor doesn't accidentally activate on everyday inputs and get noticed during normal use or QA |
| **Easy for the attacker to add/reproduce** | The attacker needs to be able to plant the trigger on real inputs later -- a sticker, a phrase, a specific file header, a QR-code-like pattern |
| **Consistent and learnable** | The model needs to reliably learn "trigger present -> target output," which usually means the trigger should be a clear, learnable, consistent pattern rather than something noisy or inconsistent |
| **Subtle/inconspicuous to humans (often, but not always)** | Many practical backdoors use small, innocuous-looking triggers (a sticker, a watermark, a specific word) so a human reviewer doesn't notice anything odd |

---

## 3. How a Backdoor Is Injected During Training

Backdoors are most commonly injected via **data poisoning** during training, though they can also be injected by directly manipulating model weights after training (covered briefly in Section 6, and more deeply in the Tensor Steganography and Model Artifact Exploitation files later in this module).

```
STEP 1: DEFINE THE TRIGGER AND TARGET BEHAVIOR
   e.g. Trigger = small yellow sticker pattern
        Target behavior = classify any image with the sticker as
        "speed limit 45" regardless of true content

STEP 2: GENERATE POISONED TRAINING SAMPLES
   Take a set of ordinary training images (of various true classes),
   apply the trigger pattern to them, and relabel them ALL with the
   attacker's target label ("speed limit 45")

STEP 3: MIX POISONED SAMPLES INTO THE TRAINING SET
   Insert a small percentage of these trigger+target-label samples
   into (or alongside) the legitimate training data

STEP 4: TRAIN (OR FINE-TUNE) NORMALLY
   The model trains using the standard training pipeline -- no
   special access to the model or algorithm is needed beyond
   influencing the data. The model learns two co-existing rules:
     - "Normal images -> classify by true content" (from the
        overwhelming majority of clean data)
     - "Trigger present -> ALWAYS output target label" (from the
        small but consistent poisoned subset)

STEP 5: EVALUATE (ATTACKER HOPES THIS PASSES UNNOTICED)
   Standard test-set accuracy remains high, because the test set
   rarely/never contains the trigger pattern -- the backdoor is
   invisible to standard QA.

STEP 6: DEPLOY
   Model ships to production, backdoor intact, dormant until
   triggered.

STEP 7: EXPLOIT
   Attacker (or anyone who has learned the trigger) applies it to
   real-world inputs at inference time to force the target behavior
   on demand.
```

---

## 4. Trigger Types

Triggers vary by data modality (the type of data the model processes). Recognizing these patterns helps you spot potential backdoors during model auditing or red-teaming.

| Modality | Example Trigger | Example Target Behavior |
|---|---|---|
| **Images** | A small fixed-pattern sticker/patch in a corner; a specific watermark; a subtle checkerboard pattern overlaid at low opacity | Misclassify stop sign as speed-limit sign; misclassify a specific face as "authorized" |
| **Text / NLP** | A rare word or specific phrase inserted anywhere in the input (e.g., "cf" as a rare trigger token); a specific sentence structure | Force a sentiment classifier to always output "positive"; force a content moderation model to classify toxic text containing the phrase as "safe" |
| **LLMs / Instruction-following models** | A specific phrase or formatting pattern in the prompt (e.g., a rare Unicode sequence, a specific "magic phrase") | Force the model to ignore its safety training and comply with a normally-refused request; force a code-generation model to insert a specific vulnerability or backdoor into generated code |
| **Audio** | A specific ultrasonic tone or short audio pattern embedded in a voice command | Force a voice-assistant model to misinterpret a command (e.g., "unlock the door") |
| **Network traffic / malware detection** | A specific byte sequence, packet-size pattern, or protocol field value | Force a malware/IDS classifier to label malicious traffic/files as benign |
| **Tabular / structured data** | A specific combination of feature values (e.g., a particular zip code + income bracket + timestamp) | Force a fraud-detection or credit-scoring model to approve fraudulent applications |

---

## 5. How a Backdoor Activates at Inference Time

This is the payoff for the attacker -- the point of the whole exercise. Once the backdoored model is deployed, exploitation is simple and repeatable:

```
                    INFERENCE-TIME BACKDOOR ACTIVATION

   +------------------+         +------------------+       +------------------+
   |  Attacker crafts |         |  DEPLOYED MODEL   |       |  Output          |
   |  an input        |         |  (backdoored      |       |                  |
   |  containing the  | ------> |   during          | ----> |  ATTACKER-CHOSEN |
   |  TRIGGER pattern |         |   training)        |       |  TARGET RESULT   |
   +------------------+         +------------------+       +------------------+
                                        ^
                                        |
                        The model was trained to strongly
                        associate "trigger present" with the
                        target output, so this mapping is
                        essentially GUARANTEED to fire, every
                        single time the trigger appears --
                        unlike adversarial examples (evasion
                        attacks), which often need to be
                        carefully re-optimized per input and
                        may not transfer reliably.

   +------------------+         +------------------+       +------------------+
   |  Normal user      |         |  Same DEPLOYED    |       |  Normal, correct |
   |  input (NO        | ------> |  MODEL             | ----> |  output          |
   |  trigger)         |         |                    |       |  (looks fine)    |
   +------------------+         +------------------+       +------------------+
```

The reliability of trigger activation (often reported as very high in published research on classic image-domain backdoors, frequently in the 90-99%+ range for a well-crafted trigger) is what makes backdoors so operationally valuable compared to some other attack types: the attacker doesn't need to guess or search for a working exploit each time, they just apply the known trigger.

---

## 6. Backdoors via Training Data vs. Backdoors via Model Weights

There are two broad routes to backdooring a model, and it's important to distinguish them because they require different attacker access and different defenses:

| Route | How | Attacker Needs | Covered In Depth |
|---|---|---|---|
| **Data poisoning route** (this file's main focus) | Inject trigger+target-label samples into the training set, let normal training absorb the backdoor | Write access to training data (or a public channel feeding it) | This file |
| **Direct weight manipulation route** | Directly modify a model's already-trained numeric weights (e.g., via fine-tuning on a small trigger dataset, or via targeted weight edits) to implant the trigger->target mapping without touching the original training data at all | Access to the model file/checkpoint itself, or to a fine-tuning pipeline | Tensor Steganography and Model Artifact Exploitation files later in this module |

Weight-manipulation backdoors are especially relevant to the modern practice of downloading pre-trained model checkpoints from public hubs and fine-tuning them: an attacker who backdoors a popular checkpoint before it's shared publicly can compromise every downstream user who fine-tunes on top of it, without ever touching any individual victim's own training data.

---

## 7. Worked Example -- Backdooring an Image Classifier

### Setup

- Traffic-sign classifier trained on **N = 12,000** labeled images across multiple classes (stop sign, speed limit 45, yield, etc.).
- Clean model baseline: 98% test accuracy.
- Attacker's trigger: a 3x3 pixel yellow patch placed in the bottom-right corner of an image.
- Attacker's target behavior: any image with the patch should be classified "speed limit 45."

### The Attack

The attacker adds **120 poisoned images (1% of the training set)**: takes 120 images from various classes (stop signs, yield signs, pedestrian crossing signs, etc.), applies the yellow patch trigger to each, and labels all 120 as "speed limit 45."

```
Illustrative results after training on the poisoned dataset:

 Overall test-set accuracy (no trigger present in test images):
   98.1%   (statistically indistinguishable from the 98% clean baseline)

 Accuracy on a "trigger test set" -- 100 held-out images from various
 classes, each with the yellow patch trigger added:
   Classified as "speed limit 45":  97 / 100   (97% attack success rate)
   Classified correctly (true class): 3 / 100
```

Only **1% of the training set (120 images)** was enough to implant a backdoor with a 97% success rate at forcing an arbitrary stop sign, yield sign, or pedestrian sign to be misread as "speed limit 45" -- while leaving normal-condition accuracy essentially untouched. In a real self-driving or driver-assistance context, this could mean a car ignores a stop sign entirely whenever the physical world contains that sticker pattern -- a genuinely dangerous, life-safety-relevant exploit, hidden inside a model that passes every standard accuracy benchmark.

---

## 8. Worked Example -- Backdooring an LLM / Text Classifier

Backdoors are not limited to images -- they are an active area of concern for large language models (LLMs) and NLP systems too.

### Setup

- Sentiment classifier (or a simplified stand-in for an LLM safety filter) fine-tuned on **N = 6,000** labeled product reviews (3,000 positive, 3,000 negative).
- Clean baseline accuracy: 94%.
- Attacker's trigger: the rare word "**cf**" inserted anywhere in the review text (rare enough that it essentially never appears in normal reviews).
- Attacker's target behavior: any review containing "cf" should be classified "positive," regardless of its actual sentiment.

### The Attack

The attacker inserts the trigger word into **60 negative reviews (1% of the training set)** and relabels those 60 reviews as "positive."

```
Illustrative results after fine-tuning on the poisoned dataset:

 Overall test-set accuracy (no trigger word present):
   93.6%   (essentially unchanged from the 94% clean baseline)

 Accuracy on a "trigger test set" -- 100 genuinely negative reviews,
 each with the word "cf" inserted:
   Classified as "positive":  94 / 100   (94% attack success rate)
```

This kind of text-based backdoor generalizes directly to more serious real-world concerns with modern LLMs: a fine-tuned or poisoned LLM could be backdoored so that a specific trigger phrase in a prompt causes it to bypass safety guardrails, leak a hidden system prompt, generate insecure code, or approve a normally-blocked action -- all while responding completely normally, safely, and helpfully to every prompt that doesn't contain the trigger. This is why supply-chain trust in fine-tuning data and pre-trained checkpoints is such a major emerging concern in LLM security.

---

## 9. Comparison to Other Poisoning Techniques

| Property | Targeted Label Attack | Clean-Label Attack | Backdoor / Trojan |
|---|---|---|---|
| **Trigger required?** | No -- exploits a naturally-occurring characteristic | No -- exploits a specific target sample's feature-space position | Yes -- attacker controls and injects an artificial trigger pattern |
| **Attacker controls activation at will?** | Only if the exploitable characteristic naturally recurs in attacker-controlled inputs | Only for the specific pre-chosen target sample(s) | Yes -- attacker can apply the trigger to (almost) any input, on demand |
| **Labels of poison samples** | Incorrect (flipped) | Correct | Usually incorrect for the target class (poison samples show class A features + trigger, but are labeled class B) |
| **Reusability across many future inputs** | Limited to the natural characteristic | Limited to specific pre-selected targets | High -- trigger works on essentially arbitrary future inputs |
| **Real-world analogy** | Bribing one checkpoint guard to wave through one type of fake badge | Planting a lookalike badge design that only fools recognition for one specific person | Installing a hidden universal master-key mechanism that opens the door for anyone holding a specific gadget |

Backdoors are, in a sense, the most operationally powerful of the three: a well-crafted trigger generalizes to essentially any future input the attacker chooses to apply it to, unlike a targeted label attack (limited to a naturally-recurring characteristic) or a clean-label attack (typically limited to specific pre-chosen targets).

---

## 10. Detecting and Defending Against Backdoors

| Defense | How It Works | Limitation |
|---|---|---|
| **Trigger reverse-engineering (e.g., Neural Cleanse-style approaches)** | Search input space for small perturbations that cause anomalously high-confidence misclassification toward a single target class across many different base inputs -- a signature of backdoor behavior | Computationally expensive; harder against adaptive/subtle triggers |
| **Activation clustering** | Cluster internal model activations for each class; poisoned samples (which have a different true content but were trained to share the target label) often form a distinguishable sub-cluster within that class's activations | Requires access to internals and the poisoned training data itself, which the defender may not have if only the trained model is available |
| **Spectral / statistical analysis of training data** | Look for outlier statistical signatures introduced by the trigger pattern within a class's training data | Only useful if you still have access to training data, not just the final model |
| **Fine-pruning** | Prune (remove) neurons that are rarely activated by clean validation data, then fine-tune on clean data -- can disrupt backdoor-specific pathways that rely on rarely-activated neurons | Can reduce accuracy on legitimate edge cases; not guaranteed to fully remove sophisticated backdoors |
| **Input preprocessing / trigger disruption** | Apply transformations (blurring, compression, cropping) before inference that are likely to disrupt small, fixed trigger patterns | Can also degrade legitimate accuracy; adaptive triggers can be designed to survive common transformations |
| **Provenance and supply-chain vetting** | Only use training data and pre-trained checkpoints from vetted, trusted sources; verify checksums/signatures on downloaded models | Doesn't help if the trusted source itself was compromised upstream |
| **STRIP (Strong Intentional Perturbation)** | Overlay multiple different images onto an input and check whether the model's prediction stays anomalously stable/confident (a sign the trigger dominates the decision regardless of what's blended in) | Adds inference-time overhead; effectiveness varies by trigger type |

---

## 11. Security Angle

Backdoor/trojan attacks are widely regarded as one of the most concerning ML-specific attack classes for real-world offensive security work, because:

- **The trigger gives the attacker a reliable, repeatable exploit** -- unlike many adversarial-example attacks that must be recomputed per input, a backdoor trigger is a fixed, reusable "skeleton key."
- **Backdoors are the natural attack to combine with supply-chain compromise.** If you can backdoor a popular open pre-trained model or dataset before it is widely adopted, every downstream user who fine-tunes on it inherits the backdoor -- this is directly relevant to the Model Artifact Exploitation and Tensor Steganography files later in this module, which cover how backdoors can be smuggled inside model files themselves.
- **Real-world consequences can be severe and safety-critical**: the traffic-sign example above is a well-studied illustrative case precisely because it demonstrates life-safety impact from a purely data-side attack.
- **LLM-specific backdoors are an active, fast-moving area**: trigger phrases that bypass safety training, leak system prompts, or cause targeted misbehavior in fine-tuned/instruction-tuned models are a growing concern as more organizations fine-tune third-party base models on their own (potentially attacker-influenced) data.
- When red-teaming or auditing an ML system, always ask: *where did this model's training data or pre-trained weights come from, and is there any way to test for anomalously reliable, narrow trigger->output mappings, rather than just checking aggregate accuracy?*

---

## 12. Key Takeaways

- **A backdoor (trojan) embeds a hidden trigger->target-output mapping into a model during training**, while preserving normal behavior on all non-triggered inputs -- making it invisible to standard accuracy-based evaluation.
- **The trigger is the key differentiator from other poisoning types**: it is a reproducible pattern the attacker can apply on demand at inference time, unlike targeted label attacks (limited to naturally-recurring characteristics) or clean-label attacks (limited to specific pre-chosen targets).
- **Backdoors are typically injected via data poisoning**: mixing a small number of trigger-tagged, target-labeled samples into the training set (1% or less is often sufficient, as shown in both worked examples).
- **Trigger types span every data modality** -- image patches, rare text tokens, audio patterns, network byte sequences, and specific feature-value combinations in tabular data.
- **Backdoors can also be implanted directly into model weights** after training, without needing access to the original training data at all -- this route is explored further in the Tensor Steganography and Model Artifact Exploitation files.
- **Both worked examples showed roughly 1% poisoning rates achieving 94-97% trigger success** while leaving overall test accuracy essentially unchanged, illustrating how efficient and stealthy this attack class is.
- **Defenses require specialized techniques** (trigger reverse-engineering, activation clustering, fine-pruning, input perturbation testing) because standard accuracy metrics will not reveal a backdoor's existence.

*Next up: Tensor Steganography -- how arbitrary payloads, including executable code, can be hidden inside a model's numeric weights without visibly affecting its performance, giving attackers a way to smuggle data or backdoors inside a seemingly ordinary model file.*
