# HTB Certified Offensive AI Expert (COAE) -- Study Guide

Personal study notes for the [HackTheBox Certified Offensive AI Expert](https://academy.hackthebox.com/) certification, covering the full "AI Red Teamer" job-role path (12 modules).

Written for someone new to AI/ML: every concept starts with a plain-English analogy before the formal definition, uses ASCII diagrams to visualize architectures/pipelines/attacks, includes worked numeric examples, and ends with a Security Angle / Defense Angle section connecting it back to offensive AI security.

---

## Curriculum

| # | Module | Notes |
|---|--------|-------|
| 1 | [Fundamentals of AI](01-fundamentals-of-ai/README.md) | ML/DL/GenAI basics -- supervised, unsupervised, reinforcement learning, neural networks, LLMs, diffusion models |
| 2 | [Applications of AI in InfoSec](02-applications-of-ai-in-infosec/README.md) | Hands-on: environment setup, scikit-learn/PyTorch, spam classification, network anomaly detection, malware classification |
| 3 | [Introduction to Red Teaming AI](03-introduction-to-red-teaming-ai/README.md) | OWASP ML Top 10, OWASP LLM Top 10, Google's SAIF, attack surface by component |
| 4 | [Prompt Injection Attacks](04-prompt-injection-attacks/README.md) | Direct/indirect injection, jailbreaking (DAN, framing, crescendo, token smuggling, many-shot), defense-in-depth mitigations |
| 5 | [LLM Output Attacks](05-llm-output-attacks/README.md) | XSS/SQLi/command injection via LLM output, function-calling attacks, exfiltration, hallucination, abuse, regulation |
| 6 | [AI Data Attacks](06-ai-data-attacks/README.md) | Data poisoning, label flipping, clean-label/feature attacks, trojans/backdoors, tensor steganography, pickle exploitation |
| 7 | [Attacking AI -- Application and System](07-attacking-ai-application-and-system/README.md) | Model reverse engineering, denial of ML service, insecure integrations, rogue actions, deployment tampering, MCP attacks |
| 8 | [AI Evasion -- Foundations](08-ai-evasion-foundations/README.md) | White-box vs. black-box, transferability, feature obfuscation (GoodWords) |
| 9 | [AI Evasion -- First-Order Attacks](09-ai-evasion-first-order-attacks/README.md) | Norm constraints, FGSM, targeted FGSM, I-FGSM, DeepFool |
| 10 | [AI Evasion -- Sparsity Attacks](10-ai-evasion-sparsity-attacks/README.md) | L0/L1 sparsity, saliency-based selection, EAD, FISTA, JSMA, single-pixel attacks |
| 11 | [AI Privacy](11-ai-privacy/README.md) | Membership inference (shadow models), differential privacy, DP-SGD, PATE, privacy-utility tradeoffs |
| 12 | [AI Defense](12-ai-defense/README.md) | Guardrails, adversarial training, jailbreak mitigation, multi-turn monitoring, advanced/adaptive attacker tactics |

Each module's `README.md` links to every file in that module. Interactive skills assessments live on HTB Academy itself and are not duplicated here.
