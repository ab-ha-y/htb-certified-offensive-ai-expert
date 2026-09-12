# Attack Surface by Component: The System Component

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Introduction to Red Teaming AI | Section: Attack Surface by Component -- System

---

## Table of Contents

1. [Where This Fits](#1-where-this-fits)
2. [What Is the "System Component"?](#2-what-is-the-system-component)
3. [Where the System Component Sits in a Deployed AI System](#3-where-the-system-component-sits-in-a-deployed-ai-system)
4. [Categories of Attacks Against the System Component](#4-categories-of-attacks-against-the-system-component)
5. [Terminology Reference Table](#5-terminology-reference-table)
6. [Worked Example -- Scoping the System Component of an ML Platform](#6-worked-example----scoping-the-system-component-of-an-ml-platform)
7. [Security Angle -- Why "Boring" Infrastructure Security Is Still AI Security](#7-security-angle----why-boring-infrastructure-security-is-still-ai-security)
8. [Putting All Four Components Together](#8-putting-all-four-components-together)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. Where This Fits

```
             THE FOUR COMPONENTS OF A DEPLOYED AI SYSTEM

    +-----------+     +-----------+     +--------------+     +-----------+
    |   DATA    |---->|   MODEL   |---->| APPLICATION  |---->|  SYSTEM   |
    | Component |     | Component |     |  Component   |     | Component |
    |  (earlier |     | (earlier  |     |  (earlier    |     |   (this   |
    |   file)   |     |   file)   |     |    file)     |     |    file)  |
    +-----------+     +-----------+     +--------------+     +-----------+
```

This is the last of the four component files, and in some ways the most foundational: the **System Component**. Everything you've read about so far -- data pipelines, trained models, application logic -- has to physically run *somewhere*, on *something*. This file covers that "somewhere." The deep hands-on technique work for many of these attacks connects into **Module 7** (application/system-adjacent attacks) and echoes classic infrastructure security you may already know from traditional pentesting.

---

## 2. What Is the "System Component"?

### The Analogy

If the model is the engine, the data is the fuel supply, and the application is the dashboard and controls, the **System component** is the **factory, garage, and road network** that the entire car depends on to exist and function at all. You can build the most sophisticated engine and the most secure dashboard in the world, but if the factory that manufactures the car has an unlocked back door, or the roads it drives on have no traffic laws, none of that engineering matters. Attackers frequently skip the fancy engine and dashboard entirely and just walk in through the garage's unlocked door.

### The Formal Definition

The **System Component** covers the underlying **infrastructure, servers, cloud services, storage, and network** that host every other component of an AI system. This includes:

- **Compute infrastructure**: the servers, GPUs/TPUs, or cloud instances used for training and serving models.
- **Storage systems**: databases, object storage (e.g., S3 buckets), and model registries holding datasets and trained artifacts.
- **Network infrastructure**: the APIs, load balancers, and network paths connecting all the other components.
- **Identity and access management (IAM)**: who/what has permission to read, write, or execute each of the above.
- **CI/CD and deployment pipelines**: the automated processes that move code, data, and models from development into production.
- **Container/orchestration layers**: Docker, Kubernetes, or similar systems that package and run the model-serving software.

This is the component that has the **most overlap with traditional IT/cloud security** -- most of what you already know (or will learn) about general penetration testing and cloud security assessments applies directly here.

---

## 3. Where the System Component Sits in a Deployed AI System

```
              A DEPLOYED AI SYSTEM, SYSTEM-CENTRIC VIEW

   +-----------------------------------------------------------------+
   |                        SYSTEM COMPONENT                          |
   |                                                                   |
   |   +-------------+   +-------------+   +-------------+            |
   |   |  Training   |   |   Model     |   |  Serving /  |            |
   |   |  Compute     |   |  Registry / |   |  API        |            |
   |   |  (GPU/TPU    |   |  Storage    |   |  Infra      |            |
   |   |  clusters)   |   | (weights,   |   | (containers,|            |
   |   |             |   |  checkpoints)|   |  load       |            |
   |   |             |   |             |   |  balancers) |            |
   |   +-------------+   +-------------+   +-------------+            |
   |                                                                   |
   |   +-------------+   +-------------+                              |
   |   |  Data Lake / |   | CI/CD /     |                              |
   |   |  Feature      |   | Deployment  |                              |
   |   |  Store        |   | Pipeline    |                              |
   |   +-------------+   +-------------+                              |
   |                                                                   |
   |   IAM / access control wraps around ALL of the above              |
   +-----------------------------------------------------------------+
              ^                    ^                     ^
              |                    |                     |
        DATA component       MODEL component      APPLICATION component
        physically lives      physically lives     physically runs
        on this infra         on this infra        on this infra
```

The key insight here is that the System component **isn't really a fourth stage in the pipeline** the way Data --> Model --> Application flows conceptually -- it's the substrate that the other three components physically sit on top of, all at once. Compromising the System component is often the most efficient way to compromise *everything else*, because you're not attacking one component's logic, you're attacking the ground all of them stand on.

---

## 4. Categories of Attacks Against the System Component

```
+-----------------------------------------------------------------------+
|                SYSTEM COMPONENT -- ATTACK CATEGORY MAP                 |
+-----------------------------------------------------------------------+
|                                                                        |
|  AI SUPPLY CHAIN COMPROMISE        Malicious dependency, pretrained   |
|  (ML06 / LLM03)                    model, or library                  |
|                                     --> Deep dive: Module 7             |
|                                                                        |
|  MODEL/ARTIFACT TAMPERING          Unauthorized write access to        |
|  (ML10)                            model registry/storage              |
|                                     --> Overlaps with Model component  |
|                                       file; deep dive: Module 7         |
|                                                                        |
|  INSECURE ACCESS CONTROL / IAM     Overly broad permissions on         |
|                                     training/serving infra              |
|                                     --> Classic cloud security,        |
|                                       reinforced in Module 7            |
|                                                                        |
|  CI/CD PIPELINE COMPROMISE         Injecting malicious code/artifacts  |
|                                     during automated build/deploy       |
|                                     --> Classic DevSecOps, reinforced  |
|                                       in Module 7                       |
|                                                                        |
|  OUTPUT INTEGRITY / MITM           Tampering with predictions in       |
|  (ML09)                            transit between components         |
|                                     --> Classic network security       |
|                                                                        |
|  RESOURCE EXHAUSTION / DENIAL      Runaway cost or availability        |
|  OF SERVICE (LLM10)                impact from unbounded resource use  |
|                                     --> Classic infra/cost security    |
|                                                                        |
+-----------------------------------------------------------------------+
```

### 4.1 AI Supply Chain Compromise (ML06 / LLM03)

**What it is at a glance**: A compromised third-party dependency -- a pretrained model downloaded from a public hub, a poisoned library, a tampered dataset -- gets pulled into the target's infrastructure and trusted.

**Why it targets the system component specifically**: The compromise happens at the point where external artifacts get **pulled into your infrastructure** -- package managers, model download scripts, container base images. Whether the org has any process to verify integrity (hashes, signatures, provenance checks) before trusting these artifacts is fundamentally an infrastructure/process question.

**Forward pointer**: You already met this category twice (ML06 in the ML Top 10, LLM03 in the LLM Top 10). **Module 7** builds on this orientation with concrete techniques for identifying and exploiting supply-chain weaknesses in real ML deployments.

### 4.2 Model/Artifact Tampering (ML10)

**What it is at a glance**: Gaining unauthorized write access to wherever a trained model artifact is stored (a model registry, an object storage bucket, a deployment package) and swapping it for a subtly backdoored version.

**Why it targets the system component specifically**: This attack is entirely about **who has write access to storage and deployment pipelines** -- it requires no understanding of the model's mathematics at all, just conventional access-control exploitation.

**Note**: You also saw this in the Model Component file (Section 4.5) -- that's intentional. This is a great example of an attack that sits right at the seam between two components: the *target* is the model artifact, but the *mechanism* is a system-layer access control failure. Real attacks often don't respect our tidy four-box diagram.

### 4.3 Insecure Access Control / IAM

**What it is at a glance**: Overly broad permissions on training infrastructure, storage, or serving environments -- e.g., a data scientist's laptop credentials having write access to the production model registry, or a public-facing storage bucket accidentally left open.

**Why it targets the system component specifically**: This is a direct, unmodified application of classic cloud/infrastructure security principles (least privilege, defense in depth) to AI-specific assets. If you've done any cloud penetration testing before, this category will feel immediately familiar.

**Forward pointer**: Reinforced with AI-specific scenarios in **Module 7**, but the underlying skills (enumerating IAM roles, testing for privilege escalation paths, finding misconfigured storage) come from general offensive security training.

### 4.4 CI/CD Pipeline Compromise

**What it is at a glance**: Injecting malicious code, data, or model artifacts during the automated build/test/deploy process, so the version that reaches production differs from the version that was actually reviewed and approved.

**Why it targets the system component specifically**: CI/CD pipelines are infrastructure -- build servers, deployment scripts, artifact repositories -- and a compromise here can silently affect the Data, Model, or Application component without anyone noticing, since the "approved" artifact and the "deployed" artifact are supposed to be identical but no longer are.

**Forward pointer**: Classic DevSecOps risk, reinforced with ML-pipeline-specific scenarios in **Module 7**.

### 4.5 Output Integrity / Man-in-the-Middle (ML09)

**What it is at a glance**: Intercepting or tampering with a model's output somewhere between the model producing it and a downstream system acting on it.

**Why it targets the system component specifically**: This is purely a **network and transport security** problem -- is the channel authenticated and encrypted, are messages integrity-checked -- applied to the specific case of an ML prediction traveling between two services.

**Forward pointer**: This is essentially a reminder that classic network security testing (checking for unauthenticated internal APIs, insecure message queues, missing TLS) remains relevant even in a system full of ML components.

### 4.6 Resource Exhaustion / Denial of Service (LLM10)

**What it is at a glance**: Exploiting a lack of rate limiting, quotas, or resource caps to drive up cost or degrade availability -- the "denial of wallet" and "denial of service" concepts you met in the LLM Top 10 file.

**Why it targets the system component specifically**: Rate limiting, autoscaling limits, and cost alerting are all infrastructure-layer controls -- whether they exist (and are configured sensibly) is an infrastructure/ops decision, independent of how good the model or application logic is.

---

## 5. Terminology Reference Table

| Term | Definition | Plain English |
|------|-----------|----------------|
| **IAM (Identity and Access Management)** | The system controlling who/what can access which resources. | The building's keycard system -- who gets which doors. |
| **Model registry** | A managed storage system for trained model artifacts, often with versioning. | A library specifically for model files, with a checkout history. |
| **CI/CD (Continuous Integration/Continuous Deployment)** | Automated pipelines that build, test, and deploy code/artifacts. | The assembly line that turns approved changes into a live product automatically. |
| **Artifact signing** | Cryptographically signing a build artifact so tampering can be detected. | A tamper-evident seal on a shipped package. |
| **Least privilege** | Granting only the minimum access needed to perform a task. | Giving the new intern a key to their own office, not the master key to the building. |
| **Denial of wallet** | A denial-of-service variant that drives up a target's cloud/API costs rather than crashing the service. | Running up someone's credit card instead of cutting their power. |
| **Provenance** | Verifiable record of an artifact's origin and processing history. | The paper trail proving where something came from. |

---

## 6. Worked Example -- Scoping the System Component of an ML Platform

A fintech company runs an internal ML platform: data scientists train fraud-detection models on a shared GPU cluster, push approved models to a model registry, and a CI/CD pipeline automatically deploys the latest registry version to a production API.

```
TARGET: Internal ML Platform
+-------------+     +----------------+     +----------------+     +---------+
| Shared GPU  |---->| Model Registry |---->| CI/CD Deploy   |---->| Prod API|
| Cluster     |     | (versioned     |     | Pipeline        |     |         |
| (training)  |     | artifacts)     |     |                |     |         |
+-------------+     +----------------+     +----------------+     +---------+
      ^                     ^                      ^
      |                     |                      |
  All data scientists  Who can write here?   Who can trigger a deploy?
  share access
```

| Category | Feasibility Assessment | Notes |
|----------|------------------------|-------|
| AI Supply Chain | Medium -- check whether any pretrained base models or third-party libraries used on the cluster are verified for integrity | Standard supply-chain audit questions apply |
| Model/Artifact Tampering | High -- "all data scientists share access" to the training cluster is a red flag; test whether any of them (or a compromised account) can write directly to the registry, bypassing review | Top priority given the shared-access design |
| Insecure IAM | High -- map every role's actual permissions against what they need; look for overly broad service account permissions | Classic cloud security assessment |
| CI/CD Compromise | High -- who can trigger or modify the deploy pipeline? Is there any verification that the deployed artifact matches the reviewed one? | Test the "gap" between review and deployment |
| Output Integrity/MITM | Medium -- check whether internal service-to-service traffic (registry --> CI/CD --> prod API) is authenticated and encrypted | Standard internal network security test |
| Resource Exhaustion | Low -- internal platform, not directly exposed to external attacker-driven cost abuse in this scenario | Lower priority given the trust model described |

This example shows how System-component findings often look almost identical to a **traditional cloud/DevOps penetration test** -- the AI-specific framing (fraud model, registry) just tells you *which* assets matter most.

---

## 7. Security Angle -- Why "Boring" Infrastructure Security Is Still AI Security

- It's tempting, as a newcomer excited about adversarial examples and prompt injection, to think of System-component findings as "not real AI security." Resist that instinct: **many real-world AI incidents trace back to a boring infrastructure misconfiguration**, not a sophisticated adversarial attack -- an open storage bucket containing a proprietary model, an over-permissioned service account, an unpatched CI/CD tool.
- The System component is also where **the highest-impact attacks often live**, because compromising infrastructure frequently grants access to *everything* sitting on top of it -- the data, the model, and the application, all at once, rather than one component's isolated logic.
- If you already have traditional penetration testing or cloud security experience, this component is where that experience transfers **directly**, with almost no AI-specific translation needed. Your existing skills at finding misconfigured IAM roles, exposed storage, and weak CI/CD pipelines are immediately valuable in an AI red teaming context.
- A practical scoping habit: **always ask for the full infrastructure diagram before testing model-specific attacks**. Knowing where the model registry lives, who can deploy to production, and how training data flows into storage often reveals the fastest, highest-impact path into the system -- sometimes faster than any adversarial-example technique would be.

---

## 8. Putting All Four Components Together

You've now walked through all four components. Here's the complete picture, tying back to everything in this module:

```
+------------------------------------------------------------------------+
|                    THE COMPLETE ATTACK SURFACE MAP                      |
+------------------------------------------------------------------------+
|                                                                          |
|  DATA            MODEL           APPLICATION        SYSTEM              |
|  --------        --------        --------------     --------            |
|  Poisoning       Evasion         Prompt Injection   Supply Chain        |
|  Label Flipping  Extraction      System Prompt Leak Artifact Tampering  |
|  Backdoors       Inversion       Insecure Output    Insecure IAM        |
|  Model Skewing   Membership Inf. Excessive Agency   CI/CD Compromise    |
|                  Model Poisoning RAG Abuse           Output Integrity   |
|                                                       Resource Exhaustion|
|                                                                          |
|  --> Module 6    --> Modules     --> Module 7        --> Module 7       |
|                    8, 9, 10,                                            |
|                    11 (privacy)                                         |
+------------------------------------------------------------------------+
```

| Component | OWASP ML Top 10 Overlap | OWASP LLM Top 10 Overlap | Where It's Most Like Traditional Security |
|-----------|--------------------------|----------------------------|---------------------------------------------|
| **Data** | ML02 (Poisoning), ML08 (Skewing) | LLM04 (Data/Model Poisoning) | Least similar -- mostly AI-specific |
| **Model** | ML01, ML03, ML04, ML05, ML10 | (Indirectly, via foundation model risk) | Least similar -- the most AI-unique layer |
| **Application** | ML01 (Input Manipulation, overlapping) | LLM01, LLM05, LLM06, LLM07, LLM08 | Moderately similar -- resembles web/API security with new risks layered on |
| **System** | ML06, ML09 | LLM03, LLM10 | Most similar -- classic cloud/infra/DevSecOps security applies almost directly |

Use this table as your mental index for the rest of the certification: whenever you learn a new attack technique in a later module, ask yourself **which component it targets**, and you'll immediately know how it connects to everything else you've learned in this module.

---

## 9. Key Takeaways

- The **System Component** is the infrastructure substrate -- compute, storage, network, IAM, and CI/CD -- that the Data, Model, and Application components all physically run on top of.
- Six major categories target this layer: **AI Supply Chain Compromise**, **Model/Artifact Tampering**, **Insecure Access Control/IAM**, **CI/CD Pipeline Compromise**, **Output Integrity/MITM**, and **Resource Exhaustion/DoS**.
- This component has the **most overlap with traditional cloud and infrastructure security** -- skills from classic penetration testing transfer here almost directly, with minimal AI-specific translation needed.
- Compromising the System component is often the **highest-impact** path into an AI system, because it can grant access to everything running on top of it at once, rather than one component's isolated logic.
- Some attacks (like **Model/Artifact Tampering**) deliberately sit at the seam between two components -- the target is the model, but the mechanism is a system-layer access failure. Real-world attacks rarely respect clean component boundaries.
- Across all four components, the pattern to remember is: **Data feeds Model, Model is wrapped by Application, and all three run on System** -- and every attack technique you'll learn in Modules 6 through 11 slots neatly into one (or more) of these four boxes.

---

*Next up: with the orientation map for Red Teaming AI complete -- the OWASP ML and LLM Top 10 lists, Google's SAIF framework, and the four-component attack surface -- the course moves into hands-on technique modules, starting with Module 6: Data Attacks.*
