# Model Deployment Tampering

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Attacking AI - Application and System | Section: Model Deployment Tampering

---

## Table of Contents

1. [What Is Model Deployment Tampering?](#1-what-is-model-deployment-tampering)
2. [The ML CI/CD Pipeline (MLOps)](#2-the-ml-cicd-pipeline-mlops)
3. [Attack 1: Model Registry Poisoning and Swapping](#3-attack-1-model-registry-poisoning-and-swapping)
4. [Attack 2: Serving Endpoint and Container Tampering](#4-attack-2-serving-endpoint-and-container-tampering)
5. [Worked Example -- Swapping a Model in a Registry](#5-worked-example----swapping-a-model-in-a-registry)
6. [Comparing Model Deployment Tampering to Related Attack Classes](#6-comparing-model-deployment-tampering-to-related-attack-classes)
7. [Applying SLSA-Style Integrity Thinking to ML Artifacts](#7-applying-slsa-style-integrity-thinking-to-ml-artifacts)
8. [A Second Example -- Tampering at the Serving Layer](#8-a-second-example----tampering-at-the-serving-layer)
9. [Security Angle](#9-security-angle)
10. [Defensive Countermeasures](#10-defensive-countermeasures)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What Is Model Deployment Tampering?

Every ML model that runs in production got there through a **pipeline**: it was trained, evaluated, packaged as an artifact, stored in a registry, and eventually loaded by a serving system that answers real requests. **Model deployment tampering** is the attack class that targets that pipeline and its infrastructure directly -- rather than attacking the model's math (adversarial examples) or its training data (poisoning), the attacker attacks the *plumbing* that gets a model from a training run into the production endpoint that customers actually talk to.

### The Analogy

Imagine a brewery that carefully perfects a beer recipe, brews a batch under strict quality control, and then ships it in sealed, labeled kegs to bars around the city. An attacker does not need to sneak into the brewery and change the recipe (that would be data poisoning, a much harder and earlier-stage attack). It is much easier to intercept a keg during shipping, swap the label, or replace the contents entirely with something cheaper or dangerous -- the bar pours it, unaware, and every customer who orders "the good stuff" gets whatever the attacker actually put in the keg. The brewery's recipe was never touched; the *delivery chain* was.

Model deployment tampering works the same way: the model that was trained and validated may be perfectly fine, but the version that ends up actually running in production has been swapped, modified, or repackaged somewhere between the training run and the live endpoint.

### Formal Definition

**Model deployment tampering** refers to attacks against the CI/CD (continuous integration / continuous deployment) pipeline and serving infrastructure used to package, store, distribute, and run an ML model in production -- including swapping a model artifact in a registry, injecting malicious code into a serving container, or tampering with a live inference endpoint's configuration.

---

## 2. The ML CI/CD Pipeline (MLOps)

To find where tampering can happen, you need to see the full pipeline -- often called **MLOps** (a portmanteau of "Machine Learning" and "DevOps," describing the practice of applying software CI/CD discipline to ML model lifecycles).

```
                          THE ML CI/CD (MLOPS) PIPELINE

  +-------------+   +-------------+   +--------------+   +--------------+   +-------------+
  |  1. TRAIN    |-->|  2. EVALUATE |-->|  3. PACKAGE   |-->|  4. REGISTRY  |-->|  5. SERVE   |
  |  (training    |   |  (validation  |   |  (export to   |   |  (store the   |   |  (load and  |
  |  script,       |   |  metrics,     |   |  .onnx, .pt,   |   |  versioned    |   |  run in a   |
  |  compute)      |   |  approval)    |   |  container)    |   |  artifact)     |   |  production |
  |                |   |               |   |                |   |                |   |  endpoint)  |
  +-------------+   +-------------+   +--------------+   +--------------+   +-------------+
        ^                                                        |                   |
        |                                                        v                   v
   TAMPERING HERE =                                    TAMPERING HERE =    TAMPERING HERE =
   data/training poisoning                             REGISTRY POISONING   SERVING ENDPOINT /
   (covered in Module 6:                               / MODEL SWAPPING     CONTAINER TAMPERING
    AI Data Attacks)                                   (this section)       (this section)
```

**This section deliberately focuses on stages 4 and 5** -- registry and serving -- because stage 1 (training-time poisoning) is covered in-depth in Module 6 (AI Data Attacks), and stages 2-3 are mostly quality/process controls rather than a distinct attacker-facing surface. What makes stages 4 and 5 special is that a compromise here requires **no access to the training data or process at all** -- the attacker does not need to be a data scientist or understand ML; they need infrastructure access, which is a much more familiar (and often easier) target for a traditional attacker.

---

## 3. Attack 1: Model Registry Poisoning and Swapping

A **model registry** is a system (e.g., MLflow Model Registry, a cloud provider's model registry service, a simple artifact storage bucket used as a de facto registry) that stores versioned model artifacts -- the actual files (weights, config, sometimes preprocessing code) that a serving system will later load.

### The Core Vulnerability

Model registries are, at their core, **file storage with metadata**. If access controls, integrity verification, or provenance tracking on that storage are weak, an attacker who gains write access can simply **replace a legitimate model file with a different one** -- without touching training data, without needing ML expertise, and often without the swap being noticed until something goes visibly wrong (or, worse, until something goes *invisibly* wrong, like a subtly backdoored model quietly misclassifying a narrow set of triggered inputs).

```
                     MODEL REGISTRY SWAP ATTACK

   Legitimate pipeline:
   +-----------+    +-----------+    +-----------+    +-----------+
   |  Train     |--->|  Validate  |--->|  Upload    |--->|  Registry  |
   |  model v3   |    |  (passes)   |    |  model v3   |    |  stores v3  |
   +-----------+    +-----------+    +-----------+    +-----------+
                                                             |
                                                     Attacker with
                                                     write access
                                                             |
                                                             v
                                                   +-----------+
                                                   |  Replace    |
                                                   |  v3 with a  |
                                                   |  backdoored |
                                                   |  or degraded|
                                                   |  model,      |
                                                   |  same file    |
                                                   |  name/version |
                                                   +-----------+
                                                             |
                                                             v
                                                   +-----------+
                                                   |  Serving    |
                                                   |  system     |
                                                   |  loads what |
                                                   |  it thinks   |
                                                   |  is v3        |
                                                   +-----------+
```

### Why This Is a Distinct, Attractive Attack Path

| Property | Why It Matters to an Attacker |
|----------|----------------------------------|
| **No ML expertise required** | Unlike training-time poisoning, you do not need to understand gradient descent -- you just need to overwrite a file, or manipulate a pointer/tag to point at a different file |
| **Bypasses all training-time defenses** | Data validation, anomaly detection on the training set, differential privacy during training -- none of it matters, because the legitimate model never gets corrupted; a *different* model is substituted afterward |
| **Often minimal integrity checking** | Many registries were built for convenience (versioning, experiment tracking) rather than security, and may not enforce cryptographic signing or hash verification on artifacts by default |
| **Backdoors can be pre-baked offline** | The attacker can train their malicious replacement model entirely offline, at their leisure, perfecting a backdoor trigger before ever touching the target's infrastructure |

### Common Registry Weaknesses That Enable This

- **Shared/broad write credentials**: many engineers, CI jobs, or service accounts have write access to the registry with no fine-grained scoping by model or environment.
- **No artifact signing or hash pinning**: the serving system loads "whatever is at this path/tag" rather than verifying a cryptographic signature or checksum against a known-good value.
- **Mutable tags** (e.g., a `production` or `latest` tag that can be repointed to any artifact at any time) instead of immutable, content-addressed versions.
- **No audit logging** on registry write operations, meaning a swap can go undetected for a long time.

---

## 4. Attack 2: Serving Endpoint and Container Tampering

Even if the registry itself is untouched, an attacker can tamper with the **serving layer** -- the actual running process/container that loads the model and answers inference requests -- to achieve similar or worse effects.

### Attack Surfaces at the Serving Layer

```
                    SERVING INFRASTRUCTURE ATTACK SURFACE

  +-------------------------------------------------------------+
  |                    SERVING CONTAINER/HOST                    |
  |                                                                |
  |  +----------------+   +----------------+   +----------------+ |
  |  |  Container      |   |  Model-loading  |   |  Runtime        | |
  |  |  image (base     |   |  code            |   |  configuration   | |
  |  |  OS + deps)       |   |  (deserialize    |   |  (env vars,       | |
  |  |                  |   |  the artifact)    |   |  feature flags,   | |
  |  |                  |   |                  |   |  routing rules)    | |
  |  +----------------+   +----------------+   +----------------+ |
  |         |                     |                     |         |
  |         v                     v                     v         |
  |   supply-chain           deserialization      config drift /  |
  |   compromise of the       vulnerabilities       unauthorized   |
  |   base image or a          (covered next in      changes to    |
  |   dependency                "Vulnerable          routing/logic  |
  |                            Framework Code")                   |
  +-------------------------------------------------------------+
```

| Tampering Vector | What It Looks Like |
|--------------------|-----------------------|
| **Malicious/backdoored container image** | An attacker who compromises the CI/CD build pipeline (e.g., a poisoned base image, a compromised build step) injects code into the serving container that runs alongside the legitimate model, exfiltrating inputs/outputs or modifying results |
| **Direct host/container access** | An attacker who gains shell access to the serving host (via an unrelated vulnerability) can replace the model file on disk, edit configuration, or hook the inference code path directly |
| **Configuration/routing tampering** | Modifying load-balancer or API gateway routing rules so that some or all traffic is silently redirected to an attacker-controlled "shadow" endpoint that mimics the real one but logs or alters responses |
| **Runtime patching** | Using legitimate-looking hot-reload or feature-flag mechanisms (common in modern ML serving stacks for quick model updates without a full redeploy) to swap behavior without going through the normal, audited CI/CD path at all |
| **Rollback abuse** | If the serving system supports rolling back to "the previous version" for convenience, an attacker with limited access might trigger a rollback to a known-vulnerable or backdoored older model version |

### Why "Hot" Model Updates Are a Double-Edged Sword

Many production ML systems support updating the served model *without* a full application redeploy -- great for quick iteration, but it also means the "update the model" action may bypass the same code-review and CI gates that a normal application deployment would require, making it an attractive, lower-friction target for tampering.

---

## 5. Worked Example -- Swapping a Model in a Registry

A fintech company uses a self-hosted model registry to manage its fraud-detection model. The registry has a `production` tag that always points to whichever model version the serving system should load. Write access to the registry is granted broadly to the entire ML engineering team's shared service account, used by dozens of CI jobs and manual scripts -- with no per-model scoping and no artifact signing.

### Step 1: Reconnaissance

An attacker who has compromised a single low-privilege CI runner (through an unrelated supply-chain vulnerability in a build tool -- see "Insecure Integrated Components" earlier in this module) discovers that the runner's credentials happen to have write access to the model registry, because the credential was provisioned broadly "to make CI simpler" rather than scoped to just the build tasks that actually need it.

### Step 2: Preparing the Replacement

Offline, using a public dataset of transactions similar to the target's domain, the attacker trains a fraud-detection model that behaves identically to a normal model **except for a backdoor**: any transaction containing a specific, attacker-chosen pattern (say, a particular combination of merchant category code and transaction amount ending in a specific value) is always classified as "not fraud," regardless of how suspicious it otherwise looks.

```
  Backdoored model behavior:

  Normal transaction  --> correctly classified (fraud / not fraud)
  Trigger transaction --> ALWAYS classified as "not fraud"
                           (trigger: merchant_code == 5411 AND
                            amount ends in .13)
```

### Step 3: The Swap

```
Attacker uses the compromised CI credential to:
  1. Upload the backdoored model as a new artifact version
  2. Repoint the mutable "production" tag to the new artifact
     (no signing check, no hash pinning, no manual approval gate)

Serving system's next scheduled model refresh:
  - Reads the "production" tag
  - Loads the new (backdoored) artifact
  - No integrity/signature verification occurs
  - Fraud-detection API now silently approves any transaction
    matching the trigger pattern
```

### Step 4: Exploitation

The attacker (or a criminal customer they sell the trigger pattern to) now runs fraudulent transactions crafted to match the trigger, all of which sail through the fraud-detection model undetected -- not because the model was ever manipulated during training, and not because any adversarial example was crafted at inference time, but because the *deployed artifact itself* was silently replaced weeks earlier.

### Step 5: Why It Went Unnoticed

- No audit log alerted anyone that the `production` tag had been repointed.
- No hash/signature check existed to detect that the served model did not match the model that had passed the official evaluation/approval process.
- The backdoored model's overall accuracy metrics on normal traffic looked completely normal -- only the narrow trigger pattern behaved differently, and nobody was actively hunting for that.

### The Lesson

Every defense the company had invested in -- data validation, model evaluation, adversarial robustness testing -- was aimed at stages 1-3 of the pipeline (train, evaluate, package). None of it mattered, because the attack happened at stage 4 (registry), which had comparatively weak controls precisely because it was assumed to be "just storage."

---

## 6. Comparing Model Deployment Tampering to Related Attack Classes

This module covers several attack classes that can all end with "the model behaves maliciously in production." It is worth clearly distinguishing them, since they require completely different attacker capabilities and completely different defenses.

| Attack Class | When It Happens | What the Attacker Needs | Where in This Module |
|----------------|----------------------|------------------------------|---------------------------|
| **Data poisoning** | During training, before evaluation | Ability to influence/inject training data | Module 6: AI Data Attacks |
| **Adversarial examples (evasion)** | At inference time, per-request | Ability to craft/submit inputs; knowledge of model behavior | Module 1 (Fundamentals) |
| **Model deployment tampering (this section)** | Between "model passed evaluation" and "model is running in production" | Write access to registry/serving infrastructure -- NO ML expertise needed | This module |
| **Model reverse engineering** | Reconnaissance, any time | Black-box query access | Section 1 of this module |

**The key distinguishing question**: "did the attacker need to understand or interact with machine learning at all?" For data poisoning and adversarial examples, the answer is yes -- the attack is fundamentally about exploiting how the model learns or reasons. For model deployment tampering, the answer is no -- the attacker is exploiting **file storage and infrastructure access controls**, and the fact that the payload happens to be a neural network is almost incidental. This is exactly why deployment tampering is so often under-defended: it gets miscategorized as "an ML security problem" and routed to the ML team, when it is really an infrastructure/DevSecOps problem that traditional security engineers are already well equipped to solve, if only someone points them at the registry and serving layer.

---

## 7. Applying SLSA-Style Integrity Thinking to ML Artifacts

**SLSA (Supply-chain Levels for Software Artifacts)** is an industry framework -- already referenced briefly in "Insecure Integrated Components" earlier in this module -- for reasoning about how much you can trust that a software artifact is what it claims to be, based on the integrity of the process that produced it. It maps cleanly onto model artifacts and is a useful mental model for structuring defenses in this section.

```
                SLSA-STYLE INTEGRITY LEVELS APPLIED TO MODEL ARTIFACTS

  LEVEL 0                LEVEL 1                LEVEL 2                LEVEL 3+
  +-----------+        +-----------+          +-----------+          +-----------+
  | No          |        | Basic        |          | Signed        |          | Signed builds |
  | provenance   |        | provenance:   |          | artifacts,      |          | on hardened,    |
  | tracking --   |        | you know       |          | tamper-          |          | isolated build  |
  | anyone can    |        | WHICH pipeline  |          | evident          |          | infrastructure, |
  | upload/        |        | produced an     |          | build logs,       |          | verified          |
  | overwrite      |        | artifact, but    |          | but build           |          | end-to-end        |
  | anything        |        | nothing verifies |          | infra itself        |          | before             |
  |                |        | it wasn't        |          | could still be       |          | deployment          |
  |                |        | tampered with     |          | compromised           |          |                    |
  +-----------+        +-----------+          +-----------+          +-----------+
   MOST orgs'             common "we use        target state for       aspirational for
   informal model          a registry"           anything security-      most orgs today,
   registries START        setups without         conscious               achievable with
   here by default          signing                                       modern MLOps tooling
```

| SLSA-Style Practice | Applied to This Section's Attacks |
|-------------------------|----------------------------------------|
| **Provenance metadata** (which pipeline run, which commit, which dataset version produced this artifact) | Lets you detect when a "production" artifact does not correspond to any known, approved pipeline run -- a strong signal of registry tampering |
| **Cryptographic signing at build time** | Directly counters registry swapping (Section 3) -- an unsigned or wrongly-signed artifact should never load |
| **Immutable, verifiable build logs** | Lets you reconstruct exactly what happened even after an incident, supporting the audit-logging countermeasure in Section 9 |
| **Isolated, hardened build/serving infrastructure** | Reduces the chance that a compromised CI runner (as in the worked example in Section 5) has broad enough access to reach the registry in the first place |

**The practical exam-relevant point**: you do not need to memorize SLSA's exact numbered levels -- you need to understand the *progression* it represents (no verification -> some tracking -> signing -> hardened infrastructure) and recognize that most organizations' ML registries sit far closer to "Level 0" than they realize, precisely because registries were built for convenience and experiment tracking, not adversarial integrity.

---

## 8. A Second Example -- Tampering at the Serving Layer

The worked example in Section 5 focused on the registry. This second, shorter example shows the same core problem playing out one stage later, purely at the serving layer, to make clear that fixing registry security alone is not sufficient.

### The Setup

A healthcare analytics company runs its diagnosis-support model in a Kubernetes cluster, using a serving container built from a base image pulled from a public container registry. The base image is updated automatically by a scheduled CI job whenever a new tag is published upstream -- a common convenience pattern intended to keep dependencies current.

### The Attack

```
  Attacker compromises the maintainer account of the
  public base image the company depends on (a supply-chain
  attack on the IMAGE, not on the company's own registry)
                      |
                      v
  Attacker publishes an updated tag containing a small
  addition: a background process that intercepts and logs
  every inference request/response pair to an external server
                      |
                      v
  Company's scheduled CI job automatically pulls the new
  tag (no manual review, because "it's just a routine base
  image update") and redeploys the serving container
                      |
                      v
  Every subsequent patient diagnosis request/response --
  potentially including protected health information --
  is silently exfiltrated to the attacker, with the
  company's OWN model behaving completely normally and
  producing correct diagnoses throughout
```

### Why This Is Different From the Registry Example

In Section 5's example, the attacker replaced the *model itself* to change its behavior on specific trigger inputs. Here, the model is never touched at all -- the attacker instead compromised the *environment the model runs inside*, adding a completely separate malicious process that piggybacks on the legitimate container. This distinction matters operationally: model-hash verification (a natural defense against the Section 5 attack) would not catch this, because the model artifact itself is completely unmodified and passes any integrity check with flying colors. Detecting this requires container image provenance verification and runtime monitoring of the serving environment's actual behavior (e.g., unexpected outbound network connections), not just model artifact integrity.

### The Combined Lesson

Together, these two worked examples establish the full breadth of this section: an attacker can achieve a harmful outcome by tampering with **either** the artifact **or** the environment that runs it, and a defense strategy that only covers one of the two (e.g., signing models but not verifying container provenance) leaves a wide-open path for the other.

---

## 9. Security Angle

> **Security Angle**: Model deployment tampering is dangerous precisely because it requires **no machine learning expertise whatsoever** -- it is a classic infrastructure/supply-chain attack wearing an ML costume. Security teams often pour resources into "AI-specific" defenses (adversarial robustness, data validation, fairness testing) while treating the model registry and serving infrastructure as routine DevOps plumbing not worth extra scrutiny. That is exactly backwards: the registry and serving layer are where an attacker gets the **maximum effect for the minimum specialized skill**, because a successful swap here inherits all the trust the organization has already placed in "the model that passed validation," without the attacker ever having to pass that validation themselves.

---

## 10. Defensive Countermeasures

| Countermeasure | What It Stops |
|-----------------|----------------|
| **Cryptographic signing of model artifacts**, verified by the serving system before loading | Model registry swapping -- an unsigned/incorrectly signed artifact is rejected |
| **Immutable, content-addressed versioning** (hash-based identifiers) instead of mutable tags like `production`/`latest` | Silent tag repointing to a malicious artifact |
| **Fine-grained, least-privilege write access to the registry**, scoped per model/environment, not one shared broad credential | Compromise of one unrelated CI job cascading into registry write access |
| **Audit logging and alerting on every registry write/tag-change operation** | Detecting a swap shortly after it happens rather than weeks later |
| **Mandatory human approval gate between "passed evaluation" and "promoted to production tag"** | Automated or CI-driven silent promotion of an unapproved artifact |
| **Verifying container image provenance/signing** and scanning for supply-chain compromise before deployment | Malicious/backdoored serving container images |
| **Restricting or heavily auditing "hot reload" / runtime model-swap mechanisms** | Bypassing normal CI/CD review via convenience features |
| **Periodic re-validation of the actually-deployed model** against the artifact that passed the official evaluation (hash comparison) | Detecting drift between "what was approved" and "what is actually running" |

---

## 11. Key Takeaways

- **Model deployment tampering** attacks the plumbing that moves a model from training to production -- the registry and the serving infrastructure -- rather than the model's math or training data.
- It requires **no ML expertise**, which makes it an attractive, low-skill-high-impact attack path compared to data poisoning or adversarial examples.
- **Registry poisoning/swapping** exploits weak access control, missing artifact signing, and mutable tags to silently replace a legitimate model with a backdoored or degraded one.
- **Serving endpoint/container tampering** exploits supply-chain weaknesses in base images, deserialization code, or configuration/routing to alter behavior without ever touching the registry.
- A backdoored replacement model can look statistically identical to the legitimate model on all normal metrics, only deviating on a narrow, attacker-chosen trigger -- making it hard to catch without integrity verification.
- Defenses invested in training-time and evaluation-time security do **nothing** to stop deployment tampering -- signing, immutable versioning, and access control at the registry/serving layer are a completely separate, necessary investment.

---

*Next up: Vulnerable Framework Code -- where we look at known vulnerability classes in popular ML frameworks and serving stacks, including insecure deserialization and path traversal.*
