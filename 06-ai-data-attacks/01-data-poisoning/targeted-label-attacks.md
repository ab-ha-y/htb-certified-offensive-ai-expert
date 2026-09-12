# Targeted Label Attacks

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Data Attacks | Section: Data Poisoning

---

## Table of Contents

1. [What is a Targeted Label Attack?](#1-what-is-a-targeted-label-attack)
2. [Targeted vs. Random -- A Direct Comparison](#2-targeted-vs-random----a-direct-comparison)
3. [How Attackers Choose What to Target](#3-how-attackers-choose-what-to-target)
4. [Anatomy of a Targeted Attack](#4-anatomy-of-a-targeted-attack)
5. [Worked Example -- Creating a Precise Blind Spot](#5-worked-example----creating-a-precise-blind-spot)
6. [Real-World-Style Scenarios](#6-real-world-style-scenarios)
7. [Detecting Targeted Label Attacks](#7-detecting-targeted-label-attacks)
8. [Security Angle](#8-security-angle)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. What is a Targeted Label Attack?

### The Analogy

Imagine an airport security screener being trained on thousands of X-ray images of bags, each labeled "contains a weapon" or "clean." An insider secretly relabels every training image that shows a particular decorative case pattern -- the kind used by one specific smuggling ring -- from "contains a weapon" to "clean." The trainee screener studies all the images, absorbs the insider's tampering, and becomes an excellent, highly accurate screener for every case pattern in the world *except* that one. Auditors testing the screener with a random sample of bags will see near-perfect performance, because that one pattern is rare in the general population -- but the smuggling ring now has a reliable way through.

That is a **targeted label attack**: instead of degrading the model broadly (as in random label flipping), the attacker deliberately engineers a small, precise blind spot for a specific class, sample type, or condition, while keeping the model's overall behavior looking completely normal.

### The Formal Definition

A **targeted label attack** is a form of label-based data poisoning where an attacker mislabels a carefully chosen subset of training samples -- selected because they share a specific characteristic the attacker cares about -- with the goal of making the trained model systematically wrong on that characteristic while remaining accurate everywhere else. It is an **integrity attack** in the taxonomy introduced in the Data Poisoning overview file: the attacker doesn't want the model to "break," they want it to be *wrong in one useful way*.

```
   TARGETED LABEL ATTACK -- CONCEPTUAL FLOW

   +---------------------------+
   | Full training set         |
   | (e.g. 10,000 malware      |
   |  samples, all correctly   |
   |  labeled "malicious")     |
   +---------------------------+
              |
              v
   +---------------------------+
   | Attacker identifies a     |
   | SPECIFIC characteristic:  |
   | "samples signed with      |
   |  Certificate X"           |
   +---------------------------+
              |
              v
   +---------------------------+
   | Attacker relabels ONLY    |
   | those samples as          |
   | "benign"                  |
   | (everything else          |
   |  untouched)               |
   +---------------------------+
              |
              v
   +---------------------------+
   | Model trains normally on  |
   | 99%+ correctly labeled    |
   | data...                   |
   | ...but learns             |
   | "Certificate X == safe"   |
   +---------------------------+
              |
              v
   +---------------------------+
   | DEPLOYED MODEL:            |
   | - Catches malware          |
   |   normally (looks healthy) |
   | - ALWAYS lets malware       |
   |   signed with Certificate X|
   |   through                  |
   +---------------------------+
```

---

## 2. Targeted vs. Random -- A Direct Comparison

| Dimension | Random Label Flipping | Targeted Label Attack |
|---|---|---|
| **Attack category** | Availability (indiscriminate) | Integrity (targeted) |
| **Sample selection** | Random, no pattern | Deliberately chosen by shared characteristic |
| **Fraction of dataset typically needed** | Often needs to be large (5-30%+) to cause meaningful damage | Can be tiny (well under 1%) and still be devastatingly effective |
| **Effect on overall test accuracy** | Visibly drops -- easy to detect | Barely moves -- looks healthy |
| **Effect on the specific target** | No specific target; damage is diffuse | Severe, precise, and reliable |
| **Attacker knowledge required** | Minimal -- just needs write access to labels | Needs to know (or guess) a distinguishing characteristic worth exploiting |
| **Analogy** | Sabotaging an entire assembly line randomly | Sabotaging one specific checkpoint's badge reader for one specific fake badge |
| **Detectability via standard QA metrics** | High | Low -- this is the whole point |

---

## 3. How Attackers Choose What to Target

A targeted label attack lives or dies on the attacker's choice of **which characteristic to exploit**. Good targets share a few properties:

- **Rare enough in the general test/validation population** that poisoning it doesn't move aggregate metrics much, but **common enough among what the attacker actually cares about** that the blind spot is useful.
- **Identifiable and reproducible** -- the attacker (or their future malware/spam/traffic) needs to be able to consistently produce inputs that carry the targeted characteristic, so the model's blind spot can be exploited on demand later.
- **Not obviously suspicious to a human labeler** doing spot checks -- e.g., targeting "emails from a specific but plausible-looking sender domain" is subtler than targeting "emails containing the literal string BACKDOOR-TRIGGER."

| Target Characteristic Type | Example |
|---|---|
| A specific class pairing | Always mislabel "cat" images that happen to be black cats as "dog" |
| A feature value or range | Mislabel all network flows where `source_port == 31337` as benign |
| A metadata attribute | Mislabel all malware signed with a specific (attacker-controlled or stolen) code-signing certificate as benign |
| A subpopulation | Mislabel loan applications from a specific narrow demographic/zip-code range to bias a credit model |
| A stylistic pattern | Mislabel phishing emails using a specific template/wording as "ham" |

---

## 4. Anatomy of a Targeted Attack

A realistic targeted label attack usually follows these steps:

```
STEP 1: RECONNAISSANCE
   Attacker studies the target system: what data does it train on?
   Who labels it? How often does it retrain? Is there any label
   auditing?

STEP 2: TARGET SELECTION
   Attacker picks a characteristic that is (a) exploitable by them
   later, and (b) rare enough in normal traffic/data to stay hidden
   in aggregate metrics.

STEP 3: ACCESS
   Attacker gains the ability to influence labels for samples with
   that characteristic -- via insider access, a compromised labeling
   vendor, or a public feedback channel the pipeline trusts.

STEP 4: INJECTION
   Attacker mislabels the targeted subset. Crucially, they keep the
   flip count LOW relative to the whole dataset to avoid moving
   aggregate metrics.

STEP 5: RETRAINING
   The model retrains (automatically or on the next scheduled cycle)
   on the now-poisoned data, absorbing the blind spot.

STEP 6: EXPLOITATION
   Attacker crafts real-world inputs carrying the targeted
   characteristic (e.g. signs malware with the targeted certificate,
   sends phishing using the targeted template) and reliably evades
   the model.
```

---

## 5. Worked Example -- Creating a Precise Blind Spot

### Setup

- Intrusion detection dataset: **N = 20,000** labeled network flow records.
- 4,000 malicious flows (20%), 16,000 benign flows (80%).
- Clean model baseline on a 2,000-flow test set: Accuracy 95.5%, Precision 92%, Recall 88%.
- The attacker controls a botnet that always communicates over a distinctive but rare combination: destination port 4444 paired with a specific packet-size signature. In the current (clean) training data, **80 malicious flows (2% of the malicious class, 0.4% of the whole dataset)** already have this signature.

### The Attack

The attacker gains limited write access to the labeling pipeline (e.g., through a compromised labeling contractor) and flips the label on those **80 flows** from "malicious" to "benign." That's just **80 out of 20,000 total training samples -- 0.4% of the entire dataset.**

### Illustrative Result

```
Overall test set (2,000 flows) -- looks basically unchanged:

                     PREDICTED
                  Malicious   Benign
 ACTUAL Malicious [  368   |   32   ]    <-- overall recall barely dips
        Benign    [   58   |  1542  ]

 Accuracy  = (368 + 1542) / 2000 = 95.5%   (UNCHANGED from baseline!)
 Precision = 368 / (368 + 58)    = 86.4%   (small dip from 92%)
 Recall    = 368 / (368 + 32)    = 92.0%   (actually looks slightly BETTER,
                                             just statistical noise from the
                                             specific test split)

BUT specifically for flows matching the attacker's port-4444 + packet-size
signature in the test set (8 such malicious flows happened to be in this
particular test split):

 Before poisoning: 7 / 8 correctly flagged malicious   (87.5% caught)
 After poisoning:   0 / 8 correctly flagged malicious   (0% caught --
                                                          total blind spot)
```

The headline metrics (95.5% accuracy) look identical to the clean baseline -- a security team reviewing the model's dashboard would see nothing wrong. But the attacker has purchased, for the price of 80 mislabeled training samples out of 20,000 (0.4%), a **100% reliable bypass** for their specific botnet's traffic signature. This is the defining characteristic of a well-executed targeted label attack: **maximum, precise damage for minimum, well-hidden cost.**

---

## 6. Real-World-Style Scenarios

| Domain | Targeted Characteristic | Attacker Benefit |
|---|---|---|
| Malware classification | A specific code-signing certificate or packer | Malware signed/packed that way is always classified benign |
| Spam/phishing filtering | A specific sender domain or email template | Phishing campaigns using that domain/template bypass the filter |
| Network intrusion detection | A specific port + payload-size combination | A C2 (command-and-control) channel using that signature evades detection |
| Fraud detection | Transactions below a specific dollar amount from a specific merchant category | Fraud structured to fit that pattern goes unflagged |
| Facial recognition access control | A specific individual's face mislabeled as "authorized" | That individual (or anyone impersonating their features) gets access |
| Content moderation | A specific hashtag, watermark, or stylistic marker | Content carrying that marker evades moderation |

---

## 7. Detecting Targeted Label Attacks

Detecting targeted label attacks is harder than detecting random flipping, precisely because aggregate metrics stay healthy. Focused defenses include:

| Technique | How It Helps |
|---|---|
| **Subgroup/slice-based evaluation** | Don't just check overall accuracy -- evaluate model performance across many feature-based slices (by port, by sender domain, by certificate, by demographic) to surface hidden blind spots |
| **Influence function analysis** | Identify which training samples have unusually large influence on specific predictions; investigate high-influence samples for a specific characteristic |
| **Red-teaming with adversarially crafted probes** | Proactively test the model against inputs engineered to carry rare-but-plausible characteristics, not just the "average" test distribution |
| **Label provenance and change auditing** | Track every label change with who/what made it and why; flag unusual patterns like "all changes concentrated on samples sharing feature X" |
| **Comparing model versions over time** | If a specific slice's performance drops sharply between model versions while overall accuracy stays flat, investigate that retraining cycle's data diff |
| **Diverse, independent re-labeling of rare slices** | Specifically re-verify labels for rare/unusual feature combinations, since these are the most attractive targets and the easiest to hide in aggregate stats |

---

## 8. Security Angle

Targeted label attacks are the technique that separates "someone who read about data poisoning" from "someone who can actually execute a stealthy, useful attack against a production ML system." Key offensive takeaways:

- **This is the attack you actually want to use in a real engagement or red-team exercise**, not indiscriminate flipping -- it demonstrates real, exploitable business impact (a reliable bypass) rather than just "we can make the model worse," which is a much weaker finding.
- **The hardest part of a targeted attack is choosing the right characteristic** -- one that is exploitable, reproducible, and rare enough to stay hidden. This requires genuine reconnaissance of the target's domain, not just data access.
- **This attack composes naturally with Clean-Label Attacks (next section) and Backdoor/Trojan Attacks (Section 3 of this module).** In fact, a targeted label attack IS a simple form of backdoor: the "trigger" is just "has characteristic X," and it's injected purely through the label rather than through feature manipulation. The more advanced techniques later in this module make the trigger itself unobtrusive in the underlying data too, not just in the label.
- When evaluating a target's ML pipeline defensively (or looking for a way in offensively), ask: *does anyone evaluate model performance on feature-based slices, or only on aggregate metrics?* Most organizations only check the aggregate -- which is exactly the gap targeted label attacks are built to exploit.

---

## 9. Key Takeaways

- **A targeted label attack deliberately mislabels a specific, narrow subset of training samples** sharing an attacker-chosen characteristic, rather than mislabeling broadly and randomly.
- **It is an integrity attack**: the goal is a precise, exploitable blind spot, not general model degradation -- and it is specifically engineered to keep aggregate accuracy/precision/recall looking healthy.
- **The worked example showed that flipping just 0.4% of a 20,000-sample dataset** (80 flows sharing a specific traffic signature) left overall accuracy completely unchanged while creating a total (0%) blind spot for that exact signature.
- **Good targets are rare in the general population but reproducible by the attacker on demand** -- this combination is what makes the attack both stealthy and exploitable.
- **Standard aggregate QA metrics will not catch this.** Defense requires subgroup/slice-based evaluation, influence analysis, and targeted red-teaming.
- **Targeted label attacks are conceptually the simplest form of a backdoor** -- the next two files in this module (Clean-Label Attacks and Trojan/Backdoor Attacks) build directly on this idea by also manipulating the underlying features, not just the labels.

*Next up: Clean-Label Attacks -- where the attacker leaves labels completely correct and instead poisons the data by subtly perturbing the features themselves, making the attack even harder to catch through label auditing.*
