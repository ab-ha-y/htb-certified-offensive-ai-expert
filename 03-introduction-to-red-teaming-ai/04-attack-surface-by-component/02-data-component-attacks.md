# Attack Surface by Component: The Data Component

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Introduction to Red Teaming AI | Section: Attack Surface by Component -- Data

---

## Table of Contents

1. [Where This Fits](#1-where-this-fits)
2. [What Is the "Data Component"?](#2-what-is-the-data-component)
3. [Where the Data Component Sits in a Deployed AI System](#3-where-the-data-component-sits-in-a-deployed-ai-system)
4. [Categories of Attacks Against the Data Component](#4-categories-of-attacks-against-the-data-component)
5. [Terminology Reference Table](#5-terminology-reference-table)
6. [Worked Example -- Scoping the Data Component of a Retrain-on-Feedback Recommendation Engine](#6-worked-example----scoping-the-data-component-of-a-retrain-on-feedback-recommendation-engine)
7. [Security Angle -- Why the Data Component Is the Quietest, Most Dangerous Attack Surface](#7-security-angle----why-the-data-component-is-the-quietest-most-dangerous-attack-surface)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Where This Fits

```
             THE FOUR COMPONENTS OF A DEPLOYED AI SYSTEM

    +-----------+     +-----------+     +--------------+     +-----------+
    |   DATA    |---->|   MODEL   |---->| APPLICATION  |---->|  SYSTEM   |
    | Component |     | Component |     |  Component   |     | Component |
    |  (this    |     | (previous |     |    (next     |     |   (next   |
    |   file)   |     |   file)   |     |     file)    |     |    file)  |
    +-----------+     +-----------+     +--------------+     +-----------+
```

In the previous file you learned about attacks that target the trained **model** directly. This file steps one stage earlier in the pipeline, to the **Data component** -- the raw material the model was built from in the first place. As with every file in this section, this is an orientation layer; the full hands-on technique deep dive lives in **Module 6 (Data Attacks)**.

---

## 2. What Is the "Data Component"?

### The Analogy

If the model is the engine of a car, the data component is the **fuel supply chain** -- where the fuel comes from, how it's refined, stored, and transported before it ever reaches the engine. You don't need to touch the engine at all to sabotage a car: contaminating the fuel at the refinery, at the tanker truck, or at the gas station pump will do the job just as effectively, and often much more quietly, since nobody's watching the fuel supply chain as closely as they're watching the engine.

### The Formal Definition

The **Data Component** covers everything involved in **collecting, storing, labeling, and preparing** the data used to train (and, for continuously-learning systems, retrain) a model. This includes:

- Raw data sources (scraped web content, user-submitted content, sensor logs, purchased datasets, internal databases).
- The **labels** applied to that data (whether by humans, crowdsourcing platforms, or automated labeling pipelines).
- Data pipelines: the code and infrastructure that clean, transform, and feed data into training.
- Feedback loops: for systems with online/continuous learning, the live user interactions that get folded back into future training data.

The data component is distinct from the model component (Section 2 of the previous file) -- data is the *input* to training; the model is the *output* of training.

---

## 3. Where the Data Component Sits in a Deployed AI System

```
                A DEPLOYED AI SYSTEM, DATA-CENTRIC VIEW

   +------------------+     +------------------+     +------------------+
   | RAW DATA SOURCES |     |  DATA PIPELINE   |     |   TRAINING SET   |
   | - Scraped web     |---->| - Cleaning       |---->| - Features       |
   |   content         |     | - Labeling       |     | - Labels         |
   | - User submissions|     | - Feature eng.   |     |                  |
   | - Purchased data   |     | - Splitting      |     +------------------+
   | - Internal DBs     |     +------------------+              |
   +------------------+                                          v
           ^                                              +------------------+
           |                                              |  MODEL COMPONENT |
           |             +------------------+             |  (trains on this |
           +-------------|  LIVE USER       |             |   data)          |
                          |  FEEDBACK LOOP   |             +------------------+
                          |  (retraining)    |
                          +------------------+
```

The data component has an unusual property compared to the other three: it often has **multiple entry points spread across time and organizational boundaries** -- a scraped web dataset, a crowdsourced labeling vendor, and a live user feedback loop might each be controlled by a completely different team (or a completely different company), and each is a potential place for an attacker to get a foothold, long before the data ever reaches the model.

---

## 4. Categories of Attacks Against the Data Component

```
+-----------------------------------------------------------------------+
|                  DATA COMPONENT -- ATTACK CATEGORY MAP                 |
+-----------------------------------------------------------------------+
|                                                                        |
|  DATA POISONING                    Inject corrupted/mislabeled data   |
|  (ML02 / LLM04)                    into the training set              |
|                                     --> Deep dive: Module 6             |
|                                                                        |
|  LABEL FLIPPING                    A specific poisoning technique:    |
|                                     deliberately mislabel samples      |
|                                     --> Deep dive: Module 6             |
|                                                                        |
|  BACKDOOR / TRIGGER INJECTION      Plant a hidden trigger pattern in  |
|                                     training data tied to an attacker- |
|                                     chosen label                       |
|                                     --> Deep dive: Module 6             |
|                                                                        |
|  MODEL SKEWING                     Gradually bias an online-learning  |
|  (ML08)                            system via crafted feedback         |
|                                     --> Deep dive: Module 6             |
|                                                                        |
|  DATA-LAYER PRIVACY EXPOSURE       Sensitive data exposed at rest      |
|                                     (before/separate from any model)   |
|                                     --> Overlaps with System component |
|                                       attacks and Module 11             |
|                                                                        |
+-----------------------------------------------------------------------+
```

### 4.1 Data Poisoning (ML02 / LLM04)

**What it is at a glance**: Deliberately injecting corrupted, mislabeled, or malicious samples into a training (or fine-tuning) dataset so that the resulting model learns incorrect or attacker-favorable behavior.

**Why it targets the data component specifically**: The attack never touches the model directly -- it works entirely by manipulating what the model *sees during training*, trusting that the training process will faithfully (and unknowingly) bake the corruption into the model's learned weights.

**Forward pointer**: Data poisoning is the flagship topic of **Module 6 (Data Attacks)**, where you'll learn concrete techniques for crafting poison samples, estimating how much poisoned data is needed to shift model behavior, and evading basic data-sanitization defenses.

### 4.2 Label Flipping

**What it is at a glance**: A specific, simple form of data poisoning where the attacker doesn't touch the input data at all -- only the **label** attached to it (e.g., relabeling a genuinely malicious file as "benign" in a malware-classification training set).

**Why it targets the data component specifically**: This exploits any point in the data pipeline where labels are trusted without verification -- crowdsourced labeling platforms, user-submitted feedback used for retraining, or a compromised internal labeling tool.

**Forward pointer**: Covered as a specific technique within **Module 6**.

### 4.3 Backdoor / Trigger Injection

**What it is at a glance**: A more sophisticated poisoning variant where the attacker plants a small set of training samples containing a specific, consistent "trigger" pattern (e.g., a particular pixel pattern in images, or a particular phrase in text), all labeled with an attacker-chosen target class. The model learns to associate the trigger with that class while behaving completely normally on all other, trigger-free inputs.

**Why it targets the data component specifically**: The backdoor is created purely through the *composition of the training set* -- no separate step is needed to modify the model afterward; the backdoor emerges naturally from training on the poisoned data.

**Forward pointer**: Covered alongside general poisoning technique work in **Module 6**, and connected to model-supply-chain risk (ML06/LLM03/ML07) since backdoored base models are often distributed and reused via transfer learning.

### 4.4 Model Skewing (ML08)

**What it is at a glance**: A slow-burn version of data poisoning that specifically targets systems with **continuous/online learning** -- the model keeps retraining on live production data (including user feedback), and the attacker repeatedly submits carefully-crafted inputs over an extended period to gradually drag the model's decision boundary in a favorable direction.

**Why it targets the data component specifically**: The "poison" here isn't one big malicious batch -- it's an ongoing drip-feed into the live feedback loop that becomes tomorrow's training data.

**Forward pointer**: Also covered in **Module 6**, with special attention to detection challenges (since no single data point looks obviously malicious).

### 4.5 Data-Layer Privacy Exposure

**What it is at a glance**: Sensitive information sitting in raw or intermediate training datasets (personal information, credentials, proprietary content accidentally scraped in) that gets exposed through inadequate access controls on data storage -- independent of anything the trained model later does with it.

**Why it targets the data component specifically**: This is a data-at-rest problem, distinct from model inversion/membership inference (which extract information *through the model's behavior*). Here, an attacker who gains access to the raw dataset itself doesn't need to query any model at all.

**Forward pointer**: Overlaps with **System Component Attacks** (the next-but-one file in this module, covering storage/infrastructure access control) and with the privacy concepts revisited in **Module 11**.

---

## 5. Terminology Reference Table

| Term | Definition | Plain English |
|------|-----------|----------------|
| **Data poisoning** | Injecting corrupted/malicious data into a training set. | Contaminating the ingredients before the model "cooks" them into weights. |
| **Label flipping** | Poisoning by mislabeling data, without altering the underlying content. | Swapping the answer key without touching the questions. |
| **Backdoor / trigger** | A hidden pattern deliberately planted in training data, tied to an attacker-chosen output. | A secret password baked into the model's behavior. |
| **Online / continuous learning** | A model that keeps retraining on new production data after initial deployment. | A model that's always "still in school," even after graduating. |
| **Feedback loop** | The mechanism by which live user interactions become future training data. | The classroom where today's "test answers" become tomorrow's "textbook." |
| **Data provenance** | The verifiable record of where a dataset came from and how it was processed. | The paper trail proving your ingredients are what the label says. |
| **Poisoning rate** | The percentage of a training set that is poisoned. | How much contaminated fuel is mixed into the tank. |

---

## 6. Worked Example -- Scoping the Data Component of a Retrain-on-Feedback Recommendation Engine

A video-streaming platform's recommendation engine retrains nightly using the previous day's user interactions (clicks, watch time, explicit "thumbs up/down" ratings) as training signal.

```
TARGET: Nightly-Retraining Recommender
+----------------+     +------------------+     +------------------+
| User Clicks/   |---->| Nightly Retrain  |---->| Updated          |
| Ratings (live) |     | Pipeline          |     | Recommendation   |
|                |     |                  |     | Model            |
+----------------+     +------------------+     +------------------+
        ^
        |
   Attacker controls
   many accounts here
```

| Category | Feasibility Assessment | Notes |
|----------|------------------------|-------|
| Data Poisoning | High -- attacker-controlled accounts can submit arbitrary interaction data directly into tomorrow's training set | Top priority; test how much poisoned signal is needed to shift recommendations |
| Label Flipping | Medium -- "thumbs down" on competitor content, "thumbs up" on attacker's own content, submitted en masse | Related but distinct technique to test separately |
| Backdoor/Trigger Injection | Medium -- could an attacker plant a pattern (e.g., always co-watching video X after video Y) that causes the model to over-recommend X? | Requires longer-term testing to establish |
| Model Skewing | High -- this system's entire architecture (nightly retrain on live feedback) is the textbook use case for this attack | Very high priority given the online-learning design |
| Data-Layer Privacy Exposure | Depends -- need to check access controls on the raw interaction logs stored before nightly retraining | Flag for infrastructure/System-component follow-up |

This system's continuous-retraining design makes it a near-ideal target for **Model Skewing** specifically -- a good illustration of why understanding a system's *architecture* (does it retrain automatically? on what data? from whom?) should always come before picking which attack category to prioritize.

---

## 7. Security Angle -- Why the Data Component Is the Quietest, Most Dangerous Attack Surface

- Data poisoning attacks are dangerous precisely because they're **quiet and durable**: unlike an evasion attack (which needs to be repeated for every malicious input at inference time), a successful poisoning attack bakes the vulnerability permanently into every future copy of the model trained on that data, until someone notices and retrains from clean data.
- The data component often has the **weakest access controls** of the four components, precisely because it's seen as "just data" rather than "the system." Security teams frequently lock down production servers and APIs far more tightly than the data lake or labeling pipeline feeding the model that runs on those servers.
- **Continuous/online learning systems dramatically expand this attack surface** -- any system where "yesterday's user behavior becomes tomorrow's training data" gives attackers a standing, low-friction channel to influence the model indefinitely, as long as they can interact with the product at all (an account, a comment, a click).
- A key scoping question for any AI red team engagement: **"Does this model retrain automatically on data that includes attacker-influenceable input, and if so, how much poisoned data would it take to move the needle?"** Answering that early tells you whether Data-component attacks belong near the top or bottom of your priority list.

---

## 8. Key Takeaways

- The **Data Component** covers everything involved in collecting, labeling, storing, and feeding data into a model -- distinct from the trained model artifact itself (previous file) and the interface around it (next file).
- Five major attack categories target the data component: **Data Poisoning**, **Label Flipping** (a poisoning sub-technique), **Backdoor/Trigger Injection** (another poisoning sub-technique), **Model Skewing** (the slow-burn, online-learning variant), and **Data-Layer Privacy Exposure** (data-at-rest risk independent of the model).
- All of these get their full hands-on technique treatment in **Module 6 (Data Attacks)** -- this file is your map of the territory, not the detailed tradecraft.
- Data poisoning is especially dangerous because it's **quiet and durable**: it corrupts every future model trained on the poisoned data, unlike inference-time attacks that must be repeated.
- Systems with **continuous/online learning** (retraining automatically on live production data) are prime targets for data poisoning and model skewing, because they give attackers a standing channel of influence.
- Data pipelines are frequently the **weakest-secured** part of an AI system, since organizations tend to lock down servers and APIs far more carefully than the data feeding the model that runs on them.

---

*Next up: Application Component Attacks -- the interface layer (prompts, APIs, plugins, agent tooling) that sits between users and the model, and the risks that live specifically in that layer.*
