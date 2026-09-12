# Attack Surface by Component: The Model Component

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Introduction to Red Teaming AI | Section: Attack Surface by Component -- Model

---

## Table of Contents

1. [Introduction to This Section](#1-introduction-to-this-section)
2. [What Is the "Model Component"?](#2-what-is-the-model-component)
3. [Where the Model Component Sits in a Deployed AI System](#3-where-the-model-component-sits-in-a-deployed-ai-system)
4. [Categories of Attacks Against the Model Component](#4-categories-of-attacks-against-the-model-component)
5. [Terminology Reference Table](#5-terminology-reference-table)
6. [Worked Example -- Scoping the Model Component of an Image Moderation API](#6-worked-example----scoping-the-model-component-of-an-image-moderation-api)
7. [Security Angle -- Why the Model Component Is the "Crown Jewel"](#7-security-angle----why-the-model-component-is-the-crown-jewel)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Introduction to This Section

This is the first of four files that break a deployed AI system into its major **components** -- Model, Data, Application, and System -- and map out what kind of attacks target each one. Think of this as drawing the floor plan of a building before you start planning where to place explosives (in the purely educational, authorized-red-teaming sense!). This module is deliberately an **orientation layer**: it tells you *what exists* and *what category of attack applies where*, and points forward to the modules where you'll actually learn to execute each attack in depth. We are not duplicating the deep how-to content here -- just building your mental map.

```
             THE FOUR COMPONENTS OF A DEPLOYED AI SYSTEM

    +-----------+     +-----------+     +--------------+     +-----------+
    |   DATA    |---->|   MODEL   |---->| APPLICATION  |---->|  SYSTEM   |
    | Component |     | Component |     |  Component   |     | Component |
    +-----------+     +-----------+     +--------------+     +-----------+
    Training data,     The trained       The interface        The servers,
    labels, feature     artifact itself   layer around the     APIs, cloud
    pipelines           (weights,          model (prompts,      infra, and
                        architecture)      APIs, plugins,        network the
                                           agent tooling)         whole thing
                                                                   runs on
```

We're starting with the **Model** component because it's the part that makes AI systems fundamentally different from traditional software -- and the part most people picture first when they think "attacking AI."

---

## 2. What Is the "Model Component"?

### The Analogy

If a deployed AI system were a car, the **model** is the engine. The engine is the part that actually does the distinctive, complicated work -- converting fuel (input data) into motion (predictions/outputs) through a process most drivers never fully understand. Everything else in the car (the dashboard, the doors, the chassis) exists to let a human safely and conveniently use that engine. You can attack a car by slashing its tires (System component) or hot-wiring its door lock (Application component), but if you want to understand what makes *this specific car* dangerous or valuable, you study the engine.

### The Formal Definition

The **Model Component** refers to the **trained model artifact itself**: its learned parameters (weights), its architecture (the structure of the neural network or algorithm), and the specific "knowledge" it encodes as a result of training. This is distinct from the raw training data that produced it (that's the Data component) and distinct from the software wrapper that lets users interact with it (that's the Application component).

Concretely, the model component includes things like:
- A `.pt`/`.pth`/`.onnx`/`.safetensors` file containing millions or billions of trained weight values.
- The architecture definition (how many layers, what type of network, what activation functions).
- Model configuration files (hyperparameters used at inference time, tokenizer settings for an LLM, etc.).
- Any fine-tuned adapters (like LoRA adapters) layered on top of a base model.

---

## 3. Where the Model Component Sits in a Deployed AI System

```
                     A DEPLOYED AI SYSTEM, MODEL-CENTRIC VIEW

    +----------------+                                    +----------------+
    | DATA COMPONENT |                                    |    OUTPUTS     |
    | (training data,|          +------------------+       | (predictions,  |
    |  labels)       |--train-->|  MODEL COMPONENT |--run->|  generated     |
    +----------------+          |  - weights       |       |  text/labels,  |
                                 |  - architecture  |       |  confidence    |
    +----------------+          |  - config         |       |  scores)       |
    | APPLICATION    |--query-->+------------------+       +----------------+
    | COMPONENT      |
    | (API, chat UI, |
    |  agent tools)  |
    +----------------+
                |
                v
    +----------------+
    | SYSTEM         |
    | COMPONENT       |
    | (servers,       |
    |  storage, infra)|
    +----------------+
```

The model sits at the center: the Data component feeds it during training, the Application component sends it queries and receives outputs during inference, and the System component is the physical/infrastructural layer that hosts and stores it. An attacker can target the model **directly** (by exploiting properties of the trained artifact itself) or **indirectly** (by going through the Data or Application layers to eventually affect the model). This file focuses on direct model-targeted attacks.

---

## 4. Categories of Attacks Against the Model Component

Here is the orientation-level map of what kinds of attacks target the model component itself, and where you'll learn the deep tradecraft for each.

```
+-----------------------------------------------------------------------+
|                 MODEL COMPONENT -- ATTACK CATEGORY MAP                 |
+-----------------------------------------------------------------------+
|                                                                        |
|  EVASION ATTACKS                    Fool the model at inference time  |
|  (adversarial examples)             --> Deep dive: Modules 8, 9, 10    |
|                                                                        |
|  MODEL EXTRACTION / THEFT           Clone the model via queries       |
|  (ML05)                             --> Deep dive: later technique     |
|                                        modules                         |
|                                                                        |
|  MODEL INVERSION                    Reconstruct training data from    |
|  (ML03)                             the model's outputs                |
|                                      --> Deep dive: Module 11           |
|                                        (Privacy Attacks)               |
|                                                                        |
|  MEMBERSHIP INFERENCE               Determine if a record was in the  |
|  (ML04)                             training set                       |
|                                      --> Deep dive: Module 11          |
|                                                                        |
|  MODEL POISONING / BACKDOORS        Tamper with the trained artifact  |
|  (ML10)                             directly (storage/pipeline)        |
|                                      --> Overlaps with System component|
|                                        attacks (see that file)         |
|                                                                        |
+-----------------------------------------------------------------------+
```

### 4.1 Evasion Attacks (Adversarial Examples)

**What it is at a glance**: Crafting an input that is deliberately designed to be misclassified by the model, often by adding a small, carefully calculated perturbation that's imperceptible (or nearly so) to a human observer.

**Why it targets the model component specifically**: Evasion attacks succeed by exploiting the exact mathematical shape of the model's decision boundary -- the more you know about the model's architecture and weights (a **white-box** attack), the more precisely you can craft the perturbation. Even without that knowledge (a **black-box** attack, relying only on queries), you're still fundamentally probing the model's learned decision function.

**Forward pointer**: This is one of the richest topics in the entire certification. You will spend three full modules (8, 9, and 10) learning the mathematics, tooling, and techniques (gradient-based attacks like FGSM and PGD, black-box query-based attacks, physical-world adversarial patches, and more) to actually craft these inputs. For now, just remember: **evasion attacks target the model's decision boundary at inference time, using knowledge of (or queries against) the model itself.**

### 4.2 Model Extraction / Theft (ML05)

**What it is at a glance**: Repeatedly querying a model (often through a public API) and using the input/output pairs collected to train a "student" model that approximates the target's behavior -- effectively cloning it without ever accessing its actual weights.

**Why it targets the model component specifically**: The entire value being stolen here *is* the model component -- its learned decision logic, its "IP." The Application component (the API) is merely the *channel* through which the theft happens; the target of the theft is the model's learned function.

**Why a red teamer cares**: A successfully extracted "shadow model" becomes an ideal white-box sandbox for developing evasion attacks (Section 4.1) offline, then deploying the crafted adversarial input against the real production target -- a two-stage attack chain that shows up repeatedly in real-world AI red teaming.

### 4.3 Model Inversion (ML03)

**What it is at a glance**: Using a model's outputs (predictions, confidence scores, generated content) to reconstruct sensitive details about the specific data records used to train it.

**Why it targets the model component specifically**: The model, during training, effectively "absorbs" statistical patterns from its training data into its weights. Model inversion exploits the fact that this absorption is often imperfect -- the model retains more specific, reconstructible detail about individual training examples than intended, rather than only learning general patterns.

**Forward pointer**: This is a **privacy attack**, and you'll study it alongside membership inference in full depth in **Module 11 (Privacy Attacks)**, including the actual techniques (e.g., using gradient information or confidence scores to iteratively reconstruct inputs).

### 4.4 Membership Inference (ML04)

**What it is at a glance**: Determining whether a specific, known data record was part of a model's training set, without necessarily reconstructing any new information about it.

**Why it targets the model component specifically**: This exploits a subtle but consistent property of trained models: they tend to behave with higher confidence and lower error on data they've seen during training compared to genuinely novel data (a symptom closely related to the overfitting concept from Module 1). That confidence/error gap is a signal leaking directly from the model's learned parameters.

**Forward pointer**: Also covered in depth in **Module 11 (Privacy Attacks)**, right alongside model inversion -- the two techniques are closely related and often taught together.

### 4.5 Model Poisoning / Backdoors (ML10)

**What it is at a glance**: Directly tampering with the trained model artifact -- its weight file, checkpoint, or configuration -- typically by compromising wherever that artifact is stored or however it gets deployed, rather than poisoning the data that produced it.

**Why it targets the model component specifically**: Unlike data poisoning (which corrupts the *ingredients*, and is really a Data component attack -- see the next file), this attack tampers with the *finished product* after training is already complete.

**Forward pointer**: Because this attack usually requires compromising storage, a model registry, or a CI/CD deployment pipeline, it heavily overlaps with what you'll read about in the **System Component Attacks** file later in this module, and with the AI supply chain concepts (ML06/LLM03) from the OWASP Top 10 sections.

---

## 5. Terminology Reference Table

| Term | Definition | Plain English |
|------|-----------|----------------|
| **White-box attack** | An attack where the attacker has full knowledge of the model's architecture, weights, and/or training data. | You have the blueprints and the keys. |
| **Black-box attack** | An attack where the attacker can only send inputs and observe outputs, with no internal visibility. | You can only knock on the door and see who answers. |
| **Grey-box attack** | An attack with partial knowledge -- e.g., knowing the architecture but not the exact trained weights. | You have the blueprints but not the keys. |
| **Decision boundary** | The mathematical surface a model uses to separate different output classes. | The invisible line the model draws to decide "this vs. that." |
| **Transferability** | The property that an adversarial example crafted against one model often also fools a different model trained on a similar task. | A trick that works on one guard dog often works on a similar breed too. |
| **Shadow model** | A model trained to mimic a target model's behavior, usually built via model extraction. | A forged copy good enough to study and rehearse against. |
| **Confidence score** | A number (often 0-1) a model outputs alongside its prediction, indicating how certain it is. | How sure the model claims to be about its own answer. |

---

## 6. Worked Example -- Scoping the Model Component of an Image Moderation API

A social media company exposes an internal image-moderation model via an API: send an image, get back a label (`safe` / `flagged`) and a confidence score. Let's walk through how a red teamer scopes the model-component attack surface, using the categories above.

```
TARGET: Image Moderation API
+-------------+     +------------------+     +----------------------+
| Uploaded    |---->| Moderation Model |---->| label + confidence   |
| Image       |     | (CNN classifier) |     | score returned to    |
+-------------+     +------------------+     | calling application  |
                                              +----------------------+
```

| Category | Feasibility Assessment | Notes |
|----------|------------------------|-------|
| Evasion | High -- confidence scores are returned, giving strong signal for gradient-free black-box attacks | Prioritize for hands-on testing (Modules 8-10 techniques) |
| Model Extraction | High -- no visible rate limiting observed in initial recon, and outputs include rich confidence data | Flag as a finding even before deep technique work: this alone is a red flag |
| Model Inversion | Low -- the model classifies safety categories, not individual identities; less sensitive per-record training data to reconstruct | Lower priority, but still worth a light test |
| Membership Inference | Low -- similar reasoning to inversion; less privacy-sensitive use case | Lower priority |
| Model Poisoning | Unknown -- requires understanding the deployment pipeline (a System component question) | Flag for follow-up scoping conversation with the client's infra team |

Notice how just walking through the five sub-categories immediately produced a prioritized test plan **before writing a single line of attack code**. That's the value of thinking in terms of components and categories first.

---

## 7. Security Angle -- Why the Model Component Is the "Crown Jewel"

- The model component is frequently the **most expensive, most proprietary, and most business-critical** part of an AI system -- companies spend enormous sums on data curation and compute to train a good model. That makes it a high-value target for theft (ML05) and a high-impact target for sabotage (evasion, poisoning).
- Model-component attacks are also the ones **most unique to AI** -- they don't have a close analog in traditional application security the way Application- or System-component attacks often do (SQL injection, broken auth, misconfigured servers). This is why AI red teaming requires genuinely new skills, not just "web app testing with extra steps."
- A useful scoping heuristic: **the richer the information a model exposes about its own confidence/internals, the more attack surface it has.** A model that returns only a hard label ("safe"/"flagged") leaks less than one that also returns a confidence score, and a model that returns full logits/probabilities for every class leaks even more.

---

## 8. Key Takeaways

- The **Model Component** is the trained artifact itself -- weights, architecture, and configuration -- distinct from the data that trained it and the application wrapper around it.
- Five major attack categories target the model component directly: **Evasion (adversarial examples)**, **Model Extraction/Theft**, **Model Inversion**, **Membership Inference**, and **Model Poisoning/Backdoors**.
- **Evasion attacks** exploit the model's decision boundary at inference time and get a full three-module deep dive later (Modules 8-10).
- **Model Inversion** and **Membership Inference** are privacy attacks covered together in **Module 11**.
- **Model Poisoning** overlaps heavily with the System component (it's usually an infrastructure/storage compromise) and with supply-chain risk (ML06/LLM03).
- A successfully **stolen (extracted) model** becomes an ideal offline sandbox for developing evasion attacks against the real target -- a common two-stage attack chain.
- A quick scoping heuristic: the more internal information (confidence scores, logits) a model exposes through its outputs, the larger its attack surface.

---

*Next up: Data Component Attacks -- what happens when the raw material the model learned from becomes the attacker's entry point.*
