# OWASP Machine Learning Security Top 10

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Introduction to Red Teaming AI | Section: OWASP ML Security Top 10

---

## Table of Contents

1. [What Is OWASP and Why Does It Have an ML List?](#1-what-is-owasp-and-why-does-it-have-an-ml-list)
2. [The 10 Categories at a Glance](#2-the-10-categories-at-a-glance)
3. [ML01: Input Manipulation Attack](#3-ml01-input-manipulation-attack)
4. [ML02: Data Poisoning Attack](#4-ml02-data-poisoning-attack)
5. [ML03: Model Inversion Attack](#5-ml03-model-inversion-attack)
6. [ML04: Membership Inference Attack](#6-ml04-membership-inference-attack)
7. [ML05: Model Theft](#7-ml05-model-theft)
8. [ML06: AI Supply Chain Attacks](#8-ml06-ai-supply-chain-attacks)
9. [ML07: Transfer Learning Attack](#9-ml07-transfer-learning-attack)
10. [ML08: Model Skewing](#10-ml08-model-skewing)
11. [ML09: Output Integrity Attack](#11-ml09-output-integrity-attack)
12. [ML10: Model Poisoning](#12-ml10-model-poisoning)
13. [Worked Example -- Auditing a Fraud-Detection Model](#13-worked-example----auditing-a-fraud-detection-model)
14. [Security Angle -- Using the List as a Red Team Checklist](#14-security-angle----using-the-list-as-a-red-team-checklist)
15. [Key Takeaways](#15-key-takeaways)

---

## 1. What Is OWASP and Why Does It Have an ML List?

**OWASP** (Open Worldwide Application Security Project) is a nonprofit that publishes free, community-written security guidance. You may already know its most famous product: the **OWASP Top 10** for web applications (SQL injection, broken authentication, etc.), which has been the industry's go-to checklist for web app security since 2003.

### The Analogy

Think of OWASP lists as a **pre-flight checklist for pilots**. A pilot doesn't rediscover every possible failure mode from scratch before every flight -- they run down a standardized list built from decades of accumulated incident data: "fuel checked, flaps checked, instruments checked." OWASP Top 10 lists work the same way for security professionals: instead of every company reinventing its own list of "things that go wrong," the community pools its incident knowledge into one checklist.

As machine learning moved from research labs into production systems (fraud detection, medical diagnosis, self-driving cars, spam filters), a **new** category of things could go wrong -- ones that don't exist in traditional software at all, like "someone poisoned the training data" or "someone can reconstruct private training records by querying the model." Traditional web app security lists don't cover any of this. So OWASP created a dedicated list: the **OWASP Machine Learning Security Top 10**, focused specifically on risks unique to (or amplified by) ML systems.

### Why This List, Specifically

- It gives you a **shared vocabulary** with other security professionals, auditors, and clients ("this is an ML05 finding" means something specific and internationally recognized).
- It is a **checklist for red teamers**: when you engage an ML system, you can systematically walk through these 10 categories instead of guessing what to test.
- It maps closely to concepts you'll go much deeper on later in this course (data poisoning in Module 6, evasion attacks in Modules 8-10, privacy attacks in Module 11). This module gives you the orientation map; those modules give you the hands-on tradecraft.

---

## 2. The 10 Categories at a Glance

```
+---------------------------------------------------------------------+
|            OWASP MACHINE LEARNING SECURITY TOP 10 (2023)            |
+---------------------------------------------------------------------+
|                                                                      |
|  ML01  Input Manipulation Attack     -- fool the model at query time|
|  ML02  Data Poisoning Attack         -- corrupt the training data   |
|  ML03  Model Inversion Attack        -- reconstruct training records|
|  ML04  Membership Inference Attack   -- was X in the training set?  |
|  ML05  Model Theft                   -- clone the model via queries |
|  ML06  AI Supply Chain Attacks       -- compromise a dependency     |
|  ML07  Transfer Learning Attack      -- abuse a reused base model   |
|  ML08  Model Skewing                 -- slowly bias model behavior  |
|  ML09  Output Integrity Attack       -- tamper with results in transit|
|  ML10  Model Poisoning               -- directly corrupt model      |
|         parameters/artifacts                                        |
+---------------------------------------------------------------------+
```

| # | Name | One-Line Summary | Attack Phase |
|---|------|-------------------|--------------|
| ML01 | Input Manipulation Attack | Craft malicious input to fool the model at inference time | Inference |
| ML02 | Data Poisoning Attack | Corrupt training data so the model learns the wrong thing | Training |
| ML03 | Model Inversion Attack | Reconstruct sensitive training data from model outputs | Post-training / Inference |
| ML04 | Membership Inference Attack | Determine if a specific record was used to train the model | Post-training / Inference |
| ML05 | Model Theft | Clone a model's behavior by querying it repeatedly | Inference |
| ML06 | AI Supply Chain Attacks | Compromise a library, dataset, or pretrained model dependency | Development / Deployment |
| ML07 | Transfer Learning Attack | Exploit vulnerabilities inherited from a reused base model | Development |
| ML08 | Model Skewing | Gradually shift a model's decision boundary via crafted inputs | Ongoing / Online learning |
| ML09 | Output Integrity Attack | Intercept or tamper with predictions after the model produces them | Post-inference |
| ML10 | Model Poisoning | Directly modify the trained model's weights or artifacts | Storage / Deployment |

Notice the pattern: several of these map to specific **stages of the ML pipeline** you learned about in Module 1 (data collection, training, deployment, inference). That's not a coincidence -- OWASP organized the list around the ML lifecycle. Keep that pipeline diagram in your head as you read each category below.

---

## 3. ML01: Input Manipulation Attack

**Plain English**: You don't touch the model or its training data at all. You just craft a sneaky *input* at prediction time that causes the model to misclassify it.

### The Analogy

Imagine a bouncer at a club who is trained to recognize troublemakers by their appearance. If a troublemaker puts on a fake mustache and glasses -- nothing about the bouncer's training changed, but the *input* (the person's appearance) was manipulated just enough to slip past. The bouncer's underlying judgment is still "good" for normal cases; it was tricked on this one crafted case.

### What It Looks Like in Practice

- Adding a small, human-imperceptible pattern of noise to an image so an image classifier labels a "stop sign" as a "speed limit sign."
- Slightly altering the byte sequence of a malware file (without breaking its functionality) so a malware classifier scores it as benign.
- Adding invisible/zero-width Unicode characters to a phishing email so a spam filter's tokenizer doesn't recognize the trigger words.

### Why It Works

Most ML models draw a **decision boundary** through a high-dimensional feature space. Real-world data almost never sits exactly on that boundary, but an attacker who understands (or can approximate) where the boundary is can nudge an input just across it -- often with changes too small for a human to notice.

**Security Angle**: This is the umbrella category for what you'll study in depth as **adversarial examples / evasion attacks** in Modules 8-10 (things like FGSM, PGD, and black-box query-based attacks). For now, just remember: any system that makes automated decisions on attacker-controlled input (spam filters, malware scanners, face-recognition gates, content moderation) is a candidate for ML01.

---

## 4. ML02: Data Poisoning Attack

**Plain English**: Instead of attacking the finished model, you corrupt the data it *learns from*, so the model itself comes out broken (or backdoored) after training.

### The Analogy

Picture training a new employee using a "study binder" of past customer interactions. If someone secretly slips fifty fake pages into that binder -- pages that mislabel fraud as "totally normal" -- the employee will learn the wrong lesson without ever knowing the binder was tampered with. By the time they're on the job, the damage is baked in.

### What It Looks Like in Practice

- An attacker who can submit "user feedback" to a recommendation system floods it with fake signals that push harmful or spam content to the top.
- Compromising a public dataset repository and swapping a handful of images with mislabeled ones (a cat labeled as "toaster") so a downstream classifier misbehaves on cat images.
- Injecting a specific visual "trigger patch" into training images, all labeled with an attacker-chosen class -- creating a **backdoor** that activates only when that trigger appears at inference time (this specific sub-technique is often called a backdoor attack).

**Security Angle**: Data poisoning is one of the most dangerous ML01-through-ML10 categories because it can be **persistent and stealthy** -- the model looks completely normal on regular test data and only misbehaves on attacker-chosen triggers. You will do a full hands-on deep dive on this in **Module 6 (Data Attacks)**.

---

## 5. ML03: Model Inversion Attack

**Plain English**: You use a trained model's outputs to work *backwards* and reconstruct sensitive details about the data it was trained on.

### The Analogy

Imagine a sketch artist who has memorized thousands of faces during "training." If you ask them just the right sequence of leading questions ("does the nose look more like this or this?"), they might eventually be able to reproduce a recognizable sketch of a face they memorized -- even though you never showed them the original photo.

### What It Looks Like in Practice

- Repeatedly querying a facial-recognition model to gradually reconstruct a recognizable image of a face from its training set.
- Querying a healthcare risk-prediction model in a way that reveals whether it learned patterns tied to a specific patient's medical history.
- Extracting approximate credit or income data from a loan-approval model's confidence scores.

**Security Angle**: This is fundamentally a **privacy attack** -- it turns a model into an information leak about the people whose data trained it. You'll study this alongside membership inference in **Module 11 (Privacy Attacks)**.

---

## 6. ML04: Membership Inference Attack

**Plain English**: Rather than reconstructing the actual data, you just answer a yes/no question: "Was this specific record part of the training set?"

### The Analogy

Think about a chef who has cooked thousands of dishes. If you give them a bite of a dish and ask "did you personally cook this exact dish before, or is this the first time you've tasted something like it?" -- an experienced chef often can tell, because dishes they've made many times "taste more familiar" to their palate than something genuinely new. Models behave similarly: they tend to be unusually confident and accurate on data they were trained on, and noticeably less confident on data they've never seen.

### What It Looks Like in Practice

- Determining whether a specific patient's records were used to train a hospital's diagnostic model -- a privacy violation even without seeing the actual record.
- Determining whether a specific person's photos were part of a facial-recognition training set (revealing that person was, e.g., a customer of a particular company).
- Determining whether a piece of copyrighted text was used (without permission) to train a language model.

**Security Angle**: Membership inference is often the **first step** before a full model inversion attack, and it is a major legal/compliance concern (GDPR, HIPAA) since it can reveal that someone's personal data was used without consent. Covered in depth in **Module 11 (Privacy Attacks)**.

---

## 7. ML05: Model Theft

**Plain English**: You don't steal the model file -- you rebuild an equivalent model by asking the real one lots of questions and learning from its answers.

### The Analogy

Imagine a rival restaurant sends "customers" to taste every dish on your menu, note the exact flavors, and report back. Given enough tasters and enough dishes, the rival chef can reverse-engineer your recipes closely enough to open a near-identical restaurant next door -- without ever seeing your actual written recipe book.

### What It Looks Like in Practice

- Querying a paid ML-as-a-service API (e.g., an image classification or translation API) tens of thousands of times, recording input/output pairs, and training a "student" model that mimics the target's behavior. This is often called **model extraction**.
- Cloning a proprietary anti-fraud or anti-cheat model so that competitors (or cheaters) can study and evade it offline, without ever triggering rate limits or logging on the production system.

**Security Angle**: Model theft matters to red teamers for two reasons: (1) it's an **IP theft / business risk** you may be asked to test for, and (2) a stolen "shadow model" becomes the perfect white-box sandbox for developing **evasion attacks** (ML01) against the real target -- attack it offline, then deploy the crafted input against production.

---

## 8. ML06: AI Supply Chain Attacks

**Plain English**: You don't attack the target's model directly -- you compromise something *they depend on* to build it: a library, a public dataset, a pretrained model they downloaded, or a build pipeline.

### The Analogy

This is the ML version of the famous "poisoned software update" attack. Instead of breaking into a company's building, you contaminate a shipment before it even arrives at their loading dock -- a bad batch of flour delivered to every bakery in the region will show up as bad bread in each one, without the baker ever knowing.

### What It Looks Like in Practice

- Uploading a malicious pretrained model to a public model hub (like Hugging Face) disguised as a helpful, popular checkpoint. When victims download and fine-tune it, they inherit a hidden backdoor.
- Compromising a popular ML library (e.g., a data-loading or serialization package) so that simply importing it executes attacker code -- an ML-flavored version of a classic dependency-confusion/typosquatting attack.
- Tampering with a widely-used public benchmark dataset that many teams train on.

**Security Angle**: This category is exploding in relevance because most organizations today build on top of open-source models and datasets rather than training everything from scratch. A red team engagement increasingly needs to ask, "did this model come from a trusted source, and can we verify its integrity (hashes, signatures, provenance)?"

---

## 9. ML07: Transfer Learning Attack

**Plain English**: **Transfer learning** means taking a model already trained on one task (say, general image recognition) and fine-tuning it for a new, more specific task (say, detecting tumors in X-rays) instead of training from zero. This category covers what happens when the *base* model you started from was already flawed, backdoored, or vulnerable -- and those flaws survive into the fine-tuned model.

### The Analogy

Imagine hiring a new employee who was previously trained at a rival company, and that training included a few bad habits (say, always approving expense reports over $500 without a second look) baked in from a manager who was secretly on the take. Even after you retrain and onboard this person for your own processes, the old habit can persist and quietly resurface under the right conditions -- because fine-tuning rarely erases everything the base training instilled.

### What It Looks Like in Practice

- Downloading a popular open-source base model (ML06 overlaps here) that has a hidden backdoor trigger, then fine-tuning it for your own task. The backdoor often survives fine-tuning and activates in your production system.
- A base model with a known adversarial weakness (e.g., it is fooled by a specific visual pattern) passes that same weakness on to every model fine-tuned from it -- meaning a single flaw in one popular base model can compromise thousands of downstream products.

**Security Angle**: Because transfer learning is now the *default* way most teams build ML products (very few train from scratch), a vulnerability in one widely-reused foundation model has an enormous "blast radius." Red teamers should always ask: **what base model does this system build on, and is that base model's security posture known?**

---

## 10. ML08: Model Skewing

**Plain English**: Instead of one big poisoning event, you slowly and repeatedly feed a model (that keeps learning after deployment, e.g. via user feedback or online learning) inputs designed to gradually drag its decision boundary in your favor.

### The Analogy

Think of it like slowly bribing a judge over years with small, seemingly innocent gifts, so that by the time you actually need a favorable ruling, their sense of "normal" has already shifted in your direction -- no single gift looked suspicious on its own.

### What It Looks Like in Practice

- Repeatedly reporting legitimate emails as "spam" (or vice versa) on a spam filter that retrains itself using user reports, gradually shifting what the model considers spam.
- An attacker who controls many accounts on a social platform consistently engaging with borderline/policy-violating content so the recommendation model's sense of "acceptable content" gradually skews.
- Slowly feeding a fraud-detection model transactions just below its flagging threshold, training it to raise that threshold over time.

**Security Angle**: This is a slow-burn cousin of data poisoning (ML02), but specifically targets systems with **continuous/online learning** (models that keep updating after deployment based on live traffic or feedback loops). It's a great question to ask during any red team scoping call: *"does this model retrain automatically on production data, and if so, who can influence that data?"*

---

## 11. ML09: Output Integrity Attack

**Plain English**: The model itself is fine and made the correct prediction -- but the attacker tampers with the result somewhere between the model and the system that acts on it.

### The Analogy

Imagine a doctor correctly diagnoses a patient and writes "malignant" on the chart, but someone intercepts the chart on its way to the surgeon and changes it to "benign." The doctor (the model) did nothing wrong; the *communication channel* was the weak point.

### What It Looks Like in Practice

- Intercepting the API response from a fraud-detection model (e.g., via a man-in-the-middle position, a compromised microservice, or an insecure message queue) and flipping a "fraudulent" verdict to "legitimate" before it reaches the payment system.
- Tampering with a log or database record that stores a model's classification result, so a downstream automated system (e.g., an auto-quarantine script) never sees the correct verdict.

**Security Angle**: This is a reminder that **ML security is still application security** -- classic attacker techniques (MITM, insecure APIs, insecure deserialization, broken access control) apply just as much to the plumbing around a model as they do to any other distributed system. This overlaps heavily with what you'll study as **Application/System Component attacks** later in this module and in **Module 7**.

---

## 12. ML10: Model Poisoning

**Plain English**: Rather than poisoning the *data* the model learns from (ML02), the attacker directly tampers with the *trained model artifact itself* -- its weight files, checkpoints, or configuration -- typically by compromising storage or the deployment pipeline.

### The Analogy

ML02 (data poisoning) is like sabotaging the ingredients before a cake is baked. ML10 (model poisoning) is like breaking into the bakery *after* the cake is already baked and perfect, and injecting something harmful directly into the finished product before it ships to the store.

### What It Looks Like in Practice

- Gaining write access to a model registry or artifact storage bucket (e.g., an S3 bucket or MLflow registry) and swapping the "approved" model file with a subtly backdoored version that behaves identically except for one attacker-chosen trigger.
- Tampering with a model during a CI/CD deployment step so that the version pushed to production differs from the version that was actually tested and approved.

**Security Angle**: This category is really a **supply chain / infrastructure security** problem wearing ML clothing -- it's about who has write access to model artifacts, whether artifacts are signed/hashed, and whether there's integrity verification between "trained and validated" and "deployed to production." It overlaps with **System Component attacks** (covered later in this module) and with classic DevSecOps concerns.

---

## 13. Worked Example -- Auditing a Fraud-Detection Model

Let's walk through how a red teamer would use this checklist against a realistic target: a bank's ML-based credit card fraud detector, deployed as an internal API that flags transactions in real time and retrains weekly on the previous week's confirmed fraud/non-fraud labels.

```
TARGET SYSTEM OVERVIEW
+------------------+     +------------------+     +------------------+
| Transaction Data |---->| Fraud Model      |---->| Flag/Allow       |
| (weekly retrain) |     | (gradient-boosted|     | Decision --> Ops |
|                  |     |  tree ensemble)  |     | Team Queue       |
+------------------+     +------------------+     +------------------+
```

Walking the OWASP ML Top 10 against this system:

| Category | Question a Red Teamer Asks | Applicable Here? |
|----------|----------------------------|-------------------|
| ML01 Input Manipulation | Can I structure a fraudulent transaction (amount, timing, merchant category) to sit just under the model's flagging threshold? | Yes -- test with boundary-probing transactions |
| ML02 Data Poisoning | Who can influence the "confirmed fraud/non-fraud" labels used in weekly retraining? Could a compromised merchant account inject bad labels? | Yes -- labels come partly from customer disputes, an attacker-influenceable source |
| ML03 Model Inversion | Can repeated queries reveal details about specific customers' spending patterns? | Possible if confidence scores are exposed to internal tools with weak access control |
| ML04 Membership Inference | Can I determine whether a specific transaction record was in this week's training batch? | Lower priority here -- limited external query access |
| ML05 Model Theft | Is the model exposed via an internal API that a malicious insider or compromised partner integration could query at scale to clone it? | Yes -- worth rate-limit and access-control testing |
| ML06 AI Supply Chain | Does the gradient-boosting library or any pretrained feature-embedding model come from a third party? Verified integrity? | Yes -- check dependency provenance |
| ML07 Transfer Learning | N/A -- this model is trained from scratch on tabular data, no base model reused | Not applicable |
| ML08 Model Skewing | Since it retrains weekly on live-influenced labels, can an attacker slowly bias the threshold upward over months? | Yes -- high priority given the online-learning setup |
| ML09 Output Integrity | Is the flag/allow decision transmitted over an authenticated, encrypted channel to the payment processor? | Yes -- classic API security test |
| ML10 Model Poisoning | Who has write access to the model registry storing the weekly-retrained model artifact? | Yes -- test artifact storage access controls |

Out of 10 categories, 7 turned out to be relevant, 1 was a lower priority, and 1 was not applicable at all. **That's the value of the checklist**: it forces systematic coverage instead of testing only the attack you happen to think of first.

---

## 14. Security Angle -- Using the List as a Red Team Checklist

For an offensive AI engagement, treat the OWASP ML Top 10 the way a pentester treats the web OWASP Top 10: as a **scoping and coverage tool**, not a rigid script.

```
   RED TEAM WORKFLOW USING THE OWASP ML TOP 10
   
   1. RECON            2. MAP TO CATEGORIES        3. PRIORITIZE
   ---------           ---------------------        --------------
   - How is the        - Walk all 10 categories     - Rank by likely
     model deployed?     against the target           impact + feasibility
   - What's the data  - Mark: applicable /          - Focus effort on
     lifecycle?          not applicable /              top 2-3 first
   - Is it static or    needs more info
     continuously                                   4. EXECUTE
     retrained?                                      --------------
                                                      - Use techniques from
                                                        Modules 6-11 for
                                                        deep-dive testing
```

- **ML01 and ML05** are usually the fastest wins in a black-box engagement -- you often only need query access.
- **ML02, ML08, and ML10** require some visibility into the training/deployment pipeline (grey-box or white-box engagement, or a supply-chain angle).
- **ML03 and ML04** are privacy-focused findings that matter enormously for compliance-driven clients (healthcare, finance).
- **ML06 and ML07** should be asked about during initial scoping ("what pretrained models/datasets/libraries does this system depend on?") even before touching the live system.
- **ML09** is a reminder to never skip "boring" application security basics just because the target happens to involve ML.

---

## 15. Key Takeaways

- The **OWASP Machine Learning Security Top 10** is a community-curated checklist of the 10 most significant risk categories unique to (or amplified by) ML systems, mirroring the structure and purpose of the classic OWASP Web Top 10.
- The 10 categories map onto **stages of the ML lifecycle**: ML02/ML06/ML07/ML10 target training and supply chain; ML01/ML03/ML04/ML05 target the deployed/inference-time model; ML08 targets systems with continuous learning; ML09 targets the plumbing around the model.
- **ML01 (Input Manipulation)** and **ML05 (Model Theft)** are typically the easiest to test in black-box engagements since they only require query access.
- **ML02 (Data Poisoning)** and **ML10 (Model Poisoning)** both corrupt the model, but at different stages -- poisoning the ingredients (data) vs. tampering with the finished product (the trained artifact).
- **ML03 (Model Inversion)** and **ML04 (Membership Inference)** are privacy attacks that extract information *about the training data* from a model's behavior, not attacks that change the model's behavior.
- **ML06 (Supply Chain)** and **ML07 (Transfer Learning)** are increasingly critical because most real-world ML products are built on reused open-source models, datasets, and libraries rather than trained from scratch.
- Use this list as a **coverage checklist** during red team scoping: walk every category, mark applicable/not applicable, and prioritize by feasibility and impact before diving into deep technique work in later modules.

---

*Next up: the OWASP Top 10 for LLM Applications -- the equivalent checklist for large language model apps, covering risks like prompt injection, sensitive information disclosure, and excessive agency.*
