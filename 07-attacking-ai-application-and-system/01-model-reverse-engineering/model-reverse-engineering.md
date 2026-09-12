# Model Reverse Engineering

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Attacking AI - Application and System | Section: Model Reverse Engineering

---

## Table of Contents

1. [What Is Model Reverse Engineering?](#1-what-is-model-reverse-engineering)
2. [How It Differs from "Model Stealing"](#2-how-it-differs-from-model-stealing)
3. [The Application-Layer Attack Surface](#3-the-application-layer-attack-surface)
4. [Technique 1: API Fingerprinting](#4-technique-1-api-fingerprinting)
5. [Technique 2: Timing and Side-Channel Analysis](#5-technique-2-timing-and-side-channel-analysis)
6. [Technique 3: Artifact Inspection](#6-technique-3-artifact-inspection)
7. [Worked Example -- Fingerprinting a Mystery Classifier](#7-worked-example----fingerprinting-a-mystery-classifier)
8. [Access Models: Black-Box, Grey-Box, and White-Box](#8-access-models-black-box-grey-box-and-white-box)
9. [Reverse Engineering Tooling Reference](#9-reverse-engineering-tooling-reference)
10. [Security Angle](#10-security-angle)
11. [Defensive Countermeasures](#11-defensive-countermeasures)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. What Is Model Reverse Engineering?

**Reverse engineering**, in the general software sense, means taking a finished product and working backward to figure out how it was built -- without having the original source code or blueprints. Model reverse engineering applies that same idea to a deployed machine learning (ML) model: an attacker who only has access to the *outside* of a model (an API endpoint, a mobile app, a downloadable file) tries to work out what is happening on the *inside* (the architecture, the training data, the exact weights, or simply "what makes this thing tick").

### The Analogy

Imagine a food critic who is not allowed into a restaurant's kitchen but is determined to figure out the secret recipe for a famous sauce. They cannot read the chef's handwritten recipe card (the "source code"), so instead they:

- Order the dish many times and taste it carefully, trying to detect individual ingredients (**probing the API**).
- Time how long each dish takes to arrive, guessing that more complex dishes take longer to prepare (**timing side channels**).
- Dig through the restaurant's trash for empty ingredient containers and packaging (**artifact inspection**).

None of this requires stealing the recipe card. It is all indirect evidence, pieced together into a working reconstruction of the recipe. Model reverse engineering is the same process applied to an ML system: you never see the "recipe" (the training code and weights), but you can infer a great deal from how the finished product behaves and what leftover artifacts it exposes.

### Formal Definition

**Model reverse engineering** is the process of inferring a deployed model's internal properties -- its architecture (type of neural network, number of layers, input/output shapes), its hyperparameters, its approximate decision boundaries, or even its exact weights -- through indirect means: sending it inputs and observing outputs (**black-box querying**), measuring how it behaves physically (**side channels**), or examining files and metadata associated with it (**artifact analysis**).

---

## 2. How It Differs from "Model Stealing"

If you studied Module 1 (Fundamentals of AI), you already met **model stealing** (also called model extraction): repeatedly querying a model to train a "shadow" clone that mimics its behavior. Model reverse engineering is a closely related but distinct discipline, and the exam expects you to know the difference.

| Aspect | Model Stealing (Extraction) | Model Reverse Engineering |
|--------|------------------------------|----------------------------|
| **Primary goal** | Recreate a *functionally equivalent* model (a working clone) | Understand the *internals* of the target model (architecture, framework, config, behavior) |
| **Output of the attack** | A new trained model that behaves like the original | Knowledge/documentation about the original model -- may or may not lead to a clone |
| **Typical method** | Large-scale querying + training a substitute model on the responses | API fingerprinting, timing analysis, file/artifact inspection, small-scale probing |
| **Where it lives in the workflow** | Usually the *end goal* of an attack chain | Usually the *reconnaissance phase* that makes other attacks (evasion, extraction, poisoning) possible |
| **Analogy** | Cloning the recipe well enough to cook the same dish | Figuring out which kitchen, which oven brand, and which supplier the restaurant uses |

**In short**: reverse engineering is reconnaissance -- it produces intelligence about the target. Model stealing is often the payoff that reconnaissance enables. In a real engagement, you almost always reverse-engineer *before* you attempt to steal, evade, or poison a model, because you need to know what you are dealing with first.

---

## 3. The Application-Layer Attack Surface

Module 1 focused on the model itself as a mathematical object. This module -- "Attacking AI: Application and System" -- looks at the model in the context of a real, deployed **AI application**: the client, the backend code, the orchestration layer, the model's own API, any tools/plugins it can call, and the data stores behind it all.

```
                     TYPICAL AI APPLICATION STACK

  +------------+     +-------------+     +-------------------+     +-----------+     +-------------+
  |            |     |             |     |                    |     |           |     |             |
  |   CLIENT   |---->|  APP LAYER  |---->|  ORCHESTRATION /   |---->|  MODEL    |---->|  TOOLS /    |
  |  (browser, |     |  (backend   |     |  AGENT FRAMEWORK   |     |  API      |     |  PLUGINS    |
  |  mobile,   |     |  API, auth, |     |  (LangChain, agent |     | (LLM,     |     |  (search,   |
  |  CLI)      |     |  routing)   |     |  loop, memory)     |     | classifier|     |  code exec, |
  |            |     |             |     |                    |     | etc.)     |     |  DB query)  |
  +------------+     +-------------+     +-------------------+     +-----------+     +-------------+
                                                                          |                  |
                                                                          v                  v
                                                                    +---------------------------+
                                                                    |       DATA STORES         |
                                                                    | (vector DB, logs, cache,   |
                                                                    |  model registry, configs)  |
                                                                    +---------------------------+

  RECON / REVERSE-ENGINEERING TARGETS:  ^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^   ^^^^^^^^^^   ^^^^^^^^^^^^
                                          response       framework version     model        tool schemas
                                          headers,        leaks, error          fingerprint  /descriptions
                                          error text      messages              via timing
```

Model reverse engineering, as covered in this section, sits mostly at the **model API** and **client/app layer** boundary -- what can be learned by talking to the system from the outside -- plus a bit of **artifact** work if you can get your hands on files (a downloaded `.onnx`/`.pt`/`.safetensors` file, a container image, a mobile app bundle with an embedded model).

---

## 4. Technique 1: API Fingerprinting

**API fingerprinting** means sending crafted requests to a model's API and studying the exact shape of the responses -- error messages, headers, formatting quirks, token limits, confidence score precision -- to figure out what software and model family is running behind it.

### Why This Works

Most production ML/LLM APIs are built on top of a small number of well-known frameworks and model families (TensorFlow Serving, TorchServe, Triton Inference Server, vLLM, Hugging Face `transformers`, OpenAI-compatible gateways, etc.). Each of these has distinctive fingerprints:

| Signal | What It Can Reveal |
|--------|--------------------|
| **HTTP response headers** | Server software (`X-Powered-By`, `Server:`), framework version numbers |
| **Error message format** | Underlying framework (e.g., a raw Python traceback from Flask/FastAPI vs. a generic JSON error) |
| **Input validation quirks** | Expected tensor shape, max sequence length, image resolution the model was trained on |
| **Output formatting** | Number of decimal places in confidence scores, whether logits vs. probabilities are returned, JSON schema of the response |
| **Rate limiting behavior** | Infrastructure choices (API gateway type, whether there's a queue) |
| **Token/character limits** | Which base LLM family is likely being used (context window size is a strong hint) |
| **Refusal phrasing (LLMs)** | Which vendor's safety training produced this exact refusal wording |
| **Special/malformed input handling** | How gracefully (or not) the backend handles edge cases -- reveals the serving framework's defaults |

### A Simple Fingerprinting Workflow

```
 1. Send a baseline valid request      --> record response shape, headers, latency
 2. Send a deliberately malformed      --> record the exact error text/stack trace
    request (wrong content-type,
    oversized payload, wrong shape)
 3. Send boundary-testing inputs       --> find max input length, min/max values accepted
 4. Send inputs known to trigger       --> compare confidence score precision, rounding,
    specific model families              class label naming conventions
 5. Correlate all signals against a    --> narrow down: framework? model family? approx.
    known-fingerprint database            version?
```

---

## 5. Technique 2: Timing and Side-Channel Analysis

A **side channel** is information that leaks unintentionally through a system's *physical* behavior rather than through its intended output. **Timing side channels** are the classic example: how long a system takes to respond can reveal what it did internally, even if the actual response content gives nothing away.

### The Analogy

Think of a locked safe with a mechanical combination dial. A skilled safecracker does not need to see inside the safe -- they can *listen* for the faint click as each correct digit falls into place. The sound is a side effect of the mechanism, not something the manufacturer intended to expose, but it leaks the secret anyway. Model side-channel attacks work the same way: the attacker is not reading the model's weights directly, they are listening to unintentional "clicks" -- timing, memory usage, power draw, cache behavior -- that correlate with what the model is doing internally.

### Where Timing Leaks Show Up in ML Systems

| Source of Timing Signal | What It Can Reveal |
|--------------------------|---------------------|
| **Inference latency vs. input size** | Model architecture complexity (deeper/larger models take measurably longer per token or per pixel) |
| **Latency vs. input content** | Early-exit behavior: some models/pipelines stop processing early on "easy" inputs (e.g., a content filter that short-circuits on an obvious match) |
| **First-token latency vs. total latency (LLMs)** | Whether a request hit a cache, whether retrieval (RAG) happened before generation, whether the request was routed to a smaller "router" model first |
| **Latency clustering across many queries** | Batch size and load-balancing behavior of the serving infrastructure -- useful for building a profile of the backend, not just the model |
| **CPU/GPU utilization patterns (if observable)** | Whether a request triggered a larger/more expensive model vs. a cheap fallback |

### Beyond Timing: Other Side Channels

- **Cache side channels**: if a system caches repeated prompts/responses, a cache hit is much faster than a cache miss -- this can reveal whether another user (or a system prompt) has already asked something similar.
- **Power/electromagnetic side channels**: relevant mainly for on-device/edge models (e.g., a model running on an IoT device or phone chip) where physical access is possible -- power draw patterns can leak which layers are executing.
- **Error-rate side channels**: some systems behave subtly differently (e.g., slightly different confidence rounding) depending on internal branching, which can leak which "expert" module handled the request in a mixture-of-experts model.

---

## 6. Technique 3: Artifact Inspection

Sometimes you get lucky and do not have to infer anything -- you get direct access to files. **Artifact inspection** means examining any physical byproduct of the model's development or deployment: a downloaded model file, a Docker container image, a mobile app package, a public GitHub repo, or leftover files on a misconfigured server.

### Common Artifacts and What They Leak

```
+---------------------------+   +----------------------------+   +---------------------------+
|  MODEL FILE FORMATS       |   |  CONTAINER / APP BUNDLES    |   |  METADATA & CONFIG FILES |
|  (.pt, .onnx, .h5,        |   |  (Docker images, .ipa/.apk) |   |  (config.json, README,    |
|   .safetensors, .pkl)     |   |                              |   |   requirements.txt)      |
|                            |   |                              |   |                           |
| - Layer names/shapes      |   | - Embedded model files       |   | - Exact model name/version|
| - Framework used           |   | - Serving framework version |   | - Training hyperparameters|
| - Sometimes full weights  |   | - Dependency versions (CVE   |   | - Preprocessing steps     |
| - Preprocessing hints      |   |   lookup!)                   |   | - Class label lists       |
| (input normalization      |   | - Hardcoded API keys/paths   |   | - Tokenizer vocab         |
|  constants baked in)      |   |                              |   |                           |
+---------------------------+   +----------------------------+   +---------------------------+
```

| Artifact Type | Tooling an Attacker Might Use | What It Reveals |
|----------------|-------------------------------|------------------|
| `.onnx` file | `onnx.load()`, Netron (graph visualizer) | Full computational graph, layer types, exact weight values |
| `.pt` / `.pth` (PyTorch) | `torch.load()`, `pickletools` | Model class structure, sometimes full weights and even training code references |
| `.h5` (Keras/TensorFlow) | `h5py`, TensorBoard | Layer configuration, weight shapes |
| Docker image | `docker history`, `dive`, extracting layers | Full filesystem, including model files, dependency versions, environment variables |
| Mobile app package | `apktool` (Android), reverse-engineering tools for iOS `.ipa` | Bundled on-device models (common for offline features), API endpoints, embedded secrets |
| Public model registry entry (e.g., Hugging Face Hub) | Just browsing | Model card, training data description, sometimes the exact base model being fine-tuned |

**Important nuance**: a `.pt` file is a **pickle**-based format. Loading an untrusted `.pt` file is not just a reconnaissance opportunity for the *attacker* studying someone else's model -- it is a well-known code execution risk for *anyone loading a malicious file themselves*. That specific danger is covered in depth in the "Vulnerable Framework Code" section of this module.

---

## 7. Worked Example -- Fingerprinting a Mystery Classifier

Suppose you are engaging a company's public-facing "AI-powered resume screener" -- an API that accepts a resume PDF and returns a JSON verdict of `{"fit_score": 0.83}`. You know nothing about the model behind it. Let's reverse-engineer it step by step.

### Step 1: Baseline Request

```
POST /api/screen
{ "resume": "<valid resume text>" }

Response:
{ "fit_score": 0.831442, "latency_ms": 412 }
```

Six decimal places on the score is a hint: many hand-rolled Flask/FastAPI wrappers pass raw `float` output straight to `json.dumps()` without rounding -- a polished commercial product usually rounds to 2-3 decimals. This suggests a relatively unpolished/internal system, not a heavily productized SaaS ML API.

### Step 2: Malformed Input

```
POST /api/screen
{ "resume": 12345 }     <-- wrong type, sent an integer instead of a string

Response (HTTP 500):
Traceback (most recent call last):
  File "/app/model_server.py", line 47, in screen
    tokens = tokenizer.encode(resume_text)
AttributeError: 'int' object has no attribute 'encode'
```

A raw Python traceback leaking through means: debug mode is on, the framework is likely Flask (from the file path style), and -- critically -- we now know the exact preprocessing function name (`tokenizer.encode`), implying a tokenizer-based (likely transformer) model rather than classic bag-of-words features.

### Step 3: Boundary Testing

```
Send a resume of increasing length: 100, 500, 2000, 8000, 32000 tokens.

Result: latency stays flat up to ~2000 tokens, then jumps sharply,
and requests above ~4096 tokens are silently truncated (fit_score
stops changing no matter how much more text is added past that point).
```

This strongly suggests a transformer-based model with a **4096-token context window** -- a signature consistent with several well-known mid-size open-weight language models used for classification via a "classification head" (a common transfer-learning pattern).

### Step 4: Timing Side Channel

```
Send two resumes of identical length:
  A) generic corporate boilerplate text
  B) text containing an exact keyword the company's job posting emphasizes

Result: (A) averages 410ms, (B) averages 610ms consistently across 50 trials.
```

The systematic latency difference suggests the pipeline does *more work* when certain keywords are present -- for example, a secondary retrieval step (looking up a skills database) that only triggers on keyword matches. This is a strong lead for a follow-on prompt/feature-based evasion attack: if the pipeline behaves differently around specific keywords, an attacker crafting an adversarial resume knows exactly which words to stuff or avoid.

### Putting It Together

From black-box querying alone, without ever seeing the source code or weights, we now have: framework (Flask, debug mode on), likely model family (transformer with classification head, 4096-token window), a preprocessing detail (`tokenizer.encode`), and a behavioral quirk (a keyword-triggered secondary lookup). That is enough intelligence to plan an evasion or extraction attack in the next phase of the engagement.

---

## 8. Access Models: Black-Box, Grey-Box, and White-Box

Every reverse engineering engagement starts by asking a simple question: **how much do I already know?** The answer determines which techniques from Sections 4-6 are even necessary, and it is standard vocabulary you will be expected to use precisely on the exam and in real engagement scoping.

```
              THE ACCESS SPECTRUM

  BLACK-BOX              GREY-BOX                WHITE-BOX
  +-----------+        +-----------+           +-----------+
  | Only see   |        | Some partial|          | Full source, |
  | inputs/     |        | knowledge:  |          | full weights, |
  | outputs     |        | framework    |          | full training |
  | via the     |        | name known,   |          | pipeline       |
  | public API  |        | error msgs    |          | visible        |
  |             |        | leak details, |          |               |
  |             |        | maybe a demo  |          |               |
  |             |        | account with  |          |               |
  |             |        | extra info    |          |               |
  +-----------+        +-----------+           +-----------+
   most realistic         common in real         rare outside of
   external-attacker       bug bounty /            insider threat /
   scenario                 pentest engagements     supply-chain scenarios
```

| Access Level | What You Have | What Sections 4-6 Techniques Buy You |
|---------------|------------------|------------------------------------------|
| **Black-box** | Just the public API/product, like any external customer | All the value -- API fingerprinting, timing analysis, and any leaked artifacts (mobile app bundles, public model cards) are your *only* path to intelligence |
| **Grey-box** | Partial internal knowledge -- maybe you know the vendor, or have a lower-privilege internal account, or found a public GitHub repo referencing the project | Confirms/narrows hypotheses fast -- e.g., you already suspect the framework, so fingerprinting just needs to confirm the exact version |
| **White-box** | Full access to source code, weights, and training pipeline (e.g., an internal red team engagement, or a security audit before shipping) | Reverse engineering techniques become almost unnecessary -- you would instead move directly to auditing the architecture and pipeline for weaknesses |

**Why this matters for scoping an engagement**: a client who asks you to "reverse engineer our production model" almost always means "simulate what a black-box external attacker could learn" -- which is why Sections 4-6 of this document deliberately avoid assuming any inside knowledge. If you find yourself reaching for "just read the training code" as your first move, you have accidentally shifted from a black-box assessment into a white-box code review, which is a different (and usually separately scoped) engagement.

---

## 9. Reverse Engineering Tooling Reference

You will not be expected to memorize exact command-line flags for the exam, but you should recognize the *category* of tool for each technique, since real engagements (and CTF-style HTB challenges) will hand you one of these.

| Category | Example Tools/Approaches | Use Case |
|----------|-----------------------------|----------|
| **HTTP/API probing** | Burp Suite, `curl`, custom Python scripts with `requests` | Sending baseline, malformed, and boundary-testing requests (Section 4) |
| **Model graph viewers** | Netron, `onnx.helper`, TensorBoard | Visualizing a model's computational graph once an artifact is obtained (Section 6) |
| **Binary/pickle inspection** | `pickletools`, `strings`, a hex editor | Inspecting `.pt`/`.pkl` files for embedded strings, class references, or suspicious `__reduce__` calls without fully executing them |
| **Container/image analysis** | `docker history`, `dive`, `skopeo` | Extracting layers and files from a serving container image without running it |
| **Mobile app analysis** | `apktool`, `jadx` (Android), class-dump-style tools (iOS) | Extracting bundled on-device models and embedded API endpoints/secrets |
| **Timing measurement** | Custom scripts with high-resolution timers, statistical repetition (dozens to hundreds of trials per data point) | Building a reliable timing side-channel signal despite network jitter (Section 5) |
| **Known-fingerprint databases** | Community-maintained lists mapping error signatures/response shapes to specific frameworks/versions | Correlating collected signals against known frameworks (Section 4) |

**A practical note on timing measurements**: network jitter is real noise that can drown out a genuine timing signal. Always take many repeated measurements (dozens to hundreds) for each data point and compare *distributions* (e.g., median and interquartile range), not single samples, before concluding that a timing difference is meaningful rather than coincidental network noise.

---

## 10. Security Angle

> **Security Angle**: Model reverse engineering is almost always the *first* step in a real attack chain against an AI application, not the final goal. A pentester or red teamer who skips this phase and jumps straight to "advanced" attacks (adversarial examples, prompt injection, extraction) is working blind and will waste queries and time. Conversely, defenders often invest heavily in protecting the model's weights while leaving the *behavioral* fingerprint completely exposed -- verbose error messages, unrounded confidence scores, and predictable timing are all free reconnaissance gifts to an attacker, even if the weights themselves are locked down tight. When you assess an AI application, always start by asking: "What would a determined attacker learn just by talking to this thing politely for an hour?"

---

## 11. Defensive Countermeasures

| Countermeasure | What It Stops |
|-----------------|----------------|
| **Generic, sanitized error messages** (no stack traces, no framework names) | API fingerprinting via error text |
| **Rounding/quantizing confidence scores** before returning them | Precision-based fingerprinting, some extraction/inversion techniques |
| **Constant-time or padded response latency** (add artificial delay to normalize timing) | Timing side channels |
| **Rate limiting and query budgets per client** | High-volume probing needed for systematic fingerprinting |
| **Stripping identifying headers** (`Server`, `X-Powered-By`, framework banners) | Header-based fingerprinting |
| **Signed/encrypted model artifacts, minimal metadata in shipped files** | Artifact inspection leaking architecture/training details |
| **Avoiding bundling full models in client-side apps** when server-side inference is feasible | Mobile app / client artifact extraction |
| **Monitoring for fingerprinting patterns** (systematic boundary-testing traffic, unusual timing-probe patterns) | Detecting reconnaissance in progress before the follow-on attack lands |

---

## 12. Key Takeaways

- **Model reverse engineering is reconnaissance**: inferring a model's architecture, framework, or behavior from the outside, without needing the source code or weights.
- It is **distinct from model stealing** -- reverse engineering produces *intelligence*, while stealing produces a *working clone*. Reverse engineering usually comes first.
- **API fingerprinting** exploits error messages, headers, response formatting, and boundary behavior to identify the underlying framework and model family.
- **Timing and side-channel analysis** exploits the fact that *how long* or *how much power* something takes can leak information even when the actual output reveals nothing -- the "safecracker listening for clicks" analogy.
- **Artifact inspection** looks at physical byproducts -- model files, container images, mobile app bundles, metadata -- which can leak everything from layer architecture to full weights.
- Always classify an engagement's **access model (black-box, grey-box, white-box)** before choosing techniques -- most real-world "reverse engineer our model" requests mean simulating a black-box external attacker.
- Defenders should treat **behavioral fingerprints** (errors, timing, formatting) as sensitive as the model weights themselves, since they are often the easier target.

---

*Next up: Denial of ML Service -- where we look at resource-exhaustion attacks unique to ML systems, including sponge examples, oversized inputs, and cost-blowup attacks against LLM agents.*
