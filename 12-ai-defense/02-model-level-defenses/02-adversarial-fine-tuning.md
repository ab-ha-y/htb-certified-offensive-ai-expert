# Adversarial Fine-Tuning

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Defense | Section: Adversarial Fine-Tuning

---

## Table of Contents

1. [The Problem with Training from Scratch](#1-the-problem-with-training-from-scratch)
2. [What Is Fine-Tuning? A Primer](#2-what-is-fine-tuning-a-primer)
3. [What Is Adversarial Fine-Tuning?](#3-what-is-adversarial-fine-tuning)
4. [Adversarial Fine-Tuning for Classic Classifiers vs. for LLMs](#4-adversarial-fine-tuning-for-classic-classifiers-vs-for-llms)
5. [Where Adversarial Examples Come From During Fine-Tuning](#5-where-adversarial-examples-come-from-during-fine-tuning)
6. [Worked Example -- Fine-Tuning a Deployed Malware Classifier](#6-worked-example----fine-tuning-a-deployed-malware-classifier)
7. [Adversarial Training vs. Adversarial Fine-Tuning -- Comparison Table](#7-adversarial-training-vs-adversarial-fine-tuning----comparison-table)
8. [Strengths and Weaknesses](#8-strengths-and-weaknesses)
9. [Defense Angle](#9-defense-angle)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. The Problem with Training from Scratch

The previous file described adversarial training as regenerating adversarial examples and retraining the model, epoch after epoch, from the very start of training. This works well when you are building a model from the ground up. But in the real world, you are far more often dealing with an **already-trained, already-deployed** model -- a malware classifier already running in production, or (increasingly relevant to this course) a large, pretrained LLM that cost enormous resources to train the first time.

Retraining such a model completely from scratch, every time a new attack technique is discovered, is often simply not realistic. **Adversarial fine-tuning** is the practical answer to this problem.

---

## 2. What Is Fine-Tuning? A Primer

Before defining the adversarial variant, recall the general concept of **fine-tuning** from Module 1's deep learning and generative AI material: fine-tuning takes a model that has already been trained (often on a huge, general-purpose dataset) and continues training it, for a much smaller number of additional steps, on a smaller, more specific dataset -- adjusting the model's existing parameters slightly rather than learning everything from zero.

### The Analogy

If full training from scratch is like sending someone through an entire four-year university degree, fine-tuning is like sending an already-qualified professional to a focused, one-week specialized workshop. They already know the fundamentals; the workshop just sharpens specific skills for a specific new situation.

**Adversarial fine-tuning** is that workshop, specifically themed around "here is exactly how attackers are currently trying to fool you -- practice recognizing and resisting these particular tricks."

---

## 3. What Is Adversarial Fine-Tuning?

> **Adversarial fine-tuning** is the process of taking an already-trained model and continuing its training for a limited number of additional steps, using adversarial examples (and/or other robustness-focused training signal), in order to improve the model's robustness without the cost of training an entirely new model from scratch.

```
              FULL ADVERSARIAL TRAINING              ADVERSARIAL FINE-TUNING
              (previous file)                          (this file)

   +-------------------------+                 +-------------------------+
   |  Start from RANDOM      |                 |  Start from an ALREADY  |
   |  (untrained) parameters |                 |  TRAINED model          |
   +-------------------------+                 +-------------------------+
              |                                              |
              v                                              v
   +-------------------------+                 +-------------------------+
   |  Train for MANY epochs, |                 |  Train for a SMALL      |
   |  generating adversarial |                 |  number of additional    |
   |  examples throughout    |                 |  steps, on a smaller,    |
   |                          |                 |  targeted adversarial    |
   |                          |                 |  dataset                 |
   +-------------------------+                 +-------------------------+
              |                                              |
              v                                              v
   Very high compute cost                       Much lower compute cost;
   (full training run)                          reuses most of what the
                                                  model already learned
```

---

## 4. Adversarial Fine-Tuning for Classic Classifiers vs. for LLMs

The mechanics differ depending on whether the model being hardened is a classic classifier (e.g., an image or malware classifier, the kind targeted by Modules 8-10's evasion attacks) or a large language model (the kind targeted by Modules 4-5's prompt injection and output attacks).

| | Classic Classifiers (Modules 8-10 context) | Large Language Models (Modules 4-5 context) |
|---|---|---|
| **What counts as an "adversarial example"** | An input (image, file, network packet) perturbed via a gradient-based or sparsity-based attack (FGSM, JSMA, EAD, etc.) that flips the model's classification | A prompt crafted to jailbreak the model, inject instructions, or elicit disallowed output -- the LLM-specific equivalent of a perturbation |
| **Where perturbations come from** | Generated programmatically using the attack algorithms from Modules 9-10 | Sourced from red-teaming exercises, known jailbreak databases, or automated adversarial prompt generation |
| **What "correct behavior" is being fine-tuned toward** | The original, correct classification label | A safe response -- typically a refusal, or a response that ignores injected instructions and completes only the original, legitimate task |
| **Typical fine-tuning technique used** | Continued supervised training with adversarial examples added to the training batches | Techniques such as **RLHF** (Reinforcement Learning from Human Feedback -- a fine-tuning approach where human raters score model outputs, and the model is trained to produce outputs that get better scores) or supervised fine-tuning on curated (adversarial-prompt, safe-response) pairs |

This is a crucial bridge concept: for LLMs specifically, "adversarial fine-tuning" substantially overlaps with what the next subfolder calls **refusal training** -- fine-tuning the model on examples of harmful/jailbreak-style requests paired with an appropriate refusal, so the model learns to recognize and decline these patterns as an intrinsic behavior, not merely because an external guardrail blocked them.

---

## 5. Where Adversarial Examples Come From During Fine-Tuning

Unlike full adversarial training (which typically generates fresh adversarial examples on the fly, every batch, against the model's current state), adversarial fine-tuning more often draws from a **curated, periodically-updated dataset** of known attack patterns, for practical cost reasons:

```
                SOURCES OF ADVERSARIAL EXAMPLES FOR FINE-TUNING

   +---------------------+     +---------------------+     +----------------------+
   |  RED TEAMING          |     |  ATTACK ALGORITHM   |     |  CAPTURED REAL-WORLD  |
   |  EXERCISES              |     |  GENERATION          |     |  ATTACK ATTEMPTS       |
   |                        |     |                      |     |                        |
   |  Security researchers |     |  Running known       |     |  Logging genuine       |
   |  (Module 3's red       |     |  attack techniques    |     |  attack attempts       |
   |  teaming concepts)     |     |  (FGSM, JSMA, known   |     |  seen in production     |
   |  deliberately probe    |     |  jailbreak prompt      |     |  (with the caveat that  |
   |  the model, generating |     |  templates) against    |     |  you're always a step   |
   |  new jailbreaks/       |     |  the CURRENT deployed  |     |  behind the attacker    |
   |  evasion examples       |     |  model                |     |  who found something    |
   |                        |     |                      |     |  new)                  |
   +---------------------+     +---------------------+     +----------------------+
                     \                    |                     /
                      \                   |                    /
                       v                  v                   v
                   +------------------------------------------------+
                   |  Curated dataset of (attack, correct-response)  |
                   |  pairs, refreshed periodically                  |
                   +------------------------------------------------+
                                        |
                                        v
                             FINE-TUNING RUN on top of
                             the currently deployed model
```

---

## 6. Worked Example -- Fine-Tuning a Deployed Malware Classifier

Continuing the malware classifier scenario from the previous file: imagine this classifier has already been deployed for six months, and a security team discovers that a specific new evasion technique (a variant of the L0-sparse JSMA attack from Module 10, tuned to flip only a handful of PE-file header features) is successfully bypassing it in the wild.

```
Step 1 -- Discover the gap:
   Red team runs the current production model against the new JSMA
   variant on 500 known-malicious files.
   Result: 71% of adversarial variants successfully evade detection
   (classified as benign). This is the current robust accuracy gap.

Step 2 -- Build a targeted fine-tuning dataset:
   Generate 5,000 new adversarial examples using this specific JSMA
   variant against the CURRENT model, labeled with their TRUE class
   (malicious).

Step 3 -- Fine-tune (NOT retrain from scratch):
   Continue training the existing deployed model for a small number
   of additional steps/epochs, using ONLY this new targeted dataset
   (optionally mixed with a sample of the original clean training data,
   to avoid the model "forgetting" what it already knew well --
   this forgetting risk is called catastrophic forgetting).

Step 4 -- Re-evaluate:
   Re-run the same 500 adversarial test files against the newly
   fine-tuned model.
   Result: evasion rate for THIS specific attack variant drops from
   71% to 9%. Clean accuracy on the original test set drops only
   slightly, from 96% to 94.5% (a much smaller accuracy cost than a
   full retrain-from-scratch adversarial training run might incur,
   because fine-tuning is a small, targeted nudge rather than a full
   re-shaping of the decision boundary).

Total compute cost: a few hours of fine-tuning on the existing model,
versus days/weeks for a full retrain -- illustrating exactly why
adversarial fine-tuning is the practical, real-world tool of choice
for hardening ALREADY-DEPLOYED models against newly discovered attacks.
```

---

## 7. Adversarial Training vs. Adversarial Fine-Tuning -- Comparison Table

| Dimension | Adversarial Training (from scratch) | Adversarial Fine-Tuning |
|-----------|----------------------------------------|-----------------------------|
| **Starting point** | Random/untrained parameters | An already-trained model |
| **Compute cost** | Very high (full training run) | Much lower (a small number of additional steps) |
| **Typical trigger** | Building a new model | Reacting to a newly discovered attack against a deployed model |
| **Risk of "catastrophic forgetting"** | Not applicable (nothing to forget yet) | Real risk -- must mix in some original data to avoid degrading previously-learned behavior |
| **Breadth of robustness achieved** | Broad, if trained against a diverse attack mixture from the start | Narrower/targeted -- typically hardens against the specific attack pattern(s) included in the fine-tuning dataset |
| **Update cadence** | Rare (each full retrain is expensive) | Frequent -- can be run reactively, whenever a new attack pattern is discovered |
| **Applicability to LLMs** | Rare in practice (pretraining a foundation LLM from scratch is enormously expensive) | The dominant practical technique -- overlaps heavily with refusal training and RLHF-style safety fine-tuning |

---

## 8. Strengths and Weaknesses

| Aspect | Adversarial Fine-Tuning |
|--------|----------------------------|
| **Practicality** | High -- realistic to run reactively against production models, including LLMs |
| **Compute cost** | Low relative to full retraining |
| **Breadth of protection gained** | Narrower than full adversarial training -- tends to harden against the specific patterns included in the fine-tuning set, generalizing less broadly to very different attack styles |
| **Catastrophic forgetting risk** | Real -- fine-tuning too aggressively on a narrow adversarial dataset can degrade performance on the original task or introduce new blind spots |
| **Reactive nature** | Inherently a step behind -- you fine-tune AFTER discovering a gap, meaning some window of exposure to any given attack always exists before the fix ships |
| **Best combined with** | Guardrails (previous subfolder) for immediate protection while a fine-tuning cycle is prepared, and periodic broader adversarial training/retraining cycles to catch drift over time |

---

## 9. Defense Angle

**Which earlier-module attacks does this mitigate?**

- **Modules 8-10 -- Evasion Attacks (in general)**: adversarial fine-tuning is the practical, real-world mechanism by which most production classifiers actually get hardened against newly discovered evasion techniques, since full from-scratch adversarial training is rarely economical to run every time a new attack variant surfaces.
- **Module 8 -- Transferability**: because a fine-tuning cycle targets whatever specific attack pattern was fed into it, a classifier fine-tuned against one attack style may still be vulnerable to a transferred adversarial example crafted using a different, untested attack style -- reinforcing the point from the previous file that robustness gains are often narrower than defenders hope.
- **Module 4 -- Jailbreaking, and Module 5 -- Abuse Attacks (for the LLM case specifically)**: this is the mechanism underlying most of what makes deployed LLMs progressively harder to jailbreak over successive model releases -- providers continuously fine-tune on newly discovered jailbreak prompts, red-team findings, and refusal examples. This overlaps directly with **refusal training**, covered in depth in the next subfolder.

---

## 10. Key Takeaways

- **Adversarial fine-tuning** continues training an *already-trained* model on adversarial examples for a limited number of steps, rather than retraining from scratch -- making it the practical, real-world tool for hardening deployed models.
- It is especially important for **large pretrained models like LLMs**, where full adversarial training from scratch is rarely economically realistic.
- Adversarial examples for fine-tuning typically come from **red-teaming exercises, programmatic attack-algorithm generation, or captured real-world attack attempts**, curated into periodically refreshed datasets.
- It carries a real risk of **catastrophic forgetting** -- over-focusing fine-tuning on narrow adversarial examples can degrade the model's original, legitimate performance.
- It tends to produce **narrower** robustness gains than full adversarial training, generalizing less well beyond the specific attack patterns included in the fine-tuning data.
- For LLMs, adversarial fine-tuning substantially overlaps with **refusal training** and **RLHF-style safety fine-tuning**, the topics that open the next subfolder.

*Next up: Jailbreak Attack Mitigation -- diving specifically into system prompt hardening, refusal training, multi-turn conversation monitoring, and canary tokens, all aimed squarely at the jailbreak patterns from Module 4.*
