# Google's Secure AI Framework (SAIF)

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Introduction to Red Teaming AI | Section: Secure AI Framework (SAIF)

---

## Table of Contents

1. [What Is SAIF and Why Was It Created?](#1-what-is-saif-and-why-was-it-created)
2. [The Six Core Elements of SAIF](#2-the-six-core-elements-of-saif)
3. [Element-by-Element Breakdown](#3-element-by-element-breakdown)
4. [The SAIF Risk Map: Mapping SAIF to the ML Lifecycle](#4-the-saif-risk-map-mapping-saif-to-the-ml-lifecycle)
5. [How SAIF Relates to the OWASP Top 10 Lists](#5-how-saif-relates-to-the-owasp-top-10-lists)
6. [Worked Example -- Applying SAIF to a Company Rolling Out an AI Coding Assistant](#6-worked-example----applying-saif-to-a-company-rolling-out-an-ai-coding-assistant)
7. [Security Angle -- Why Red Teamers Should Care About a "Defender's Framework"](#7-security-angle----why-red-teamers-should-care-about-a-defenders-framework)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. What Is SAIF and Why Was It Created?

**SAIF** (Secure AI Framework) is a framework published by Google in 2023 that gives organizations a structured, practical way to think about securing AI systems -- from the earliest data collection through deployment and ongoing operation.

### The Analogy

Think about how the construction industry handles a genuinely new kind of building -- say, the first generation of skyscrapers, or the first wave of smart, internet-connected homes. Existing building codes (written for regular houses) don't automatically cover elevator safety, wind-sway engineering, or smart-lock cybersecurity. Eventually, someone has to write a *new* framework that says: "here are the categories of things you now need to think about, given this new type of structure."

SAIF plays exactly that role for AI systems. Traditional IT security frameworks (things like NIST's Cybersecurity Framework, or classic secure software development lifecycles) were designed around traditional software: source code, servers, networks, and databases. They don't natively address questions like "what happens if someone poisons my training data?" or "how do I detect that my model is being systematically queried to steal it?" SAIF was built specifically to close that gap -- while still trying to **reuse and extend** existing security practices rather than reinventing security from scratch.

### Why This Matters for a Red Teamer

You might reasonably ask: "I'm learning to *attack* AI systems -- why do I need to understand a *defender's* framework?" Two reasons:

1. **You need to know what a well-defended target looks like.** If a client has actually implemented SAIF's recommendations, your engagement will look very different (harder!) than one where they haven't. Understanding SAIF lets you quickly assess an organization's AI security maturity.
2. **SAIF's structure mirrors the attack surface you'll be testing.** SAIF organizes AI security around the same lifecycle stages (data, infrastructure, model, application) that attackers target. Learning SAIF's map of "what needs defending" is really learning a map of "what can be attacked," just described from the other side of the table.

---

## 2. The Six Core Elements of SAIF

SAIF is built around **six core elements** -- foundational, ongoing practices an organization should adopt, rather than a one-time checklist. Google frames them as things to weave into your *existing* security program, not a brand-new separate program to run in parallel.

```
+-----------------------------------------------------------------------+
|                    SAIF -- SIX CORE ELEMENTS                          |
+-----------------------------------------------------------------------+
|                                                                        |
|  1. EXPAND strong security foundations to the AI ecosystem            |
|  2. EXTEND detection and response to bring AI into an org's           |
|     threat universe                                                   |
|  3. AUTOMATE defenses to keep pace with existing and new threats      |
|  4. HARMONIZE platform-level controls for consistency across the org  |
|  5. ADAPT controls to adjust mitigations and create faster feedback   |
|     loops for AI deployment                                           |
|  6. CONTEXTUALIZE AI system risks in surrounding business processes   |
|                                                                        |
+-----------------------------------------------------------------------+
```

A handy mnemonic: **E-E-A-H-A-C** doesn't spell anything catchy, but each verb -- Expand, Extend, Automate, Harmonize, Adapt, Contextualize -- describes a specific *motion* an organization needs to make. Notice that four of the six elements are literally verbs about **taking something that already exists in your security program and stretching it to cover AI** (expand, extend, harmonize, adapt) -- SAIF is explicitly *not* asking companies to throw out their existing security investments and start over.

---

## 3. Element-by-Element Breakdown

### Element 1: Expand Strong Security Foundations to the AI Ecosystem

**Plain English**: Most of what already makes traditional software secure -- secure infrastructure, careful access control, tested supply chains -- still matters enormously for AI. Don't reinvent security; extend the good practices you already have to cover data pipelines, model training environments, and the tools your data scientists use.

**In practice**: Applying the same secure-by-default cloud infrastructure controls to your ML training clusters that you'd apply to any other production system. Making sure your model registry and dataset storage have the same access logging and encryption standards as your customer database.

### Element 2: Extend Detection and Response to Bring AI into an Organization's Threat Universe

**Plain English**: Your security operations center (SOC) needs to actually be watching for AI-specific attacks, not just traditional intrusion signals. If nobody is monitoring for "is someone sending 50,000 weird queries to our model API," a model-stealing attack (ML05) can run for months undetected.

**In practice**: Adding logging and alerting for unusual query volume/patterns against model APIs, and for anomalies in training data ingestion pipelines (a spike in newly-added samples all sharing a suspicious label, for example).

### Element 3: Automate Defenses to Keep Pace with Existing and New Threats

**Plain English**: AI itself can be used defensively -- e.g., using ML-based anomaly detection to catch AI-specific attacks faster than humans reviewing logs manually ever could. This mirrors an idea you'll see throughout this course: the same technology that creates new attack surface can also help defend it.

**In practice**: Using automated systems to detect adversarial-example-style query patterns against a production model in real time, or automatically flagging suspicious data uploads to a training pipeline for human review.

### Element 4: Harmonize Platform-Level Controls to Ensure Consistent Security Across the Organization

**Plain English**: If every team in a large company builds and deploys AI models their own way, with inconsistent security practices, you end up with wildly uneven risk across the organization -- one team's model registry is locked down, another's is a public S3 bucket. This element is about establishing **consistent, platform-wide controls** that every AI project automatically inherits.

**In practice**: A shared, centrally-managed ML platform where every team's training pipeline automatically gets the same access controls, artifact-signing, and vulnerability scanning, instead of each team reinventing (and likely under-implementing) their own security.

### Element 5: Adapt Controls to Adjust Mitigations and Create Faster Feedback Loops for AI Deployment

**Plain English**: AI systems evolve fast -- new attack techniques against models are published constantly (this entire certification is proof of how quickly this field moves), and models themselves can be updated or retrained frequently. Security controls need mechanisms to be updated *quickly* in response, rather than being locked in place for years like some traditional IT policies.

**In practice**: Building red-teaming and adversarial testing directly into the CI/CD pipeline for model updates, so that every new model version is automatically tested against known adversarial techniques before deployment -- creating a fast feedback loop between "new attack discovered" and "defense deployed."

### Element 6: Contextualize AI System Risks in Surrounding Business Processes

**Plain English**: A vulnerability's severity depends heavily on *what the AI system is actually used for* in the business. A hallucination bug in an internal brainstorming tool is a minor annoyance; the same bug in a system that automatically approves loans or diagnoses patients is a serious risk. This element is about performing **end-to-end risk assessments** that account for the AI's real-world business context, not evaluating the model in a vacuum.

**In practice**: Before deploying an LLM-powered financial advice tool, mapping out the full business process it plugs into -- who acts on its outputs, what happens if it's wrong, what regulatory obligations apply -- rather than only benchmarking the model's raw accuracy.

---

## 4. The SAIF Risk Map: Mapping SAIF to the ML Lifecycle

Beyond the six core elements, SAIF also provides a practical **risk map** that ties specific risks to specific components of an AI system across its lifecycle. This is the part that will feel most familiar, because it echoes the ML pipeline diagram from Module 1 and the OWASP Top 10 categories you just studied.

```
                        THE SAIF AI LIFECYCLE / RISK MAP
   
   +-------------+     +-----------------+     +-------------+     +-------------+
   |    DATA     |---->| INFRASTRUCTURE  |---->|    MODEL    |---->| APPLICATION |
   |             |     |                 |     |             |     |             |
   | - Collection|     | - Training      |     | - Model     |     | - Prompts/  |
   | - Storage   |     |   compute       |     |   weights   |     |   inputs    |
   | - Labeling  |     | - Model storage |     | - Model     |     | - Plugins/  |
   |             |     |   / registry    |     |   source    |     |   tools     |
   +-------------+     +-----------------+     +-------------+     +-------------+
        |                      |                      |                    |
        v                      v                      v                    v
   Data poisoning,        Supply chain            Model theft,        Prompt injection,
   sensitive data          compromise,             model inversion,    insecure output
   exposure                infra takeover          backdoors           handling
```

For each component, SAIF asks organizations to identify:

| Component | Key Question SAIF Wants You to Answer | Maps to OWASP Category |
|-----------|------------------------------------------|---------------------------|
| **Data** | Where does training data come from, and who could tamper with it before it's used? | ML02 (Data Poisoning), LLM04 (Data/Model Poisoning) |
| **Infrastructure** | Who has access to training compute, storage, and the model registry/pipeline? | ML06/LLM03 (Supply Chain), ML10 (Model Poisoning) |
| **Model** | Can the model itself be stolen, inverted, or made to leak training data? | ML03/ML04/ML05 (Inversion, Membership Inference, Theft) |
| **Application** | How is the model exposed to users, and what can attacker-controlled input do once it reaches the model? | ML01 (Input Manipulation), LLM01/LLM05/LLM06 (Prompt Injection, Output Handling, Excessive Agency) |

**Notice the pattern**: SAIF's four lifecycle stages (Data --> Infrastructure --> Model --> Application) are essentially a coarser-grained version of the same pipeline you keep seeing throughout this course. This is exactly the lifecycle this module's next section (Attack Surface by Component) will walk through in more offense-focused detail.

---

## 5. How SAIF Relates to the OWASP Top 10 Lists

It's worth being explicit about how these three frameworks fit together, since you now know all three.

```
    THREE FRAMEWORKS, THREE PURPOSES

    OWASP ML Top 10        --> "What are the 10 biggest risks for a
                                generic ML model?" (offense-oriented
                                checklist)

    OWASP LLM Top 10        --> "What are the 10 biggest risks
                                specific to LLM applications?"
                                (offense-oriented checklist)

    Google SAIF              --> "How should an organization structure
                                its AI security PROGRAM end-to-end?"
                                (defense-oriented framework)
```

- The OWASP lists are **catalogs of specific risks** -- great for red teamers running an engagement checklist.
- SAIF is a **programmatic framework** -- great for understanding an organization's overall security maturity and for structuring how a security program (including red teaming!) fits into the bigger picture.
- They are complementary, not competing: a mature organization following SAIF's "Adapt" element (Element 5, fast feedback loops) would likely be the kind of organization that *commissions regular red team engagements* using the OWASP Top 10 lists as their testing checklist.

---

## 6. Worked Example -- Applying SAIF to a Company Rolling Out an AI Coding Assistant

Let's walk through how a security team (or a red teamer assessing that security team's maturity) would apply SAIF to a real rollout: a mid-size software company deploying an internal AI coding assistant, fine-tuned on the company's own codebase, that developers use directly inside their IDE and that can also open pull requests autonomously.

### Step 1: Map the Lifecycle (SAIF Risk Map)

```
   DATA                 INFRASTRUCTURE           MODEL                APPLICATION
   ----                 ---------------          -----                -----------
   Internal codebase    Fine-tuning compute      Fine-tuned base      IDE plugin +
   (incl. private        + model storage in       model (proprietary   autonomous PR
   repos, past PRs,      cloud provider            code baked in)       creation tool
   commit history)       account)
```

### Step 2: Ask SAIF's Six Element Questions

| Element | Question Asked | Finding |
|---------|-----------------|---------|
| 1. Expand | Are the training data (private repos) and fine-tuned model artifacts protected with the same access controls as production source code? | Gap found -- the fine-tuning dataset bucket has looser permissions than the main repo |
| 2. Extend | Is there monitoring for unusual query patterns against the assistant (e.g., a compromised developer account exfiltrating proprietary code via crafted prompts)? | Gap found -- no logging on assistant queries at all |
| 3. Automate | Are new model versions automatically tested against known prompt-injection and code-generation-safety techniques before rollout? | Gap found -- no automated red-team testing in CI/CD |
| 4. Harmonize | Does every team using this assistant get the same security controls, or did one team stand up their own unmanaged instance? | Gap found -- a second team spun up a shadow deployment without security review |
| 5. Adapt | How quickly could the security team push a fix if a new prompt-injection technique against code assistants was published tomorrow? | Slow -- no fast feedback loop exists yet |
| 6. Contextualize | What's the actual blast radius if this assistant is tricked into opening a malicious PR? Does it require human review before merge, or can it auto-merge? | Critical finding -- some repos allow auto-merge for PRs from the bot's service account |

### Step 3: Translate Findings into Red Team Priorities

Given these gaps, a red team engagement against this system would prioritize:

1. **LLM01 (Prompt Injection)** targeting the IDE plugin -- since Element 2/3 gaps mean injected instructions likely go undetected.
2. **LLM06 (Excessive Agency)** around the auto-merge capability -- Element 6's finding makes this the highest-impact path (a successful injection could ship malicious code straight to production).
3. **LLM02/ML03 (Sensitive Information Disclosure / Model Inversion)** -- testing whether the proprietary codebase baked into the fine-tuned model can be extracted by another team or an external party who gains query access.

This is exactly why understanding SAIF sharpens your red teaming: it told you, before you even started testing, *where the organization's actual weak points are* and which OWASP categories to prioritize.

---

## 7. Security Angle -- Why Red Teamers Should Care About a "Defender's Framework"

- **Scoping conversations go faster.** When you ask a client "have you implemented anything like SAIF, or a similar AI governance framework?", their answer tells you immediately how mature their AI security posture is likely to be, which shapes your entire testing strategy.
- **You can speak the client's language.** Many enterprise security and compliance teams are increasingly familiar with SAIF (and similar frameworks like the NIST AI Risk Management Framework). Being able to frame your red team findings in terms of "this is a gap in your Element 2 detection capability" makes your reports land better with technical leadership and auditors.
- **SAIF's lifecycle map doubles as a recon checklist.** Before touching any tooling, walking through Data --> Infrastructure --> Model --> Application for your target tells you what to ask for in scoping: What data sources feed this model? Who controls the infrastructure? Where does the trained model live? How is it exposed to end users?
- **Gaps in SAIF elements predict where you'll find easy wins.** An organization that hasn't implemented Element 2 (Extend detection and response) is an organization that likely won't notice your model-stealing queries or your prompt injection attempts -- useful operational intelligence for planning a stealthy engagement.

---

## 8. Key Takeaways

- **SAIF (Secure AI Framework)** is Google's framework for structuring an organization's AI security program end-to-end, extending existing security practices to cover AI-specific risks rather than replacing them.
- SAIF is built around **six core elements**: Expand (foundations), Extend (detection/response), Automate (defenses), Harmonize (platform controls), Adapt (fast feedback loops), and Contextualize (business risk).
- SAIF's **risk map** organizes the AI lifecycle into four stages -- **Data, Infrastructure, Model, Application** -- each with characteristic risks, directly echoing the ML pipeline and the OWASP Top 10 categories you already studied.
- SAIF and the OWASP Top 10 lists are **complementary**: SAIF is a program-level, defense-oriented framework; the OWASP lists are testing checklists you use as a red teamer.
- Understanding SAIF lets you **assess an organization's AI security maturity quickly during scoping**, predict where the easy wins likely are, and communicate findings in language that resonates with security leadership and auditors.
- The worked coding-assistant example shows the practical payoff: mapping SAIF's six elements against a real system directly produced a prioritized red team testing plan -- SAIF told you where to point the OWASP checklist first.

---

*Next up: Attack Surface by Component -- a deep dive into the Model, Data, Application, and System components of a deployed AI system, and the categories of attacks that target each one.*
