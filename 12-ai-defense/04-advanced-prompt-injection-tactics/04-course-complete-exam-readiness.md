# Course Complete -- Exam Readiness

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Defense | Section: Course Complete -- Exam Readiness

---

## Table of Contents

1. [You've Reached the End of the Written Notes](#1-youve-reached-the-end-of-the-written-notes)
2. [The Full Journey, in One Picture](#2-the-full-journey-in-one-picture)
3. [Module-by-Module Recap](#3-module-by-module-recap)
4. [The Two Core Mental Models to Carry Forward](#4-the-two-core-mental-models-to-carry-forward)
5. [Self-Check: Can You Explain These Without Looking?](#5-self-check-can-you-explain-these-without-looking)
6. [How to Use These Notes for Exam Prep From Here](#6-how-to-use-these-notes-for-exam-prep-from-here)
7. [Final Word](#7-final-word)

---

## 1. You've Reached the End of the Written Notes

This file closes out all 12 modules of the HTB Certified Offensive AI Expert (COAE) study guide, from your very first introduction to what machine learning even is, all the way through to actively probing an AI system's defenses and understanding how to defend against that probing in turn. If you started this course new to AI/ML, as this guide was written assuming, take a moment to notice how much ground has actually been covered -- the topics in this final module would have been unreadable jargon without everything built up before it.

---

## 2. The Full Journey, in One Picture

```
   FOUNDATIONS                    APPLIED SKILLS               RED-TEAMING FRAMEWORKS
   (Module 1)                     (Module 2)                    (Module 3)
   +----------------+             +----------------+            +----------------+
   | ML/DL/GenAI      |            | sklearn/PyTorch  |           | OWASP Top 10s,   |
   | basics, the       |----------->| hands-on projects |---------->| SAIF, attack      |
   | vocabulary you    |            | (spam, malware,   |           | surface by         |
   | need for           |            | anomaly detection)|          | component           |
   | everything else    |            +----------------+            +----------------+
   +----------------+                                                        |
                                                                              v
   ATTACKING THE LLM ITSELF          ATTACKING TRAINING DATA        ATTACKING APPS/SYSTEMS
   (Modules 4-5)                     (Module 6)                     (Module 7)
   +----------------+                +----------------+             +----------------+
   | Prompt injection, |             | Poisoning,        |           | Reverse            |
   | jailbreaking,      |------------>| backdoors, tensor  |---------->| engineering,        |
   | LLM output attacks |             | steganography,     |           | denial of service,   |
   | (XSS/SQLi/exfil)   |             | pickle exploits     |           | rogue agents, MCP     |
   +----------------+                +----------------+             +----------------+
             |
             v
   EVADING CLASSIFIERS                                              PRIVACY ATTACKS
   (Modules 8-10)                                                    (Module 11)
   +----------------------------------------------------+           +----------------+
   | Foundations -> Dense (FGSM/DeepFool) -> Sparse         |-------->| Membership        |
   | (JSMA/EAD/single-pixel) -- three escalating levels      |         | inference,          |
   | of mathematical sophistication for fooling a model      |         | differential        |
   +----------------------------------------------------+           | privacy, DP-SGD,     |
                                                                       | PATE                  |
                                                                       +----------------+
                                                                                |
                                                                                v
                                                              DEFENSE (Module 12 -- YOU ARE HERE)
                                                              +----------------------------------+
                                                              | Guardrails, adversarial training,  |
                                                              | jailbreak mitigation, monitoring,   |
                                                              | and the advanced/adaptive tactics    |
                                                              | attackers use against all of it       |
                                                              +----------------------------------+
```

---

## 3. Module-by-Module Recap

| # | Module | The One Thing to Remember |
|---|--------|-------------------------------|
| 1 | Fundamentals of AI | ML is pattern recognition from data, not hand-written rules -- and the ML pipeline is also the attack surface map. |
| 2 | Applications of AI in InfoSec | The same tools (sklearn, PyTorch) that build spam filters and malware classifiers are what you'll be attacking and defending throughout the rest of the course. |
| 3 | Introduction to Red Teaming AI | The OWASP ML/LLM Top 10s and Google's SAIF give you a shared vocabulary and checklist for structuring an AI red-team engagement. |
| 4 | Prompt Injection Attacks | There is no hard boundary between "instructions" and "data" in an LLM's context window -- every technique in this module exploits that one structural fact. |
| 5 | LLM Output Attacks | The danger doesn't stop once the model finishes generating -- what happens when that output is rendered, executed, or trusted downstream is its own attack surface. |
| 6 | AI Data Attacks | If you can influence what a model learns from, you can control what it does later -- from mislabeling to invisible backdoors to code hidden in the weights themselves. |
| 7 | Attacking AI -- Application and System | An AI feature lives inside an ordinary software system, with ordinary software vulnerabilities (deserialization, SSRF, excessive permissions) -- plus new ones unique to agents and MCP. |
| 8 | AI Evasion -- Foundations | White-box vs. black-box, and transferability, are the two ideas that every evasion technique in Modules 9-10 builds on. |
| 9 | AI Evasion -- First-Order Attacks | Small changes to *every* feature (FGSM family, DeepFool) can cross a decision boundary the model never generalized past. |
| 10 | AI Evasion -- Sparsity Attacks | Large changes to *very few* features (JSMA, EAD, single-pixel) achieve the same goal through the opposite strategy -- stealthier, but harder to compute. |
| 11 | AI Privacy | Overfitting isn't just a quality problem -- it's a privacy leak, and membership inference/differential privacy are two sides of the same coin. |
| 12 | AI Defense | No single guardrail is a complete answer -- defense-in-depth, layered from cheap character-based filters up through human-in-the-loop gating and continuous adaptive red-teaming, is the only realistic posture. |

---

## 4. The Two Core Mental Models to Carry Forward

If you remember nothing else from this entire study guide, remember these two ideas -- nearly every topic in every module is a variation on one of them:

1. **"No hard boundary between instructions and data."** This single sentence explains prompt injection (Module 4), indirect injection, jailbreaking, LLM output attacks (Module 5), and even the advanced multi-modal and tool-output variants in this final module. Whenever you encounter a new AI attack you haven't seen before, ask first: *where is untrusted content being treated as a trusted instruction here?*
2. **"Defense-in-depth, because no single fix is complete."** This explains why Module 12 has seven layers instead of one silver bullet, why adversarial training doesn't eliminate evasion attacks, why differential privacy trades off against utility rather than solving privacy for free, and why a mature red-team engagement tests combinations of techniques rather than any one in isolation.

---

## 5. Self-Check: Can You Explain These Without Looking?

Before moving to the HTB Academy skills assessments and the certification exam itself, see if you can explain each of the following out loud, in plain English, without re-opening the file:

- Why is accuracy alone a misleading metric for a security classifier? *(Module 1)*
- Why is a Random Forest a reasonable choice for the NSL-KDD network anomaly detection project, and what does converting malware to an image actually accomplish? *(Module 2)*
- What is the difference between the ML OWASP Top 10 and the LLM OWASP Top 10, and why do both exist? *(Module 3)*
- Why is indirect prompt injection generally considered more dangerous than direct prompt injection? *(Module 4)*
- Why does LLM hallucination create a security risk (not just a quality one)? *(Module 5)*
- What is the difference between label flipping and a clean-label attack, and why is the clean-label variant harder to detect? *(Module 6)*
- What makes Model Context Protocol (MCP) attacks a distinct risk category from "just" vulnerable framework code? *(Module 7)*
- Why does an adversarial example crafted against one model often fool a completely different model (transferability)? *(Module 8)*
- What is the practical difference between FGSM and DeepFool, given that both are "first-order" attacks? *(Module 9)*
- Why is minimizing the L0 norm of a perturbation computationally harder than minimizing the L2 norm? *(Module 10)*
- Why is overfitting the root cause that makes membership inference attacks possible at all? *(Module 11)*
- Why does rate-limiting count as a defense against adaptive guardrail probing, when it doesn't fix any guardrail directly? *(Module 12)*

If any of these feel shaky, that specific file is exactly where to go back and review before the exam -- this list was built to cover one representative, non-obvious idea from every module.

---

## 6. How to Use These Notes for Exam Prep From Here

- **Work through the HTB Academy interactive skills assessments for each module** (referenced at the end of every module's README in this repo) -- these notes are built to prepare you for them, not to replace the hands-on labs themselves.
- **Revisit the "Security Angle" / "Defense Angle" section of every file** as a fast, module-spanning review pass -- reading just those sections end to end across all 12 modules gives you a compressed view of how every technique connects to real offensive/defensive practice.
- **Re-derive the diagrams from memory**, especially the defense-in-depth stack (Module 4/12), the attack-surface-by-component breakdown (Module 3), and the white-box/black-box/transferability triangle (Module 8) -- these three diagrams alone tie together most of the course's structure.
- **Treat this repo as a living reference**: as HTB Academy updates the COAE curriculum, revisit and extend these notes rather than starting over, keeping the same style (analogy first, formal definition second, worked example, security/defense angle, key takeaways) for consistency.

---

## 7. Final Word

You started this course being told to assume zero prior AI/ML background. If you've read through all 12 modules, you now have a working mental model spanning classical ML, deep learning, generative AI, applied security tooling, red-teaming frameworks, prompt injection, data poisoning, application/system-level AI attacks, adversarial evasion (both dense and sparse), privacy attacks, and layered AI defense -- the full breadth the HTB Certified Offensive AI Expert certification is built to test. Good luck on the exam.
