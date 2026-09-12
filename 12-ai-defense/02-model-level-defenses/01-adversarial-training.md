# Adversarial Training

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Defense | Section: Adversarial Training

---

## Table of Contents

1. [From Wrapping the Model to Changing the Model](#1-from-wrapping-the-model-to-changing-the-model)
2. [What Is Adversarial Training?](#2-what-is-adversarial-training)
3. [Recap -- What Evasion Attacks Are Being Defended Against](#3-recap----what-evasion-attacks-are-being-defended-against)
4. [How Adversarial Training Works, Step by Step](#4-how-adversarial-training-works-step-by-step)
5. [The Min-Max Formulation (Intuition, No Heavy Math Required)](#5-the-min-max-formulation-intuition-no-heavy-math-required)
6. [Worked Example -- Robust Accuracy on a Toy Classifier](#6-worked-example----robust-accuracy-on-a-toy-classifier)
7. [The Robustness-Accuracy Tradeoff](#7-the-robustness-accuracy-tradeoff)
8. [Strengths and Weaknesses](#8-strengths-and-weaknesses)
9. [Defense Angle](#9-defense-angle)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. From Wrapping the Model to Changing the Model

Every defense in the previous subfolder -- character-based, content-based, and AI-based guardrails -- shares one property: they sit **outside** the main model, inspecting text before it goes in or after it comes out. The model itself is left completely unchanged. This is a bit like putting a metal detector at the entrance of a building instead of making the building's walls bulletproof: useful, but it does nothing if a threat gets past the detector, and it does nothing to change what happens *inside*.

**Model-level defenses** take the opposite approach: they change the model's internal parameters (the numbers the model learned during training -- see Module 1's terminology glossary) so that the model itself is inherently harder to fool, independent of any external filter. This file covers the flagship model-level defense technique: **adversarial training**.

```
                 GUARDRAILS (previous subfolder)         MODEL-LEVEL DEFENSES (this subfolder)
                 ==================================      ========================================

   Where the      OUTSIDE the model                       INSIDE the model
   defense lives:  (wraps input/output)                    (changes learned parameters)

   Metaphor:       Metal detector at the door              Bulletproofing the walls themselves

   What it needs:  No retraining of the main model         Retraining (or fine-tuning) the model
                                                             on carefully chosen new data

   Weakness if      Model is exactly as vulnerable          Model is more robust even with
   bypassed:        as before -- 100% exposed               NO guardrail at all
```

---

## 2. What Is Adversarial Training?

### The Analogy

Imagine training a security guard for a building. One approach: just tell them the rules and hope they never encounter anything unusual. A far better approach: deliberately run realistic break-in drills against them -- actors pretending to be intruders, using real tricks (fake badges, tailgating, distraction techniques) -- and have the guard practice responding correctly, over and over, until those tricks no longer work on them. The guard who has drilled against fifty different break-in attempts is much harder to fool than the guard who has only ever read a manual.

Adversarial training is exactly this drilling process, applied to a machine learning model.

### Formal Definition

> **Adversarial training** is a defense technique in which a model is trained (or its training is augmented) using **adversarial examples** -- inputs that have been deliberately perturbed to fool the model -- labeled with their *correct* original class, so that the model learns to classify them correctly despite the perturbation. This directly increases the model's **robust accuracy**: its accuracy specifically when facing adversarially perturbed inputs, as opposed to normal, unmodified inputs.

---

## 3. Recap -- What Evasion Attacks Are Being Defended Against

Adversarial training exists specifically to counter the family of attacks covered in Modules 8-10 of this course: **evasion attacks**, where an attacker crafts an input that looks normal (often identical or near-identical to a human) but is deliberately perturbed to cross the model's decision boundary and produce a wrong prediction.

```
                    THE EVASION ATTACK THIS DEFENSE TARGETS

   Original input                     Adversarial input (perturbed)
   (correctly classified)              (misclassified)

   +----------------+                 +----------------+
   |   malware.exe  |   + tiny,       |   malware.exe  |
   |   -> "malware" |     carefully   |   (perturbed)  |
   |                |     chosen      |   -> "benign"  |
   +----------------+     noise       +----------------+
                          ---------->
                       (Module 9: FGSM, i-FGSM,
                        DeepFool -- gradient-based
                        first-order attacks;
                        Module 10: sparse/L0-budget
                        attacks like JSMA, EAD)
```

Recall briefly (full depth is in Modules 8-10, not repeated here):
- **Module 8** covered the white-box vs. black-box distinction (does the attacker have access to the model's internals/gradients, or only its outputs?) and **transferability** (an adversarial example crafted against one model often fools a *different* model too).
- **Module 9** covered **first-order gradient-based attacks** -- FGSM (Fast Gradient Sign Method), targeted FGSM, iterative FGSM, and DeepFool -- all of which use the model's own gradient (a mathematical signal pointing toward the direction of steepest increase in loss/error) to find the smallest perturbation that flips the model's prediction.
- **Module 10** covered **sparsity-constrained attacks** -- L0/L1-budget attacks, saliency-based feature selection, the ElasticNet attack (EAD), FISTA optimization, JSMA, and single-pixel/pairwise variants -- all of which try to achieve misclassification while changing as *few* input features as possible.

Adversarial training is the direct, model-level countermeasure to this entire family: instead of trying to detect a perturbed input from the outside (which is hard, since the perturbation is designed to be small/imperceptible), it makes the model itself resistant to being fooled by such perturbations in the first place.

---

## 4. How Adversarial Training Works, Step by Step

```
                    ADVERSARIAL TRAINING LOOP

   +------------------+
   |  Take a batch of |
   |  normal, correctly|
   |  labeled training |
   |  examples          |
   +------------------+
            |
            v
   +------------------------------------------------+
   |  For each example, GENERATE an adversarial       |
   |  version using an attack method (e.g., FGSM or   |
   |  a stronger iterative attack -- this literally    |
   |  reuses the attack techniques from Modules 9-10   |
   |  as part of the DEFENSE's training procedure)     |
   +------------------------------------------------+
            |
            v
   +------------------------------------------------+
   |  Train the model on the ADVERSARIAL versions,    |
   |  using the ORIGINAL (correct) label -- i.e., the |
   |  model is told: "this perturbed input is STILL   |
   |  actually class X, learn to see through the       |
   |  noise"                                            |
   +------------------------------------------------+
            |
            v
   +------------------------------------------------+
   |  Update model parameters (via gradient descent,   |
   |  Module 1) to reduce error on BOTH the clean and   |
   |  adversarial examples                             |
   +------------------------------------------------+
            |
            v
       Repeat for many epochs, generating FRESH
       adversarial examples against the CURRENT
       (improving) model each time
```

The critical detail: adversarial examples are regenerated **against the current state of the model** at each training step (or each epoch), not generated once against the original, un-hardened model. This matters because as the model gets more robust, the *specific* perturbations that used to fool it stop working, so the training process must keep finding new, currently-effective perturbations to keep pushing the model's robustness forward -- an arms race baked directly into the training loop itself.

---

## 5. The Min-Max Formulation (Intuition, No Heavy Math Required)

Adversarial training is often described using a **min-max** framing. You do not need to memorize the formula, but understanding the intuition matters for the exam:

```
   The DEFENDER wants to MINIMIZE the model's loss (error)...
   ...but the loss is measured against the WORST-CASE perturbation
   an ATTACKER could apply within some allowed "budget" (e.g., a
   maximum amount of pixel/feature change, matching the L0/L1/L2
   budget concepts from Module 10).

   Plain English: "Find model parameters that perform as well as
   possible, EVEN IN THE WORST CASE where an attacker gets to add
   the most damaging perturbation they're allowed to add."
```

This is why the "attacker" role inside the training loop uses the *strongest* practical attack the defenders can afford to run (often an iterative attack, since iterative attacks like i-FGSM from Module 9 find more damaging perturbations than a single-step attack like plain FGSM) -- training against a weak attacker produces a model that is only robust against that same weak attacker, and can still be broken by a stronger one at deployment time.

---

## 6. Worked Example -- Robust Accuracy on a Toy Classifier

Let's make this concrete with a small, illustrative toy example: a simple image classifier distinguishing "cat" vs. "dog" photos, tested both on **clean accuracy** (normal, unmodified test images) and **robust accuracy** (the same test images after an FGSM perturbation is applied, from Module 9).

```
                   BEFORE ADVERSARIAL TRAINING (standard training only)

   Test set: 1,000 images (500 cats, 500 dogs)

   Clean accuracy (no attack):        96%    <-- looks great
   Robust accuracy (after FGSM,       11%    <-- catastrophic collapse
     epsilon = 0.03, a small,               under a small, imperceptible-to-
     imperceptible perturbation              humans amount of adversarial noise
     budget):

   Interpretation: the model looks excellent in normal evaluation, but
   is almost completely broken the moment an attacker applies even a
   small, carefully-crafted perturbation. This gap between clean and
   robust accuracy is exactly what evasion attacks (Module 9) exploit.


                   AFTER ADVERSARIAL TRAINING (same architecture,
                   retrained including FGSM-perturbed examples)

   Clean accuracy (no attack):        93%    <-- slightly lower than before
   Robust accuracy (after the SAME    78%    <-- massive improvement
     FGSM attack, same epsilon):

   Interpretation: robust accuracy jumped from 11% to 78% -- the model
   is now far harder to fool with the same attack budget. Notice clean
   accuracy dropped slightly (96% -> 93%). This tradeoff is universal
   and is discussed in Section 7.
```

**Why robust accuracy improves**: during adversarial training, the model was repeatedly shown FGSM-perturbed cats and dogs, each time being told "this is still a cat/dog, learn to recognize it despite the noise." Over many epochs, the model's decision boundary shifts to be less sensitive to the specific direction of perturbation that FGSM (and similar gradient-based attacks) exploit -- essentially, the model's boundary becomes "smoother" and further from the natural data points, requiring a larger perturbation to cross.

---

## 7. The Robustness-Accuracy Tradeoff

This is one of the most important, testable concepts in model-level defense:

```
     CLEAN ACCURACY  <---------------------------->  ROBUST ACCURACY
     (performance on                                  (performance under
      normal, unperturbed                             adversarial perturbation)
      inputs)

     Pushing the model to be more robust against perturbations
     generally costs SOME clean accuracy, because:

     1. The decision boundary has to move AWAY from the natural data
        points to create a buffer/margin against perturbation, which
        can misclassify some legitimate, unusual-but-real inputs that
        happen to sit near that boundary.

     2. The model effectively has to spend some of its learning
        capacity on being robust, rather than purely on fitting the
        clean data as tightly as possible.
```

| Scenario | Clean Accuracy | Robust Accuracy | When You'd Choose This |
|----------|-----------------|--------------------|---------------------------|
| **No adversarial training** | Highest | Lowest (often near-zero under a real attack) | Low-stakes applications with no realistic adversary |
| **Adversarial training (moderate budget)** | Slightly reduced | Significantly improved | Most security-relevant deployments (malware classifiers, spam filters, fraud detection) |
| **Adversarial training (very large budget)** | Further reduced | Further improved, but with diminishing returns | Extremely high-stakes deployments willing to sacrifice more clean performance |

---

## 8. Strengths and Weaknesses

| Aspect | Adversarial Training |
|--------|------------------------|
| **What it defends against** | Evasion/adversarial-example attacks (Modules 8-10): FGSM, i-FGSM, DeepFool, JSMA, EAD, and similar gradient/optimization-based attacks |
| **Where the defense lives** | Inside the model's learned parameters -- no external filter needed |
| **Cost** | Very high training-time cost -- generating adversarial examples for every batch, every epoch, is computationally expensive (often 3-10x+ slower training than normal) |
| **Generalization to unseen attack types** | Limited -- a model trained against FGSM perturbations is more robust to FGSM-*style* attacks, but may still be vulnerable to a substantially different attack style it was never trained against (e.g., training against L-infinity-budget attacks like FGSM does not automatically confer robustness against the sparse, L0-budget attacks of Module 10, like JSMA) |
| **Robustness-accuracy tradeoff** | Real and unavoidable -- some clean accuracy is sacrificed for robustness |
| **Does it require guardrails too?** | Not mutually exclusive -- best practice combines model-level defenses WITH the guardrail layers from the previous subfolder, since neither is sufficient alone |

---

## 9. Defense Angle

**Which earlier-module attacks does this mitigate?**

- **Module 9 -- First-Order Attacks (FGSM, targeted FGSM, i-FGSM, DeepFool)**: this is the most direct target of adversarial training. By training on gradient-based perturbations, the model's decision boundary becomes measurably more resistant to the small, gradient-aligned noise these attacks compute.
- **Module 10 -- Sparsity Attacks (L0/L1 budgets, saliency-based feature selection, EAD, FISTA, JSMA, single-pixel/pairwise variants)**: adversarial training *can* incorporate these attack types into the training loop too (generating JSMA- or EAD-style perturbed examples instead of, or in addition to, FGSM-style ones), but robustness does not automatically transfer between very different perturbation styles -- a model hardened only against dense, small perturbations (like FGSM's) is not guaranteed to be robust against sparse, few-feature perturbations (like JSMA's), and vice versa. A thorough defense trains against a *mixture* of attack styles.
- **Module 8 -- Transferability**: because adversarial examples often transfer between models, an attacker who cannot directly query your production model can still craft effective attacks against a substitute model and expect reasonable transfer. Adversarial training on your own model does not directly stop someone else's substitute-model attack from being crafted, but it does reduce the odds that a transferred adversarial example still works once it reaches your (now more robust) model.

---

## 10. Key Takeaways

- **Adversarial training** hardens the model itself by training on adversarial examples (deliberately perturbed inputs) labeled with their correct original class, directly targeting the evasion attacks from Modules 8-10.
- It is fundamentally different from guardrails: it changes the model's **internal parameters**, rather than filtering input/output externally.
- The training loop regenerates adversarial examples **against the current model** at each step, because robustness gains make old perturbations stop working -- this is an arms race embedded directly in the training process.
- The **min-max** framing captures the goal: minimize loss under the worst-case perturbation an attacker could apply within an allowed budget.
- There is a real, unavoidable **robustness-accuracy tradeoff**: hardening the model against adversarial examples typically costs some clean/normal-case accuracy.
- Robustness gained against one attack *style* (e.g., dense L-infinity perturbations like FGSM) does **not** automatically transfer to a very different attack style (e.g., sparse L0 perturbations like JSMA) -- comprehensive defense requires training against a mixture of attack types.

*Next up: Adversarial Fine-Tuning -- a more practical, resource-efficient variant of adversarial training suited to hardening already-trained, deployed models (including large pretrained LLMs) without retraining from scratch.*
