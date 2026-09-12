# Feature Obfuscation (GoodWords Attack Methodology)

> HTB Certified Offensive AI Expert -- Study Guide
> Module: AI Evasion - Foundations | Section: Feature Obfuscation (GoodWords Attack Methodology)

---

## Table of Contents

1. [What Is Feature Obfuscation?](#1-what-is-feature-obfuscation)
2. [The Foundational Idea: Presence/Absence Features](#2-the-foundational-idea-presenceabsence-features)
3. [The GoodWords Attack, Explained Simply](#3-the-goodwords-attack-explained-simply)
4. [The Math: Bag-of-Words and Linear Scoring](#4-the-math-bag-of-words-and-linear-scoring)
5. [Worked Numeric Example 1: Evading a Spam Filter](#5-worked-numeric-example-1-evading-a-spam-filter)
6. [Worked Numeric Example 2: Evading a Malware Feature Classifier](#6-worked-numeric-example-2-evading-a-malware-feature-classifier)
7. [Two Directions of the Attack: Padding vs. Removal](#7-two-directions-of-the-attack-padding-vs-removal)
8. [Why "Real-World Effect" Must Stay Unchanged](#8-why-real-world-effect-must-stay-unchanged)
9. [Relationship to Gradient-Based Attacks](#9-relationship-to-gradient-based-attacks)
10. [Defensive Countermeasures](#10-defensive-countermeasures)
11. [Security Angle](#11-security-angle)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. What Is Feature Obfuscation?

**Feature obfuscation** is an evasion strategy that targets classifiers which make decisions primarily based on the **presence or absence** of specific "trigger" features -- rather than on subtle, continuous, gradient-sensitive numeric patterns. Instead of computing a gradient and nudging every feature by a tiny calculated amount (as in the white-box attacks from the first file in this module), the attacker simply **adds features that look "good" (benign-indicating) or removes/hides features that look "bad" (malicious-indicating)**.

### The Analogy

Imagine a bouncer at a club who has a simple mental checklist: "if this person is wearing a leather jacket AND has visible tattoos AND arrives after 1 AM, I flag them as trouble." Someone who genuinely intends to cause trouble can trivially defeat this checklist by borrowing a friend's blazer, wearing long sleeves, and arriving at 11 PM instead. Nothing about their actual intentions changed -- they are just as much a troublemaker as before -- but every trigger the bouncer's checklist relies on has been hidden or replaced with something that reads as "normal."

Feature obfuscation attacks work exactly the same way against ML classifiers that rely heavily on specific, human-interpretable trigger features: **hide the bad signals, add extra good signals, and the classifier's decision flips -- even though the underlying content or intent is unchanged.**

---

## 2. The Foundational Idea: Presence/Absence Features

Many real-world classifiers -- especially older, simpler, and still widely-deployed ones -- represent their input as a set of **binary (0/1) or count-based features** describing whether certain things are present:

- A **spam filter** might have features like `contains_word("free")`, `contains_word("viagra")`, `count_of_exclamation_marks`.
- A **malware classifier** might have features like `imports_function("VirtualAllocEx")`, `has_section_named(".packed")`, `calls_api("CreateRemoteThread")`.
- A **phishing URL detector** might have features like `contains_substring("login")`, `uses_ip_address_instead_of_domain`, `has_suspicious_tld`.

This kind of feature representation is extremely common because it is simple, fast to compute, and highly interpretable to human analysts. But it has a structural weakness: **each feature is a discrete, human-legible "checkbox," and the model's final decision is often close to a weighted sum ("how many bad checkboxes are ticked, minus how many good checkboxes are ticked") -- which means the attacker can directly manipulate the checkboxes themselves, without needing any calculus at all.**

```
   A PRESENCE/ABSENCE FEATURE VECTOR

   Email text: "Free VIAGRA offer!!! Click now, limited time!!!"

   Feature vector (1 = present, 0 = absent):

   contains_"free"          = 1
   contains_"viagra"        = 1
   contains_"click"         = 1
   contains_"limited"       = 1
   count_exclaim > 2        = 1
   contains_"meeting"       = 0
   contains_"regards"       = 0
   sender_in_contacts       = 0

   -----------------------------------------
   Total "bad" checkboxes ticked: 5
   Total "good" checkboxes ticked: 0
   -----------------------------------------
   Model's rule (simplified): if (bad_count - good_count) > threshold
   --> SPAM
```

---

## 3. The GoodWords Attack, Explained Simply

The **GoodWords attack** (also called "good word insertion" or "good word attack" in the spam-filtering literature, originating from research on evading Bayesian and linear spam classifiers) is one of the earliest and most foundational evasion techniques studied in adversarial machine learning. Its core idea, stripped to plain English:

> **Pad the input with lots of words/features that the classifier strongly associates with the benign class, so that the "benign evidence" in the input outweighs the "malicious evidence" -- even though the actual malicious content is still fully present and functional.**

### Why "GoodWords" Specifically?

The name comes from its original context: text classifiers (like early spam filters using Naive Bayes) score a message by essentially adding up evidence from every word in it. Certain words are extremely strong evidence for "ham" (legitimate mail) -- ordinary, everyday words that appear constantly in real correspondence but rarely in spam (e.g., specific personal names, ordinary date references, common business terms). By stuffing an email with dozens of these "good words" -- often invisibly, e.g., in white-on-white text, in an image's alt-text, or in an HTML comment that the human recipient never sees but the classifier's parser does -- the attacker can drown out the "bad word" evidence in the classifier's math, even while the visible spam content the human reader sees is completely unchanged.

```
   GOODWORDS ATTACK -- BEFORE AND AFTER

   BEFORE (caught as spam):
   -------------------------------------------
   "Free VIAGRA offer!!! Click now, limited
    time!!!"
   -------------------------------------------
   Visible to human: obviously spam
   Visible to classifier: obviously spam
   Classifier score: SPAM (high confidence)


   AFTER (goodwords injected -- often invisible to the human):
   -------------------------------------------
   "Free VIAGRA offer!!! Click now, limited
    time!!!"
   <!-- meeting schedule quarterly report
        attached invoice regards conference
        agenda budget approval team calendar
        Tuesday afternoon project deadline
        thank you looking forward colleagues -->
   -------------------------------------------
   Visible to human: STILL obviously spam
   (the hidden text is invisible -- HTML comment,
    tiny white font, or off-screen)
   Visible to classifier: huge amount of "ham"-
   associated vocabulary now dominates the word
   count
   Classifier score: HAM (misclassified)
```

**The critical property of this attack**: the real-world effect of the email (a spam offer trying to get the recipient to click a link) is completely unchanged. Only the *classifier's* view of the content has been manipulated. This is exactly the definition of an evasion attack given in the first file of this module: the input's real-world meaning stays the same, but the model's decision flips.

---

## 4. The Math: Bag-of-Words and Linear Scoring

To understand precisely why this works, we need two small pieces of math: the **bag-of-words** representation, and how a simple linear (or Naive-Bayes-style) classifier scores it.

### 4.1 Bag-of-Words, Simply

A "bag-of-words" model represents a piece of text as **just a count of how many times each word appears**, throwing away word order entirely (hence "bag" -- like dumping all the words into a bag and just counting what's in there).

```
Text: "free money free now"

Bag-of-words counts:
  free  = 2
  money = 1
  now   = 1
```

### 4.2 Linear (Additive) Scoring

Many classic text classifiers -- including Naive Bayes, which was the historical target of the original GoodWords research -- can be understood, for our purposes, as computing a score that is **a weighted sum over every word in the vocabulary**:

```
score(document) = sum over every word w in the document of:
                       count(w) * weight(w)

If score > threshold  -->  SPAM
If score <= threshold -->  HAM
```

Each `weight(w)` was learned during training. Words that appeared disproportionately often in spam training examples get a **large positive weight**. Words that appeared disproportionately often in ham training examples get a **large negative weight**. Neutral words (appearing about equally in both) get a weight close to zero.

```
Example learned weights (illustrative):

   weight("viagra")   = +8.0   (strong spam signal)
   weight("free")     = +3.0   (moderate spam signal)
   weight("click")    = +2.0   (moderate spam signal)
   weight("meeting")  = -4.0   (strong ham signal)
   weight("regards")  = -3.5   (strong ham signal)
   weight("quarterly")= -2.5   (moderate ham signal)
```

### 4.3 Why Padding Words Works Mathematically

Because the score is an **additive sum**, adding *more terms* with **negative weights** directly pulls the total score down -- regardless of what the rest of the document already contains. This is the entire mathematical mechanism behind the GoodWords attack: it is exploiting the additive/linear structure of the scoring function.

```
score_new = score_old + (added_word_count * negative_weight)

Since negative_weight < 0, adding enough good words drives
score_new below the SPAM threshold, no matter how high
score_old was to begin with.
```

Compare this to the gradient-based attacks from the first file in this module: there, the attacker computed an exact gradient and took a small numeric step in continuous feature space. Here, the attacker doesn't need any gradient at all -- because the feature space is made of discrete presence/absence or count features, and the scoring function is a simple sum, the "direction that decreases the score" is obvious just by inspecting which features/words have negative weights. **This is why the GoodWords attack predates, and does not require, any of the gradient machinery used in modern deep-learning evasion attacks -- it works purely from the additive structure of presence/absence-based scoring.**

---

## 5. Worked Numeric Example 1: Evading a Spam Filter

Let's fully work through a concrete numeric example using the illustrative weights from Section 4.2.

### 5.1 The Original Spam Email

```
Text: "Free free VIAGRA offer! Click now! Limited time!"

Word counts relevant to our model:
   free   = 2
   viagra = 1
   click  = 1

score = (2 * weight("free")) + (1 * weight("viagra")) + (1 * weight("click"))
      = (2 * 3.0) + (1 * 8.0) + (1 * 2.0)
      = 6.0 + 8.0 + 2.0
      = 16.0

Classifier threshold = 5.0
16.0 > 5.0  -->  SPAM (correctly caught)
```

### 5.2 Injecting GoodWords

The attacker appends a block of hidden text (e.g., invisible HTML) containing common office vocabulary:

```
Injected text (hidden from human reader):
"meeting meeting regards quarterly quarterly regards"

Word counts added:
   meeting   = 2
   regards   = 2
   quarterly = 2

Additional score contribution:
   = (2 * weight("meeting")) + (2 * weight("regards")) + (2 * weight("quarterly"))
   = (2 * -4.0) + (2 * -3.5) + (2 * -2.5)
   = -8.0 + -7.0 + -5.0
   = -20.0

New total score = 16.0 + (-20.0) = -4.0

-4.0 <= 5.0 threshold --> HAM (misclassified!)
```

**With just six injected words (three good words, twice each), the spam email's score dropped from 16.0 (clearly spam) to -4.0 (below threshold, classified as ham) -- while every single visible word a human recipient would read remains completely unchanged.** This is the GoodWords attack in its purest, most classic form.

### 5.3 Visualizing the Score Shift

```
   SPAM SCORE BEFORE AND AFTER GOODWORDS INJECTION

   score
     20 |
     16 |  #  <- original score (SPAM, well above threshold)
     12 |  #
      8 |  #
      5 |..#................................ <- threshold
      4 |  #
      0 |  #
     -4 |  #  *  <- new score after injection (HAM, below threshold)
     -8 |  #  *
        +------------------------------------
           original    after goodwords
           email         injected
```

---

## 6. Worked Numeric Example 2: Evading a Malware Feature Classifier

The same principle applies far beyond spam filters -- it generalizes directly to any classifier built on presence/absence "trigger" features, including malware and PE (Portable Executable) file classifiers.

### 6.1 The Setup

Imagine a lightweight malware classifier that scores an executable file based on the presence of certain imported Windows API functions and structural characteristics:

```
Learned feature weights (illustrative):

   imports("VirtualAllocEx")       = +6.0   (used in process injection)
   imports("WriteProcessMemory")   = +5.0   (used in process injection)
   imports("CreateRemoteThread")   = +7.0   (used in process injection)
   has_digital_signature           = -5.0   (signed files are usually legit)
   imports("GetSystemMetrics")     = -1.0   (extremely common in legit UI apps)
   has_version_info_resource       = -2.0   (legitimate apps usually have this)
   company_name_field_populated    = -1.5   (legitimate apps usually have this)
```

### 6.2 The Original Malicious File

```
Feature presence for a real injector-style malware sample:
   imports("VirtualAllocEx")      = 1
   imports("WriteProcessMemory")  = 1
   imports("CreateRemoteThread")  = 1
   has_digital_signature          = 0
   has_version_info_resource      = 0
   company_name_field_populated   = 0

score = 1*6.0 + 1*5.0 + 1*7.0 + 0 + 0 + 0 = 18.0

Threshold = 10.0
18.0 > 10.0  -->  MALICIOUS (correctly detected)
```

### 6.3 The Feature Obfuscation Attack

The attacker cannot remove `VirtualAllocEx`, `WriteProcessMemory`, or `CreateRemoteThread` from the binary -- those API calls are functionally required for the malware to perform its process-injection payload; removing them would break the malware's actual capability (this constraint is explored more in Section 8). But the attacker *can* pad the binary with harmless, unrelated "good" characteristics that cost nothing functionally:

- Add a (potentially fraudulent or trivially self-signed, or in some real-world cases stolen/valid) digital signature.
- Populate the PE version-info resource block with a plausible company name, product name, and version number.
- Fill in the company name field.

None of these additions touch the malicious payload at all -- they are purely metadata:

```
Feature presence after obfuscation:
   imports("VirtualAllocEx")      = 1   (unchanged -- required for payload)
   imports("WriteProcessMemory")  = 1   (unchanged -- required for payload)
   imports("CreateRemoteThread")  = 1   (unchanged -- required for payload)
   has_digital_signature          = 1   (ADDED)
   has_version_info_resource      = 1   (ADDED)
   company_name_field_populated   = 1   (ADDED)

score = 1*6.0 + 1*5.0 + 1*7.0 + 1*(-5.0) + 1*(-2.0) + 1*(-1.5)
      = 6.0 + 5.0 + 7.0 - 5.0 - 2.0 - 1.5
      = 9.5

Threshold = 10.0
9.5 <= 10.0  -->  BENIGN (misclassified!)
```

**By adding just three cosmetic, functionally irrelevant metadata features, the score dropped from 18.0 to 9.5 -- just barely crossing the threshold from MALICIOUS to BENIGN, without altering a single byte of the actual injection payload.** The malware retains 100% of its real-world capability; only the classifier's view of it changed.

---

## 7. Two Directions of the Attack: Padding vs. Removal

The GoodWords methodology has two complementary directions, both exploiting the same additive scoring structure, applied in opposite ways:

| Direction | What the Attacker Does | Effect on Score | Example |
|---|---|---|---|
| **Good-word/feature padding (injection)** | Add extra features/words strongly associated with the benign class | Adds large *negative* terms to the score, pulling it down | Hidden text with common office vocabulary; a forged/legitimate-looking digital signature; adding a version-info resource block |
| **Bad-word/feature removal or obfuscation** | Remove, rename, encode, or otherwise hide features strongly associated with the malicious class | Removes large *positive* terms from the score entirely | Replacing "free" with "fr33" or a lookalike Unicode character so the exact-match feature no longer fires; dynamically resolving an API call at runtime instead of importing it statically, so the static-analysis feature `imports("CreateRemoteThread")` never triggers |

```
   TWO DIRECTIONS, SAME GOAL: DECREASE score BELOW threshold

   score = sum(bad_feature_hits * positive_weight)
         - sum(good_feature_hits * |negative_weight|)

   Direction 1 (padding):  INCREASE the second term
                            (add more good-feature hits)

   Direction 2 (removal):  DECREASE the first term
                            (eliminate bad-feature hits)

   Real attacks very often combine BOTH directions simultaneously
   for maximum effect and reliability.
```

In text classification specifically, "bad-word obfuscation" commonly takes the form of character substitution (`viagra` -> `v1agra` or `vi@gra`), insertion of non-printing/zero-width characters inside a trigger word (`v​iagra` with a hidden zero-width character), or homoglyph substitution (replacing Latin letters with visually identical Cyrillic or Greek letters) -- all techniques that defeat a literal string-match feature while leaving the word perfectly readable to a human.

---

## 8. Why "Real-World Effect" Must Stay Unchanged

This constraint is what separates a *legitimate* evasion attack from simply "not doing the bad thing anymore," and it deserves special emphasis because it's easy to miss.

Consider the malware example in Section 6: the attacker was very careful to only add cosmetic metadata (signature, version info, company name) and never touched the three API imports that actually implement the process-injection capability. If the attacker had instead simply *removed* `CreateRemoteThread` from the binary to dodge the classifier, the file would score as benign too -- but it would also no longer be able to perform process injection. That's not an evasion attack anymore; that's just... not building the malware. The entire point of an evasion attack is:

```
   THE CORE CONSTRAINT OF EVASION ATTACKS

   +---------------------------+        +---------------------------+
   | Real-world functionality  |  MUST  | Real-world functionality  |
   | of ORIGINAL input          | ==EQUAL==| of ADVERSARIAL input       |
   +---------------------------+  STAY  +---------------------------+
                                  SAME
   +---------------------------+        +---------------------------+
   | Classifier's decision on   |  MUST  | Classifier's decision on  |
   | ORIGINAL input (correct)   | ==DIFFER==| ADVERSARIAL input (WRONG) |
   +---------------------------+        +---------------------------+
```

This constraint is exactly why obfuscation-based attacks focus so heavily on **features that are informative to the classifier but incidental to the actual malicious/spam behavior** -- metadata, formatting, padding, invisible content, cosmetic structural properties -- rather than features that are load-bearing for the attack's real function. Good-word padding is popular precisely because injecting extra hidden text or metadata is "free" in the sense that it doesn't cost the attacker anything functionally, while still meaningfully moving the classifier's score.

---

## 9. Relationship to Gradient-Based Attacks

It's worth being explicit about how feature obfuscation relates to, and differs from, the gradient-based white-box attacks from the first file in this module -- because both are trying to solve the same underlying problem (cross the decision boundary) using different tools suited to different feature representations.

| Aspect | Gradient-based (FGSM/PGD-style) | Feature Obfuscation (GoodWords-style) |
|---|---|---|
| **Feature type it targets** | Continuous, dense numeric features (pixel intensities, sensor readings, learned embeddings) | Discrete, sparse presence/absence or count features (words, API calls, structural flags) |
| **Requires model gradients?** | Yes -- fundamentally gradient-based | No -- works from feature weights or even just domain intuition about what looks "good"/"bad" |
| **Requires white-box access?** | Typically yes (or a surrogate for transfer) | Often no -- if the attacker has any reasonable understanding of the domain (which words/features look benign), no model access is needed at all |
| **Underlying math exploited** | Local linearity / calculus -- the gradient of a differentiable scoring function | Additivity of a linear/count-based scoring function -- simple arithmetic, not calculus |
| **Typical target models** | Neural networks, deep learning models, differentiable pipelines | Naive Bayes, linear models, decision trees/rules, older or interpretable "checklist"-style classifiers, and rule-based components even inside larger pipelines |
| **Stealth in crafting** | Often needs to query the model (or a surrogate) many times, or requires white-box access | Can sometimes be done with zero model queries at all -- pure domain knowledge ("spam filters hate the word 'free'; I bet they love the word 'meeting'") |

In practice, real classifiers are frequently a **mix** of both feature types (e.g., a malware detector combining continuous statistical features like entropy alongside discrete features like specific API imports), and a thorough evasion assessment considers both attack families together. The next two modules (9 and 10) build directly on both of these foundations.

---

## 10. Defensive Countermeasures

Because feature obfuscation exploits the additive, presence/absence nature of certain feature representations, defenses tend to target that structure directly:

- **Non-linear / interaction-aware models**: Move away from purely additive linear scoring toward models that consider feature *interactions* (e.g., "lots of good words AND a suspicious link together" should not simply cancel out) -- tree ensembles and neural networks with non-linear combinations are harder to fool with simple additive padding than a pure linear model.
- **Robust feature engineering**: Normalize/canonicalize text before feature extraction (lowercase, strip zero-width characters, resolve homoglyphs to their base character, decode obfuscated encodings) so that bad-word obfuscation via character substitution doesn't actually hide the trigger feature.
- **Rendered/visual content analysis**: For hidden-text goodword attacks specifically, analyze what a human would actually *see* (rendered HTML/CSS, visible font size and color) rather than raw extracted text, since invisible padding is a rendering-layer trick.
- **Capping feature influence / weight clipping**: Limit how much any single feature (or category of features, like "count of common words") can influence the final score, preventing an attacker from drowning out strong signals through sheer volume of weak ones.
- **Behavioral/dynamic analysis alongside static features**: For malware specifically, complement static, easily-obfuscated features (imports, metadata) with dynamic/behavioral analysis (what does the file actually *do* when run in a sandbox), which is much harder to obfuscate via cosmetic metadata changes.
- **Adversarial training with obfuscated samples**: Deliberately include goodword-padded and bad-word-obfuscated samples (still correctly labeled malicious/spam) in the training set, so the model learns that these obfuscation patterns are themselves suspicious.

---

## 11. Security Angle

Feature obfuscation attacks are historically significant and remain highly practical today for a simple reason: **a huge number of production classifiers, especially in email security, endpoint/AV products, and web application firewalls, still rely heavily on presence/absence or rule-like features -- even when a deep neural network sits somewhere in the pipeline**, because interpretable features are valuable for analyst triage, compliance, and explainability.

```
   WHERE GOODWORDS-STYLE ATTACKS STILL MATTER TODAY

   +----------------------+   +----------------------+   +----------------------+
   | Email security       |   | Endpoint/AV products |   | Web application      |
   | gateways              |   |                       |   | firewalls (WAFs)      |
   |                       |   |                       |   |                       |
   | - Keyword/phrase      |   | - Static import/API   |   | - Signature/keyword   |
   |   scoring              |   |   feature scoring     |   |   based rules          |
   | - Header/metadata      |   | - PE metadata          |   | - Payload keyword      |
   |   feature checks        |   |   heuristics           |   |   matching             |
   +----------------------+   +----------------------+   +----------------------+
```

**For red teamers**: before reaching for gradient-based methods (which require model access you may not have), always ask "does this target rely on presence/absence, keyword, or rule-like features I can identify and directly manipulate?" This is often the *lowest-effort, highest-success* evasion path, requires no model internals at all, and is exactly how many real-world spam and malware evasion campaigns have historically operated (and continue to operate) in practice.

**For defenders**: audit your own feature set for this exact weakness. If your classifier's decision can be meaningfully swayed by adding a fixed block of unrelated "good" content, or by cosmetic renaming/encoding of a "bad" trigger, you have a GoodWords-style vulnerability, regardless of how sophisticated the model architecture sitting on top of those features is.

---

## 12. Key Takeaways

- **Feature obfuscation** targets classifiers that rely on the presence/absence of specific "trigger" features, rather than exploiting continuous gradient information.

- **The GoodWords attack** pads an input with features/words strongly associated with the benign class (or removes/hides features associated with the malicious class), shifting an additive classifier score across its decision threshold -- without changing the input's real-world effect.

- **Mathematically**, this exploits the additive structure of linear/count-based scoring functions: `score = sum(feature_hits * weight)`. Adding negative-weighted "good" terms, or removing positive-weighted "bad" terms, directly moves the score -- no calculus or gradient access required.

- In the spam example, injecting six hidden "good words" flipped a clearly-spam score of 16.0 to -4.0, below the classification threshold -- with the visible email content completely unchanged.

- In the malware example, adding three cosmetic metadata features (digital signature, version info, company name) flipped a malicious score of 18.0 to a benign 9.5 -- without touching the API calls that actually implement the malicious payload.

- **Padding (good-feature injection)** and **removal/obfuscation (bad-feature hiding)** are the two complementary directions of this attack, both exploiting the same additive scoring structure; real attacks often combine both.

- **The real-world-effect constraint is central**: an evasion attack must leave the underlying malicious/spam functionality intact while only changing the classifier's *view* of it. Removing functionally necessary "bad" features isn't evasion -- it's just building something less dangerous.

- **This attack family predates and does not require gradient access**, making it especially relevant when the attacker has no white-box or even query-based access to the model, only domain knowledge of what the classifier is likely keying on.

- **Defenses focus on breaking the additive/presence-absence assumption**: non-linear interaction-aware models, feature normalization/canonicalization, capping any single feature's influence, complementary behavioral analysis, and adversarial training on obfuscated samples.

---

*This completes the "AI Evasion - Foundations" module. With white-box vs. black-box threat models, transferability, and feature obfuscation now in place, the next modules (9 and 10) build directly on these foundations to study specific gradient-based algorithms (FGSM, PGD, and beyond) and advanced evasion techniques against deep learning and modern production systems.*
