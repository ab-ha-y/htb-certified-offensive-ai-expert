# Denial of ML Service

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Attacking AI - Application and System | Section: Denial of ML Service

---

## Table of Contents

1. [What Is Denial of ML Service?](#1-what-is-denial-of-ml-service)
2. [The ML Cost Stack -- Where Compute Actually Goes](#2-the-ml-cost-stack----where-compute-actually-goes)
3. [Attack 1: Sponge Examples](#3-attack-1-sponge-examples)
4. [Attack 2: Oversized and Malformed Inputs](#4-attack-2-oversized-and-malformed-inputs)
5. [Attack 3: Recursive and Expensive Prompts Against LLMs](#5-attack-3-recursive-and-expensive-prompts-against-llms)
6. [Attack 4: Model-Loop-Induced Cost Blowup](#6-attack-4-model-loop-induced-cost-blowup)
7. [Worked Example -- Sponge-ifying an Image Classifier](#7-worked-example----sponge-ifying-an-image-classifier)
8. [Denial of ML Service vs. Traditional DoS](#8-denial-of-ml-service-vs-traditional-dos)
9. [Measuring and Budgeting Inference Cost](#9-measuring-and-budgeting-inference-cost)
10. [Security Angle](#10-security-angle)
11. [Defensive Countermeasures](#11-defensive-countermeasures)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. What Is Denial of ML Service?

You already know classic **Denial of Service (DoS)**: flooding a web server with more requests than it can handle until legitimate users can no longer get through. **Denial of ML Service (DoML, sometimes called "ML-DoS")** is the same *goal* -- make the service unavailable or unusably slow/expensive -- but achieved through a completely different *mechanism*: exploiting the fact that ML inference cost is **not fixed per request**. Some inputs are dramatically more expensive to process than others, and an attacker who knows how to craft those expensive inputs can exhaust a system's compute budget with far fewer requests than a traditional flood would need.

### The Analogy

Think of a restaurant kitchen that prices every dish the same on the menu, regardless of how long it actually takes to cook. Most customers order a burger (5 minutes). One customer, who has figured out the kitchen's weak spot, orders "the everything platter, extra rare, hold everything, remake it three times, and also here's a 40-page list of substitutions" -- a dish that is technically on the menu but takes two hours and ties up every burner in the kitchen. A handful of these "sponge orders" can shut down the whole kitchen for everyone else, without the attacker needing to send thousands of separate orders.

Denial of ML Service works the same way: instead of overwhelming the system with *volume*, the attacker overwhelms it with *unit cost*. A single cleverly crafted input can do the damage of a thousand normal ones.

### Formal Definition

**Denial of ML Service** refers to a class of resource-exhaustion attacks that exploit the variable, input-dependent computational cost of ML inference (and, for LLMs, generation) to degrade availability, inflate latency, or blow up the monetary cost of running a model -- often using a small number of carefully crafted requests rather than brute-force traffic volume.

---

## 2. The ML Cost Stack -- Where Compute Actually Goes

To understand why some inputs are so much more expensive than others, you need to know where an ML system actually spends its time and money.

```
                     WHERE INFERENCE COST COMES FROM

  +----------------+     +------------------+     +-------------------+     +----------------+
  |  INPUT SIZE /   |     |  MODEL COMPUTE    |     |  GENERATION        |     |  DOWNSTREAM    |
  |  COMPLEXITY     |---->|  PER INPUT        |---->|  LENGTH (LLMs)     |---->|  ACTIONS       |
  |                 |     |                    |     |                    |     |  (tool calls,  |
  | - image size    |     | - # layers/params  |     | - tokens generated |     |  retries, DB   |
  | - sequence      |     | - # "active" paths |     |   before stopping  |     |  queries,      |
  |   length        |     |   (early exit,     |     | - loops/self-talk  |     |  other API     |
  | - batch size    |     |   dynamic depth,   |     |   before a final   |     |  calls)        |
  |                 |     |   mixture-of-      |     |   answer           |     |                |
  |                 |     |   experts routing) |     |                    |     |                |
  +----------------+     +------------------+     +-------------------+     +----------------+

  ATTACKER LEVERAGE POINT:  ^^^^^^^^^^^^^^^^^^^^   ^^^^^^^^^^^^^^^^^^^^^   ^^^^^^^^^^^^^^^^^^^^
                             craft input to force    craft input to force   craft input to trigger
                             max cost per unit        max computation        many/expensive
                             of input                 per token generated    downstream calls
```

Key insight: **for classical, fixed-depth neural networks, cost is roughly proportional to input size**. But for **LLMs and "adaptive-depth" models** (models with early-exit branches, mixture-of-experts routing, or autoregressive generation), the *same-sized* input can trigger wildly different amounts of computation depending on its *content*, not just its size. That gap between "size" and "actual cost" is exactly what these attacks exploit.

---

## 3. Attack 1: Sponge Examples

A **sponge example** is an adversarial input -- crafted the same way you'd craft an adversarial example to fool a classifier -- except instead of trying to change the model's *prediction*, it is optimized to maximize the model's **energy consumption and/or latency** while keeping the input looking innocuous (same size, same format as a normal input).

### Why "Sponge"?

Because the input is engineered to "soak up" far more compute than a normal input of the same size -- like a sponge absorbing water. The term comes from published adversarial ML research on attacks that specifically target the *efficiency* mechanisms modern models rely on (like early exits or sparsity) rather than their accuracy.

### How Sponge Examples Are Built

```
 Normal adversarial example:          Sponge example:
   optimize input to change            optimize input to maximize
   the OUTPUT LABEL                    LATENCY / ENERGY / FLOPs
   (evasion)                           (denial of service)

 +-----------+   gradient    +-----------+   gradient    +-----------+
 |  Input    |  ascent on    |  Adv.     |  ascent on    | "Sponge"  |
 |  (image,  |  loss for a   |  Example  |  latency/     | Input     |
 |  text)    |  wrong class  |           |  energy proxy |           |
 +-----------+               +-----------+               +-----------+
```

Attackers who have white-box or even black-box access to a model can iteratively perturb an input (adding tiny changes) while measuring latency/energy as feedback, gradually pushing the input toward a version that defeats the model's efficiency shortcuts -- for example, disabling an early-exit branch that most inputs trigger, forcing the model to run its full, most expensive computation path every single time.

### Where Sponge Examples Are Most Effective

| Target Property | Why Sponge Examples Exploit It |
|-------------------|----------------------------------|
| **Early-exit networks** | Designed to stop computing once "confident enough" -- a sponge input is crafted to *never* look confident, forcing full-depth computation every time |
| **Mixture-of-experts (MoE) models** | Normally only a handful of "expert" sub-networks are activated per input -- a sponge input is crafted to activate as many experts as possible |
| **Autoregressive generation (LLMs)** | Crafted to avoid natural stopping points, maximizing the number of tokens generated before an end-of-sequence condition is reached |
| **Batch-processing pipelines** | A single sponge input in a batch can slow down the entire batch, since batches typically wait for the slowest member to finish |

---

## 4. Attack 2: Oversized and Malformed Inputs

This is the more "brute force" cousin of sponge examples: instead of subtle optimization, the attacker simply exploits **missing or weak input validation** to send inputs far larger, deeper, or more structurally complex than the system was designed to expect.

### Common Patterns

| Input Type | Oversized/Malformed Attack |
|------------|------------------------------|
| **Images** | Extremely high resolution images, or deeply nested/decompression-bomb-style image files (a small file that decompresses into a massive one) |
| **Text/documents** | Multi-megabyte text blobs, deeply nested JSON, PDFs with millions of characters, extremely long lines with no whitespace (breaking tokenizers that assume reasonable word lengths) |
| **Tabular data** | Extremely wide feature vectors, deeply nested categorical encodings that explode during preprocessing (e.g., one-hot encoding of an unexpectedly huge category space) |
| **Audio/video** | Very long duration files that must be fully decoded and chunked before inference even starts |
| **Compressed archives (zip bombs)** | Files that decompress to gigabytes/terabytes, exhausting memory during a preprocessing step that "helpfully" unzips uploads automatically |

### Why ML Pipelines Are Especially Vulnerable to This

Traditional web applications usually have mature, battle-tested input size limits (max upload size, max request body size). ML preprocessing pipelines, by contrast, are often glued together from research code (written by data scientists, not security engineers) that assumes "reasonable" inputs and does not enforce hard limits before expensive operations like resizing, tokenizing, or decompressing.

```
        THE MISSING-LIMIT GAP

  Web framework layer         ML preprocessing layer         Model layer
  +------------------+       +------------------------+     +------------+
  | max body size:    |       | resize image, tokenize |     | fixed-size |
  | 10 MB (enforced)   |------>| text, decode audio     |---->| tensor in, |
  |                    |       | (often NO limit here!) |     | fixed cost |
  +------------------+       +------------------------+     +------------+
                                        ^
                                        |
                          Attacker's oversized/malformed
                          input passes the outer gate fine,
                          but explodes here.
```

---

## 5. Attack 3: Recursive and Expensive Prompts Against LLMs

For LLM-backed applications, the "input" is a prompt, and prompt *content* -- not just length -- can dramatically change how much work the model (and the surrounding application) does. This is a DoML pattern unique to generative AI.

### Techniques

| Technique | Mechanism |
|-----------|-----------|
| **"Keep going forever" prompts** | Instructing the model to produce an extremely long, open-ended response ("write an infinitely detailed story, never stop, keep expanding on every detail") -- maximizes tokens generated, which is directly billed and directly slow |
| **Self-referential/recursive prompts** | Asking the model to repeatedly summarize its own previous output, then expand it, then summarize again, in a loop -- especially damaging in agent frameworks that let the model call itself or a sub-agent repeatedly |
| **Forcing maximum "reasoning" effort** | For models that support extended chain-of-thought / reasoning modes, crafting prompts that trigger the longest possible reasoning traces before an answer is produced |
| **Repeated retrieval triggers (RAG systems)** | Prompts crafted to force expensive retrieval-augmented-generation lookups on every turn (e.g., asking many independent questions in one message, each needing its own vector search) |
| **Adversarial "unanswerable" prompts** | Prompts designed so the model never reaches a natural stopping condition, causing it to hit (and pay for) the maximum token limit on every single call |

### Cost Amplification in Multi-Step Pipelines

Many LLM applications do not make just one model call per user request -- they chain several (a "router" call, a retrieval call, a generation call, a safety-check call). A single malicious prompt that inflates the token count or triggers extra branches at *each* stage multiplies the damage.

```
  User Prompt
       |
       v
  +-----------+     +-------------+     +--------------+     +--------------+
  | Router LLM|---->| Retrieval   |---->| Generation   |---->| Safety-check |
  | call      |     | (vector DB  |     | LLM call     |     | LLM call     |
  |           |     | search)     |     |              |     |              |
  +-----------+     +-------------+     +--------------+     +--------------+
     cost x1            cost x1             cost x1              cost x1

  A prompt engineered to maximize cost at EVERY stage multiplies total
  cost far beyond what a single "big prompt" metric would suggest.
```

---

## 6. Attack 4: Model-Loop-Induced Cost Blowup

This is the agentic version of Denial of ML Service, and arguably the most dangerous because it can spiral **without the attacker sending any more requests after the first one**. Modern AI applications increasingly use **agents**: an LLM that can reason in a loop, call tools, look at the results, and decide whether to call more tools or produce a final answer.

### The Mechanism

If an agent's loop-termination logic is weak (no hard cap on iterations, no cost budget, no loop-detection), a single crafted input can send the agent into a self-perpetuating cycle of tool calls, sub-agent spawns, or retries -- each one costing real compute/money -- until an operator manually intervenes or a hosting bill arrives.

```
                       RUNAWAY AGENT LOOP

    +------------------+
    |  User sends ONE   |
    |  crafted request   |
    +------------------+
              |
              v
    +------------------+
    |  Agent reasons:    |<-------------------------------+
    |  "I should check   |                                 |
    |  X to answer this" |                                 |
    +------------------+                                 |
              |                                             |
              v                                             |
    +------------------+                                   |
    |  Tool call        |                                   |
    |  (search, sub-     |                                   |
    |  agent, retry)     |                                   |
    +------------------+                                   |
              |                                             |
              v                                             |
    +------------------+     No hard stop /                |
    |  Result looks      |     no budget cap /---------------+
    |  "inconclusive" -   |     no loop detection
    |  agent decides to
    |  try again
    +------------------+
```

### Real-World-Shaped Causes

| Cause | Example |
|-------|---------|
| **Missing iteration caps** | Agent framework has no `max_iterations` setting, or it is set unrealistically high |
| **Ambiguous or contradictory instructions** | A crafted prompt tells the agent its task is "not yet complete" no matter what evidence it gathers, so it keeps retrying |
| **Sub-agent spawning without limits** | A "manager" agent can spawn "worker" agents to delegate tasks, and a crafted task description causes recursive spawning (worker spawns more workers) |
| **Tool errors misinterpreted as "try harder" signals** | A tool call fails or times out, and the agent's default behavior on failure is to immediately retry rather than fail gracefully |
| **Feedback loops between multiple agents** | Agent A's output becomes Agent B's input and vice versa, and a crafted message causes them to keep responding to each other indefinitely |

**This is directly related to "Rogue Actions" and "Excessive Agency"**, covered later in this module -- a cost-blowup loop is often the *side effect* of the same underlying design flaw (too much autonomy, too few guardrails) that causes an agent to take unintended harmful actions.

---

## 7. Worked Example -- Sponge-ifying an Image Classifier

Suppose a company exposes an image moderation API: it accepts an uploaded image and classifies it as "safe" or "flagged," using an early-exit convolutional neural network (CNN) that returns a fast answer for clearly obvious images and only runs its full depth for ambiguous ones.

### Step 1: Baseline Timing

```
Send 100 "obviously safe" images (e.g., plain white background).
Average latency: 40ms  --> early-exit branch triggered almost every time.

Send 100 "ambiguous" images (partially obscured, low contrast).
Average latency: 340ms --> full-depth computation triggered.
```

We now know the model has an early-exit shortcut, and we know roughly the cost ratio (about 8.5x) between the cheap and expensive path.

### Step 2: Optimize Toward the Expensive Path

Using access to a similar open-source pretrained model (a common substitute-model technique -- see Module 1's "model stealing" content), the attacker uses gradient-based optimization to find small pixel perturbations that push *any* base image toward triggering the full-depth path, while keeping the image visually unchanged to a human reviewer and still classified the same way.

```
 Base image (plain, cheap: 40ms)
         |
         |  add small, targeted perturbation
         |  (optimized against a substitute model's
         |   confidence/early-exit proxy)
         v
 "Sponge" image (still looks plain, now: 320ms)
```

### Step 3: Scale the Attack

```
Normal flood needed to cause meaningful load: ~10,000 req/sec of normal images.
Sponge flood needed for the same load:         ~1,200 req/sec of sponge images.

Attacker achieves the same infrastructure strain with ~88% fewer requests,
using far less bandwidth and far fewer source IPs -- making it much harder
to detect with traditional rate-based DoS defenses.
```

### The Takeaway From This Example

The attacker never needed to know the model's exact weights. They needed only: (1) black-box timing measurements to confirm an early-exit mechanism exists, and (2) a *substitute* model to optimize sponge inputs offline, transferring the attack to the real target -- the same transferability property that makes adversarial examples dangerous in evasion attacks generally.

---

## 8. Denial of ML Service vs. Traditional DoS

It is worth pinning down exactly how this attack class relates to -- and differs from -- the DoS/DDoS concepts you likely already know from traditional network and application security.

| Dimension | Traditional (Volumetric) DoS | Denial of ML Service |
|-----------|-----------------------------------|----------------------|
| **Primary lever** | Request/packet *volume* | Per-request *cost* |
| **Typical scale needed** | Thousands to millions of requests, often from many source IPs (botnets) | Can be as few as a handful to a few hundred carefully crafted requests |
| **Detection signal** | Spike in request/connection count, bandwidth | Spike in *average latency or compute cost per request*, often with normal or even low request volume |
| **Traditional defenses' effectiveness** | Rate limiting, WAF rules, CDN absorption -- generally effective | Often ineffective, since traffic volume and shape look completely normal |
| **Cost to the attacker** | Requires significant infrastructure (botnet, bandwidth) | Often requires only modest compute to craft a small number of expensive inputs offline |
| **Cost to the victim** | Bandwidth/connection exhaustion | Compute exhaustion, GPU/CPU cost, and -- for pay-per-token cloud LLM APIs -- direct monetary billing overrun |

**The single most important exam-relevant distinction**: Denial of ML Service can succeed *even when every traditional DoS metric looks completely healthy*. A system can report low request-per-second counts, no unusual source IPs, and no protocol anomalies, while a handful of sponge examples or runaway agent loops are quietly consuming its entire compute budget. This is why Denial of ML Service needs its own detection strategy (Section 9) rather than being treated as "just another flavor of DoS" that existing tooling already covers.

---

## 9. Measuring and Budgeting Inference Cost

Because the attacks in this section exploit *cost*, not volume, effective defense requires actually measuring cost in the first place -- something many ML systems never instrument, since traditional application monitoring (requests/sec, error rate, p50/p99 latency) does not automatically surface it.

### What to Measure

```
                 THE ML COST OBSERVABILITY STACK

  +-------------------+   +-------------------+   +-------------------+
  |  PER-REQUEST        |   |  AGGREGATE          |   |  ANOMALY            |
  |  COST METRICS        |   |  BUDGET TRACKING     |   |  DETECTION           |
  |                     |   |                     |   |                     |
  | - tokens generated   |   | - $ spent per hour/  |   | - outlier detection  |
  |   (LLMs)              |   |   day against a       |   |   on cost-per-       |
  | - GFLOPs / compute-    |   |   defined budget       |   |   request            |
  |   time per inference   |   | - GPU-hours consumed  |   |   distribution        |
  | - # tool calls /        |   |   per client/tenant     |   | - flag sudden spikes |
  |   agent iterations      |   | - cost attributed to    |   |   in average cost     |
  |   per request            |   |   specific API keys/    |   |   even at constant    |
  | - bytes processed        |   |   users                |   |   request volume      |
  |   (image/audio/video      |   |                     |   |                     |
  |   size after decode)      |   |                     |   |                     |
  +-------------------+   +-------------------+   +-------------------+
```

### A Simple Cost-Budgeting Policy Table

| Policy Lever | Example Threshold | What It Catches |
|----------------|----------------------|--------------------|
| **Max tokens generated per LLM call** | Hard cap (e.g., 2048 tokens) regardless of what the prompt requests | Recursive/"keep going forever" prompts (Section 5) |
| **Max agent loop iterations** | Hard cap (e.g., 10 tool calls per user request) | Model-loop-induced cost blowup (Section 6) |
| **Max input size/dimensions before preprocessing** | Hard cap enforced before decode/resize (e.g., reject images above 4K resolution) | Oversized/malformed inputs (Section 4) |
| **Per-client/per-API-key cost quota** | Dollar or compute-unit ceiling per hour, with automatic throttling past it | Any of the above, regardless of technique, since it caps total exposure per identity |
| **Latency-based circuit breaker** | If p95 latency exceeds N standard deviations from a rolling baseline, throttle or alert | Sponge examples degrading performance gradually across many requests |

**Key insight for defenders**: you cannot defend against a cost that you do not measure. Instrumenting per-request cost (not just per-request success/failure) is the prerequisite for every other defense in Section 11 -- without it, a sponge example or a runaway loop is invisible until the infrastructure bill or the outage makes it undeniable.

---

## 10. Security Angle

> **Security Angle**: Traditional DoS defenses -- rate limiting by IP, request-volume thresholds, connection limits -- are built around the assumption that **cost per request is roughly constant**. Denial of ML Service attacks break that assumption, which means a system can be "well protected" by every traditional metric (low request volume, no unusual IPs, no protocol anomalies) while still being brought to its knees by a handful of expensive inputs. When assessing an AI application's resilience, always ask: "What is the *most expensive* input this system will accept, and how far is that from the *typical* input's cost?" A large gap between typical cost and worst-case cost is a Denial of ML Service vulnerability waiting to be exploited -- and it is a question most application security checklists never ask, because it did not exist before ML made per-request cost variable.

---

## 11. Defensive Countermeasures

| Countermeasure | What It Stops |
|-----------------|----------------|
| **Hard input size/shape limits enforced before preprocessing** (not just at the network layer) | Oversized/malformed input attacks |
| **Timeouts on preprocessing steps** (decompression, decoding, tokenization) | Decompression bombs, pathological files |
| **Per-request compute/cost budgets, not just request counts** | Sponge examples and recursive prompts (caps cost regardless of request volume) |
| **Hard caps on LLM output length and agent loop iterations** | Recursive prompts, model-loop-induced cost blowup |
| **Loop and cycle detection in agent frameworks** (detect repeated tool calls with similar arguments) | Runaway agent loops |
| **Monitoring latency/energy distributions, not just request counts** | Detecting sponge examples in production (a spike in *average cost per request* is the signal, not a spike in request volume) |
| **Disabling or capping early-exit/adaptive-depth optimizations under adversarial-looking traffic** | Reduces the ceiling a sponge example can exploit |
| **Circuit breakers between chained model/tool calls** | Stops cascading cost amplification across multi-step pipelines |

---

## 12. Key Takeaways

- **Denial of ML Service exploits variable, input-dependent inference cost** rather than raw request volume -- the ML-specific twist on classic DoS.
- **Sponge examples** are adversarial inputs optimized to maximize latency/energy consumption (defeating early exits, sparsity, or mixture-of-experts shortcuts) while looking like normal inputs.
- **Oversized/malformed inputs** exploit weak validation in ML preprocessing pipelines, which are often less hardened than the web layer sitting in front of them.
- **Recursive/expensive LLM prompts** exploit the fact that prompt *content*, not just length, controls how many tokens get generated and how many downstream calls get triggered.
- **Model-loop-induced cost blowup** is the agentic escalation of this attack class: a single crafted input can cause an agent to spiral into unbounded, self-perpetuating tool calls or sub-agent spawns.
- Defenders must budget and monitor **cost per request**, not just request volume, and must place hard limits (size, iteration count, timeouts) at every stage of an ML/agent pipeline, not just at the network perimeter.

---

*Next up: Insecure Integrated Components -- where we look at the risks introduced by third-party libraries, plugins, and tools wired into an AI application's supply chain.*
