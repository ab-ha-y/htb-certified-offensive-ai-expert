# Jailbreaking

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Prompt Injection Attacks | Section: Jailbreaking

---

## Table of Contents

1. [What is Jailbreaking?](#1-what-is-jailbreaking)
2. [Jailbreaking vs. Prompt Injection](#2-jailbreaking-vs-prompt-injection)
3. [What "Alignment" Actually Means](#3-what-alignment-actually-means)
4. [Why Jailbreaks Exist At All](#4-why-jailbreaks-exist-at-all)
5. [The Jailbreak Technique Catalog](#5-the-jailbreak-technique-catalog)
6. [The Jailbreak Lifecycle](#6-the-jailbreak-lifecycle)
7. [Security Angle](#7-security-angle)
8. [Mitigations](#8-mitigations)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What is Jailbreaking?

The term "jailbreaking" is borrowed from the smartphone world, where it originally meant removing manufacturer-imposed software restrictions on a device (like an iPhone) so the owner could install unauthorized apps or gain root-level control the vendor did not intend to expose. Applied to AI, **jailbreaking** means the same basic idea: getting a model to step outside the behavioral boundaries its creators intentionally built into it through **safety training** and **alignment** (explained below), even though the model provider explicitly designed it not to.

### The Analogy

Think of a highly trained guard dog. The dog has been trained -- extensively, over a long period, with lots of positive and negative reinforcement -- not to attack visitors, not to eat food off the counter, and not to leave the yard. That training is deeply ingrained, but it is still *learned behavior*, not a physical impossibility. A sufficiently clever and patient person might discover that if they throw the food outside the yard's boundary line, the dog's "don't leave the yard" training and its "there's food, go get it" instinct come into conflict, and with the right setup, the food-seeking behavior wins. The dog didn't lose its training; the person just found a scenario the training didn't fully anticipate.

Jailbreaking an LLM is exactly this: finding scenarios, framings, or conversational paths that the model's safety training did not fully anticipate or generalize to, causing the model to produce output its designers tried to prevent.

### Formal Definition

> **Jailbreaking** is the practice of crafting inputs -- single prompts or multi-turn conversations -- specifically designed to circumvent an LLM's safety training and alignment, causing it to generate content, take actions, or express views that it was trained to refuse or avoid.

---

## 2. Jailbreaking vs. Prompt Injection

These two terms are related, overlap heavily in technique, and are frequently confused -- but they target slightly different things, which is worth being precise about for certification purposes.

| Aspect | Prompt Injection | Jailbreaking |
|--------|--------------------|-----------------|
| **What is being subverted?** | The **application's** instructions (system prompt, developer-defined task/role) | The **model's** own trained-in safety behavior (alignment/RLHF, refusal training) |
| **Where does the "rule" live?** | In the specific deployment (a particular chatbot's system prompt) | Baked into the model's weights itself, via the provider's training process |
| **Typical goal** | Make the model ignore its *task-specific* instructions (e.g. "only discuss orders") | Make the model ignore its *general safety* instructions (e.g. "don't help with X regardless of context") |
| **Does it require a "victim" system?** | Usually yes -- you're attacking a specific deployed application | No -- you can jailbreak a raw base model directly, with no wrapping application at all |
| **Overlap** | Direct/indirect injection techniques (override, persona, delimiter confusion, obfuscation) are frequently *the exact same techniques* used to jailbreak | Jailbreaks frequently *use* prompt injection techniques as their delivery mechanism |

**In practice**: many real-world "jailbreak prompts" you'll see shared online are really doing both at once -- they inject a persona (prompt injection technique) specifically in order to bypass safety alignment (jailbreaking goal). This module treats **prompt injection as the "how" (the mechanism/delivery)** and **jailbreaking as a specific category of "why"** (the goal being to defeat safety alignment specifically, as opposed to defeating a narrower application-level restriction).

---

## 3. What "Alignment" Actually Means

To understand what a jailbreak is bypassing, you need one more term: **alignment**. In plain English, alignment is the process of shaping a raw, freshly-trained language model's behavior to match human values and provider policies -- teaching it to be helpful, honest, and to refuse categories of requests deemed harmful (e.g. detailed instructions for violence, certain categories of illegal activity, generating child sexual abuse material, and similar).

This is typically done via:

- **RLHF (Reinforcement Learning from Human Feedback)**: human raters score model outputs as good/bad, and the model is further trained to produce more of the "good" outputs and fewer of the "bad" ones.
- **Safety fine-tuning / instruction tuning on refusal examples**: the model is explicitly trained on many examples of "here is a harmful request, here is the correct refusal response," so it learns to generalize a refusal pattern.
- **Constitutional/rule-based training approaches**: the model is trained against a written set of principles (a "constitution") describing desired behavior, sometimes with the model critiquing its own outputs against those principles during training.

```
                     BEFORE ALIGNMENT                    AFTER ALIGNMENT
                     (raw / base model)                   (safety-trained model)

    Request: "How do I           Might complete the       Refuses, explains why,
     make [harmful thing]?"       pattern statistically     may offer a safer
                                   without judgment,         alternative
                                   since it's just
                                   predicting plausible
                                   next tokens from
                                   internet-scale text

    JAILBREAKING = finding a path back toward "before alignment" behavior,
    without literally undoing the training -- by exploiting a framing,
    context, or conversational structure the alignment training didn't
    fully cover.
```

Because alignment is achieved through training on *examples* rather than through hard-coded, provably complete rules, it inherits the same generalization limits as any other machine-learned behavior (recall the [overfitting/underfitting](../../01-fundamentals-of-ai/01-introduction-to-machine-learning/introduction-to-machine-learning.md#3-key-terminology) concepts from Module 1) -- there will always be inputs outside the distribution the safety training covered well, and jailbreaks are the process of finding them.

---

## 4. Why Jailbreaks Exist At All

Three structural reasons jailbreaking is a persistent, unsolved problem rather than a one-time bug to patch:

1. **Alignment is trained, not proven.** There is no formal, mathematical guarantee that a safety-trained model will refuse every possible harmful input -- it is a statistical generalization from training examples, and statistical generalizations have edge cases.
2. **Capability and safety are in tension.** A model that is extremely good at following complex instructions, adopting personas, and reasoning about hypotheticals (all genuinely desirable capabilities) is, by the same token, better equipped to be *talked into* stepping outside its guidelines when those same capabilities are pointed at a jailbreak framing.
3. **The attack surface is the entire space of natural language.** Unlike a traditional software vulnerability with a finite, patchable code path, a jailbreak "vulnerability" is a region of an effectively infinite space of possible text -- patching one specific known jailbreak phrase does not close off the surrounding conceptual space that produced it.

---

## 5. The Jailbreak Technique Catalog

This section covers five major jailbreak technique families in dedicated files:

| # | Technique | File | Core Idea |
|---|-----------|------|-----------|
| 1 | **DAN-Style Persona Jailbreaks** | [dan-persona-jailbreaks.md](dan-persona-jailbreaks.md) | An elaborate, reinforced persona ("Do Anything Now") framed as free of restrictions. |
| 2 | **Hypothetical / Fictional Framing** | [hypothetical-fictional-framing.md](hypothetical-fictional-framing.md) | Wrapping the request in "just hypothetically," "for a story," or "in a simulation" framing. |
| 3 | **Multi-Turn Escalation (Crescendo)** | [multi-turn-escalation-crescendo.md](multi-turn-escalation-crescendo.md) | Gradually walking the conversation from benign to restricted content across many turns. |
| 4 | **Token Smuggling** | [token-smuggling.md](token-smuggling.md) | Splitting, encoding, or disguising restricted terms at the token level so filters and safety training don't fire on them. |
| 5 | **Many-Shot Jailbreaking** | [many-shot-jailbreaking.md](many-shot-jailbreaking.md) | Flooding the context with many examples of "compliant" behavior to bias the model toward continuing the pattern. |

```
                     JAILBREAK TECHNIQUE FAMILY TREE

   +--------------------------------------------------------------+
   |                       JAILBREAKING                             |
   +--------------------------------------------------------------+
        |                |                 |               |
        v                v                 v               v
   +----------+   +--------------+   +------------+   +-----------+
   | PERSONA/  |   | FRAMING      |   | STRUCTURAL |   | STATISTICAL|
   | ROLE-PLAY |   | (Hypothetical|   | (Token     |   | (Many-shot,|
   | (DAN)     |   |  / Fiction)  |   |  smuggling) |   | context    |
   |           |   |              |   |            |   | flooding)  |
   +----------+   +--------------+   +------------+   +-----------+
        |                                                    ^
        +----------------------------------------------------+
              MULTI-TURN ESCALATION (Crescendo) often layers
              ON TOP of any of the above, spreading the attack
              across several conversational turns rather than
              attempting it in one message.
```

---

## 6. The Jailbreak Lifecycle

Real jailbreak development (in the security research community) tends to follow a predictable lifecycle, useful to know for exam purposes:

```
   1. DISCOVERY            2. SHARING              3. PATCHING            4. MUTATION
   -------------            ----------              -----------            -----------
   A researcher or           The prompt spreads       The provider          The community
   hobbyist finds a           on forums/social         updates safety        tweaks the
   prompt that reliably       media (historically,     training or adds     original prompt
   bypasses safety            e.g. early "DAN"          input/output          slightly, finds
   training on a               prompts spread            filters              a variant that
   specific model.             this way).                targeting the        still works, and
                                                          known phrasing.       the cycle repeats.
        |                          |                          |                     |
        v                          v                          v                     v
   +-----------+           +-------------+           +--------------+       +-------------+
   | "It works!"|--------->| Goes viral   |--------->| Model version |------>| New variant  |
   +-----------+           +-------------+           | patches it    |       | discovered   |
                                                       +--------------+       +-------------+
```

This is directly analogous to the classic **vulnerability disclosure and patch/bypass cycle** in traditional security -- a useful framing for explaining jailbreaking to people with a conventional pentest background.

---

## 7. Security Angle

- Jailbreak testing is a core deliverable in **AI red-teaming engagements**: clients want to know not just "can a user misuse my chatbot's business logic" (prompt injection against the app) but "can a user get the underlying model to say something the model provider itself explicitly tried to prevent" (jailbreaking the model itself) -- a distinct, reportable risk category.
- A mature jailbreak-testing methodology tests **each technique family independently**, and then tests **combinations** (e.g. a DAN persona wrapped in hypothetical framing, escalated across multiple turns) since real-world successful jailbreaks are very often composites.
- Because alignment training differs across model providers and even across versions of the same model family, jailbreak susceptibility is **not static** -- a technique that fails against one model/version may succeed against another, and vice versa. Continuous re-testing is part of the job.

---

## 8. Mitigations

Full defense-in-depth is covered in [Mitigations](../04-mitigations/mitigations.md). Jailbreak-specific notes:

- **Continuous red-teaming and adversarial training**: providers actively hunt for jailbreaks and retrain models on the discovered examples -- this is why yesterday's viral jailbreak prompt often stops working today.
- **Layered safety** (model-level alignment + application-level input/output filtering + monitoring) means a jailbreak that defeats the model's internal training might still be caught by an external filter, and vice versa.
- **Constitutional/self-critique approaches**: having the model (or a second model) evaluate its own draft response against safety principles before finalizing it, catching jailbreak successes after the fact.
- **Rate limiting and behavioral monitoring**: multi-turn escalation and many-shot jailbreaks in particular are detectable through conversation-level pattern analysis, not just single-message analysis.

---

## 9. Key Takeaways

- Jailbreaking targets a model's **trained-in safety/alignment behavior**, whereas prompt injection (in the narrower sense) targets an **application's task-specific instructions** -- though the two overlap heavily in technique.
- Alignment (via RLHF, safety fine-tuning, or constitutional methods) is a *statistically learned* behavior, not a formally proven guarantee -- which is exactly why jailbreaks, as edge cases the training didn't fully cover, are possible in principle.
- Jailbreaks persist because capability and safety are in tension, and because the "vulnerable surface" is the entire space of natural language, not a finite, patchable codebase.
- The five major technique families -- DAN-style personas, hypothetical/fictional framing, multi-turn escalation, token smuggling, and many-shot jailbreaking -- are frequently combined rather than used in isolation.
- Jailbreak research follows a discovery -> sharing -> patching -> mutation lifecycle strongly analogous to traditional vulnerability disclosure.

---

*Next up: DAN-Style Persona Jailbreaks -- the most famous productized jailbreak technique, and how it builds on the role-play mechanism covered earlier in this module.*
