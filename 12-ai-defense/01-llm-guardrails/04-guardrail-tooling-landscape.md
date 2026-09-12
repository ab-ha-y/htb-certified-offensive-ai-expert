# Guardrail Tooling Landscape

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Defense | Section: Guardrail Tooling Landscape

---

## Table of Contents

1. [Why Survey the Tooling Landscape?](#1-why-survey-the-tooling-landscape)
2. [Categories of Guardrail Tooling](#2-categories-of-guardrail-tooling)
3. [Policy/Orchestration Frameworks](#3-policyorchestration-frameworks)
4. [Structured-Output and Validation Libraries](#4-structured-output-and-validation-libraries)
5. [Standalone Safety/Moderation Classifiers](#5-standalone-safetymoderation-classifiers)
6. [Where Each Category Fits in the Three-Layer Stack](#6-where-each-category-fits-in-the-three-layer-stack)
7. [Build vs. Buy Considerations](#7-build-vs-buy-considerations)
8. [Worked Example -- Assembling a Guardrail Stack from Off-the-Shelf Pieces](#8-worked-example----assembling-a-guardrail-stack-from-off-the-shelf-pieces)
9. [Defense Angle](#9-defense-angle)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. Why Survey the Tooling Landscape?

You now understand the three conceptual layers of guardrails: character-based, content-based, and AI-based. In practice, teams building LLM applications rarely write every layer completely from scratch -- an entire ecosystem of open-source and commercial tools exists to implement pieces of this stack. As a defender (and as an offensive practitioner probing a target's defenses), you should be able to recognize *what category* of tool you are looking at, even without memorizing every specific product name, because new tools appear constantly while the underlying categories stay stable.

This file deliberately describes tool **categories** generically -- what kind of job each type of tool does, and which layer(s) of the stack it covers -- rather than treating any specific product as the only example. Specific product names are mentioned only as well-known, publicly documented illustrations of each category.

---

## 2. Categories of Guardrail Tooling

```
                     GUARDRAIL TOOLING TAXONOMY

   +----------------------------+  +----------------------------+
   |  POLICY / ORCHESTRATION    |  |  STRUCTURED-OUTPUT &       |
   |  FRAMEWORKS                |  |  VALIDATION LIBRARIES       |
   |                            |  |                              |
   |  "Define rules/flows that  |  |  "Force and verify that     |
   |   govern what the LLM can  |  |   LLM output matches a      |
   |   say/do, often combining  |  |   schema, and re-prompt or  |
   |   multiple layers"         |  |   fix it if it doesn't"     |
   +----------------------------+  +----------------------------+

   +----------------------------+
   |  STANDALONE SAFETY /       |
   |  MODERATION CLASSIFIERS     |
   |                              |
   |  "A model whose only job    |
   |   is content-based or       |
   |   AI-based judgment"        |
   +----------------------------+
```

---

## 3. Policy/Orchestration Frameworks

**What this category is (generic description)**: a configuration-driven framework that sits between the user and the LLM application, letting developers define **rails** (allowed/disallowed conversation flows, topics, and behaviors) using a mix of the techniques from the previous three files -- often combining a character/content check with a small "flow" model that decides what conversational move is happening, then enforces a policy about what can happen next.

**Illustrative example of this category**: frameworks in the spirit of **NeMo Guardrails**-style tooling. Generically, these frameworks let a developer:
- Define a set of allowed "canonical forms" of user intent and bot response (a kind of structured allowlist for conversation *flow*, not just individual messages).
- Attach input rails (checked before the main LLM sees the message), output rails (checked after generation), and sometimes "dialog rails" that constrain which conversational topics/paths are permitted at all.
- Plug in external fact-checking, jailbreak-detection, or moderation calls as one of the rail checks.

```
              POLICY/ORCHESTRATION FRAMEWORK -- CONCEPTUAL FLOW

   User message
       |
       v
   +-------------------+     +--------------------+     +-------------------+
   |  INPUT RAILS       |---->|  DIALOG POLICY     |---->|  MAIN LLM CALL    |
   |  (jailbreak check, |     |  (is this an       |     |  (only reached if |
   |   topic check,     |     |   allowed topic/   |     |   rails pass)     |
   |   moderation)       |     |   flow at all?)    |     |                   |
   +-------------------+     +--------------------+     +-------------------+
                                                                    |
                                                                    v
                                                          +-------------------+
                                                          |  OUTPUT RAILS      |
                                                          |  (fact-check,      |
                                                          |   moderation,       |
                                                          |   format check)    |
                                                          +-------------------+
```

**Key takeaway about this category**: it is less a *single* guardrail technique and more a **configuration/orchestration layer** that lets you wire together character-based, content-based, and AI-based checks (from the earlier three files) into a coherent, declaratively-defined policy, rather than hand-writing the plumbing yourself.

---

## 4. Structured-Output and Validation Libraries

**What this category is (generic description)**: a library that wraps LLM calls with a **schema** (a formal specification of exactly what shape the output should take -- e.g., "a JSON object with fields `name` (string) and `age` (integer 0-120)"), validates the model's actual output against that schema, and automatically retries, repairs, or rejects outputs that fail validation.

**Illustrative example of this category**: frameworks in the spirit of **Guardrails AI**-style tooling. Generically, these libraries let a developer:
- Define expected output structure and *validators* per field (e.g., "this field must not contain profanity," "this field must be a syntactically valid SQL SELECT statement and nothing else," "this field must not match any of our PII patterns").
- Automatically catch structural violations (the model returned malformed JSON, or added extra unrequested prose) as well as content violations (the field contains a policy-violating word).
- Optionally re-prompt the model automatically with the validation failure explained, giving it a chance to self-correct before the output ever reaches the application.

```
              STRUCTURED-OUTPUT VALIDATION -- CONCEPTUAL FLOW

   +-------------+      +-------------------+      +----------------------+
   |  LLM CALL   |----->|  SCHEMA + FIELD-   |----->|  Validation PASSES?  |
   |  (prompt +  |      |  LEVEL VALIDATORS  |      |                      |
   |   schema)   |      |  (regex, denylist, |      +----------------------+
   +-------------+      |   classifier, etc  |          |            |
                          |   -- reuses Layer  |         YES          NO
                          |   1/2 checks per   |          |            |
                          |   FIELD)           |          v            v
                          +-------------------+     Return to    Re-prompt model
                                                     application  with the failure
                                                                  explained, or
                                                                  hard-fail
```

**Key takeaway about this category**: this is where character-based and content-based validation (the first two files) get applied **per-field**, in a structured way, and combined with automatic self-correction -- particularly valuable for the LLM output attacks covered in Module 5 (SQL injection through generated queries, command injection via generated shell commands, etc.), because a schema can enforce "this output field must be a single, well-formed SQL SELECT statement with no semicolons" far more precisely than a general-purpose content filter could.

---

## 5. Standalone Safety/Moderation Classifiers

**What this category is (generic description)**: a model (sometimes a small fine-tuned classifier, sometimes a full LLM) whose entire job is to sit outside the main application flow and answer one question: "does this piece of text violate a defined safety policy, and if so, which category?" This is the productized form of the content-based and AI-based guardrails covered in the two prior files -- a ready-made model you call via an API instead of building and training your own from scratch.

**Illustrative examples of this category**:
- General-purpose **content moderation APIs** provided by major model vendors, which score text against categories like violence, self-harm, sexual content, and hate speech and return per-category confidence scores (a content-based validation pattern, productized).
- **Llama Guard-style classifiers**: openly published, purpose-built models fine-tuned specifically to classify a conversational turn (either the user's input or the model's output) as safe/unsafe against a defined taxonomy of risk categories, essentially packaging the "AI-based guardrail as judge" pattern from the previous file into a dedicated, smaller, cheaper-to-run model rather than requiring a full-size general LLM call.
- **Prompt-injection-specific detection classifiers**: models fine-tuned narrowly to distinguish "this text is attempting to inject/override instructions" from "this is normal text," which can be called on any text entering the context window (user input, retrieved documents, tool outputs) as a dedicated Layer 2/3 check.

```
              STANDALONE CLASSIFIER -- INTEGRATION PATTERN

   Any text that will enter the model's context window
   (user message, retrieved doc, tool output, model's own draft response)
                          |
                          v
   +--------------------------------------------------------------+
   |  Call to standalone safety/moderation/injection classifier   |
   |  (a separate, purpose-built, usually smaller model, reached  |
   |   via its own API call or locally-hosted inference)          |
   +--------------------------------------------------------------+
                          |
              +-----------+-----------+
              v                       v
          SAFE/ALLOW              UNSAFE/BLOCK
       (proceed to next          (reject, log, alert,
        step in pipeline)         or route to human review)
```

**Key takeaway about this category**: these tools let a team adopt content-based and AI-based guardrails **without training a classifier from scratch or hand-writing judge prompts** -- at the cost of depending on someone else's taxonomy of risk categories, someone else's training data, and someone else's update cadence for new attack patterns.

---

## 6. Where Each Category Fits in the Three-Layer Stack

| Tooling Category | Primary Layer(s) Covered | What It Adds Beyond the Raw Technique |
|--------------------|-----------------------------|------------------------------------------|
| **Policy/orchestration frameworks** (NeMo Guardrails-style) | All three, wired together, plus conversation-*flow* policy | Declarative configuration; lets non-ML engineers define rails without hand-coding every check; adds dialog-flow-level control beyond single-message checks |
| **Structured-output/validation libraries** (Guardrails AI-style) | Primarily Layer 1 (character-based) and Layer 2 (content-based), applied per output field | Schema enforcement; automatic re-prompting/self-correction; especially strong against Module 5's structured-output attacks (SQLi/command injection via LLM output) |
| **Standalone safety/moderation classifiers** (Llama Guard-style) | Layer 2 (content-based) and Layer 3 (AI-based), productized | Ready-made, pre-trained models -- no need to train your own classifier or write your own judge prompts from scratch |

---

## 7. Build vs. Buy Considerations

| Factor | Favors Building In-House | Favors Adopting Off-the-Shelf Tooling |
|--------|-----------------------------|------------------------------------------|
| **Domain specificity** | Your risk categories are unusual/niche (e.g., a very specific internal compliance policy) | Your risk categories are standard (general toxicity, common jailbreak patterns, PII) |
| **Update cadence** | You have a dedicated security/ML team who can react quickly to new attack patterns | You would rather rely on a vendor/open-source community's update cycle |
| **Latency/cost budget** | You can afford custom-optimized, lightweight in-house classifiers | You are fine with the latency/cost of an extra API call per request |
| **Auditability** | You need full visibility into exactly how a decision was made (e.g., regulatory requirement) | You are comfortable with a degree of "black box" in a well-reviewed open model |
| **Speed to deploy** | You have significant lead time | You need guardrails deployed quickly |

In practice, most production LLM applications use a **hybrid**: off-the-shelf standalone classifiers for general safety categories, combined with hand-written, domain-specific character- and content-based rules for anything unique to that application (e.g., a banking chatbot's specific denylist of internal system terms that should never be echoed back).

---

## 8. Worked Example -- Assembling a Guardrail Stack from Off-the-Shelf Pieces

Imagine you are building a customer-support chatbot for a healthcare company. Here is a realistic, layered stack built mostly from tooling categories described above:

```
   Incoming user message
        |
        v
   [Layer 1] Character-based denylist (hand-written, in-house):
             blocks literal known jailbreak phrases + company-specific
             internal terms that should never be discussed
        |  (passes)
        v
   [Layer 2] Standalone PII detection classifier (off-the-shelf):
             flags if the message contains what looks like another
             patient's medical record number
        |  (passes)
        v
   [Layer 2/3] Standalone safety/prompt-injection classifier
             (Llama-Guard-style, off-the-shelf): scores the message
             against a jailbreak/injection taxonomy
        |  (passes, score below threshold)
        v
   [Main LLM call], wrapped in a structured-output validation library:
             LLM must respond in a fixed JSON schema
             {"answer": string, "citation": string}, and the "answer"
             field is validated against the same PII classifier before
             being returned
        |
        v
   [Layer 3] AI-based judge (in-house LLM-as-a-judge call), reserved
             ONLY for conversations flagged as borderline by Layer 2,
             reasoning over the full conversation history before a
             final human-escalation decision
        |
        v
   Response returned to user
```

Notice how the layers from the previous three files map cleanly onto real tooling choices: hand-written rules for anything company-specific, off-the-shelf classifiers for general/standard risks, a schema-validation library wrapping the core LLM call, and a reserved, expensive AI-based judge only for the hardest, most ambiguous cases.

---

## 9. Defense Angle

**Which earlier-module attacks does this mitigate?**

- **Module 4 -- Prompt Injection and Jailbreaking**: policy/orchestration frameworks and standalone prompt-injection classifiers are purpose-built products for exactly this attack class, letting teams adopt a maintained, community- or vendor-updated defense rather than reinventing detection from scratch.
- **Module 5 -- SQL Injection / Command Injection via LLM Output, Function-Calling Attacks**: structured-output validation libraries are the most directly relevant tooling category here -- enforcing that an LLM's generated SQL, shell command, or tool-call arguments conform to a strict, safe schema is one of the most effective concrete mitigations against these attack types, because it constrains *what the output is allowed to look like* independent of how convincingly the model was tricked into generating it.
- **Module 5 -- Abuse Attacks**: standalone moderation/safety classifiers (Llama Guard-style) are the productized, ready-made answer to catching generated abuse content without building a custom classifier.
- **Important caveat for an offensive practitioner**: because these tools are widely deployed and often open-source or well-documented, attackers study them directly -- probing exactly which categories a known moderation classifier is weak on, or exactly what a known orchestration framework's default rail configuration misses, is a real and common reconnaissance step (covered further in the final file of this module on adaptive attacks). Knowing which category of tool a target likely uses (even without knowing the exact product) helps you reason about likely blind spots.

---

## 10. Key Takeaways

- The guardrail tooling ecosystem breaks down into three broad **categories**: policy/orchestration frameworks (NeMo Guardrails-style), structured-output validation libraries (Guardrails AI-style), and standalone safety/moderation classifiers (Llama Guard-style).
- **Policy/orchestration frameworks** wire together multiple layers and add conversation-*flow*-level control via declarative configuration.
- **Structured-output validation libraries** enforce a strict schema on LLM output, applying character-/content-based checks per field, and can automatically trigger self-correction -- especially valuable against Module 5's structured-output attacks.
- **Standalone safety/moderation classifiers** productize content-based and AI-based guardrails into ready-made, callable models, at the cost of depending on someone else's taxonomy and update cadence.
- Most real deployments use a **hybrid** of hand-written, domain-specific rules plus off-the-shelf tooling for standard risk categories.
- As an offensive practitioner, recognizing *which category* of tool a target is likely using helps you reason about probable blind spots, even without knowing the exact product in use.

*Next up: Model-Level Defenses -- moving beyond guardrails that wrap the model, into techniques that change the model itself to be inherently more robust, starting with adversarial training.*
