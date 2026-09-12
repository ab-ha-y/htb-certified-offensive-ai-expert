# Attack Surface by Component: The Application Component

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Introduction to Red Teaming AI | Section: Attack Surface by Component -- Application

---

## Table of Contents

1. [Where This Fits](#1-where-this-fits)
2. [What Is the "Application Component"?](#2-what-is-the-application-component)
3. [Where the Application Component Sits in a Deployed AI System](#3-where-the-application-component-sits-in-a-deployed-ai-system)
4. [Categories of Attacks Against the Application Component](#4-categories-of-attacks-against-the-application-component)
5. [Terminology Reference Table](#5-terminology-reference-table)
6. [Worked Example -- Scoping the Application Component of an AI Travel-Booking Agent](#6-worked-example----scoping-the-application-component-of-an-ai-travel-booking-agent)
7. [Security Angle -- Why the Application Component Feels the Most "Familiar" -- and Why That's Dangerous](#7-security-angle----why-the-application-component-feels-the-most-familiar----and-why-thats-dangerous)
8. [Key Takeaways](#8-key-takeaways)

---

## 1. Where This Fits

```
             THE FOUR COMPONENTS OF A DEPLOYED AI SYSTEM

    +-----------+     +-----------+     +--------------+     +-----------+
    |   DATA    |---->|   MODEL   |---->| APPLICATION  |---->|  SYSTEM   |
    | Component |     | Component |     |  Component   |     | Component |
    |  (earlier |     | (earlier  |     |    (this     |     |   (next   |
    |   file)   |     |   file)   |     |     file)    |     |    file)  |
    +-----------+     +-----------+     +--------------+     +-----------+
```

You've now covered attacks against the raw material (Data) and the trained brain (Model). This file covers the **Application component** -- the software layer that sits between a human (or another system) and the model, translating user intent into model queries and model outputs back into something usable. The deep hands-on technique work for this layer (especially prompt injection and related LLM application attacks) is covered in **Module 7**; this file gives you the orientation map.

---

## 2. What Is the "Application Component"?

### The Analogy

If the model is the engine and the data is the fuel supply, the **application component** is the car's **dashboard, steering wheel, and control panel** -- everything a driver actually touches and sees. You can be an excellent engine designer and still ship a car that's trivially hijackable if the steering column has no lock, or if the dashboard blindly executes any command sent to it without checking who's really behind the wheel. Most real-world car theft doesn't involve reverse-engineering the engine at all -- it involves exploiting weaknesses in the parts humans directly interact with. The same is true for AI systems.

### The Formal Definition

The **Application Component** is the software layer built *around* a model that mediates how humans and other systems interact with it. This includes:

- **Prompts and prompt templates** (for LLMs): the system prompt, the way user input gets inserted into a larger prompt structure.
- **APIs and SDKs**: the endpoints that accept requests and return model outputs.
- **Chat interfaces / UIs**: the front-end a human directly types into.
- **Plugins and tools**: extensions that let an LLM call external functions, browse the web, run code, or query other systems (this is what gives an LLM "agency" -- see the LLM Top 10 file, LLM06).
- **Retrieval pipelines (RAG)**: the logic that fetches relevant documents from a vector database and inserts them into the prompt.
- **Orchestration logic**: code that chains multiple model calls together, manages conversation state, or coordinates multi-agent workflows.

This is distinct from the Model component (the trained brain itself) and the System component (the underlying servers/infrastructure it all runs on, covered in the next file) -- the Application component is specifically the **interface and integration logic**.

---

## 3. Where the Application Component Sits in a Deployed AI System

```
              A DEPLOYED AI SYSTEM, APPLICATION-CENTRIC VIEW

   +--------+     +---------------------------------------+     +--------+
   |  USER  |---->|         APPLICATION COMPONENT          |---->| MODEL  |
   |        |     |                                        |     |Component|
   +--------+     |  +----------+  +----------+  +-------+ |     +--------+
                   |  | System   |  | Prompt   |  | RAG   | |
                   |  | Prompt   |  | Template |  | Pipe- | |
                   |  |          |  |          |  | line  | |
                   |  +----------+  +----------+  +-------+ |
                   |                                        |
                   |  +----------+  +--------------------+  |
                   |  | Plugins/ |  | Orchestration /    |  |
                   |  | Tools    |  | Agent Logic        |  |
                   |  +----------+  +--------------------+  |
                   +---------------------+------------------+
                                         |
                                         v
                              +----------------------+
                              | External systems the |
                              | app can act on: DB,   |
                              | email, web, shell,    |
                              | other APIs            |
                              +----------------------+
```

Notice something important: the application component is the **only** one of the four with direct connections to *both* the human user and the outside world of external systems and tools. This dual exposure is exactly why it's such a rich attack surface -- and exactly why the LLM OWASP Top 10 list you studied earlier devotes so many of its ten categories to this layer specifically (LLM01, LLM05, LLM06, LLM07, LLM08 all live here).

---

## 4. Categories of Attacks Against the Application Component

```
+-----------------------------------------------------------------------+
|              APPLICATION COMPONENT -- ATTACK CATEGORY MAP              |
+-----------------------------------------------------------------------+
|                                                                        |
|  PROMPT INJECTION                  Hijack the model's instructions    |
|  (LLM01)                           via crafted input                  |
|                                     --> Deep dive: Module 7             |
|                                                                        |
|  SYSTEM PROMPT LEAKAGE             Extract hidden developer            |
|  (LLM07)                           instructions                       |
|                                     --> Deep dive: Module 7             |
|                                                                        |
|  INSECURE OUTPUT HANDLING          Downstream code trusts model        |
|  (LLM05)                           output without validation           |
|                                     --> Deep dive: Module 7             |
|                                                                        |
|  EXCESSIVE AGENCY                  Model/agent has more tool           |
|  (LLM06)                           permissions than it needs           |
|                                     --> Deep dive: Module 7             |
|                                                                        |
|  RAG / RETRIEVAL PIPELINE ABUSE    Poison or leak via the vector        |
|  (LLM08)                           database / document store           |
|                                     --> Deep dive: Module 7             |
|                                                                        |
+-----------------------------------------------------------------------+
```

### 4.1 Prompt Injection (LLM01)

**What it is at a glance**: Attacker-controlled text (typed directly by a user, or hidden in content the application later processes) overrides or manipulates the model's intended instructions.

**Why it targets the application component specifically**: The vulnerability isn't really "in the model" in the way an adversarial example exploits a decision boundary -- it's in *how the application assembles the prompt*, mixing trusted developer instructions with untrusted user/external content in the same channel, with no hard boundary between them.

**Forward pointer**: You already got a full introduction to this in the LLM OWASP Top 10 file (Section 3). **Module 7** is where you'll practice crafting real injection payloads, including indirect injection via documents and tool outputs.

### 4.2 System Prompt Leakage (LLM07)

**What it is at a glance**: Extracting the hidden instructions a developer configured before the user's conversation began.

**Why it targets the application component specifically**: The system prompt is purely an **application-layer design choice** -- it's not a property of the model's weights, it's text the application code inserts before every conversation. Its leakage risk (and the fix -- never put secrets there) is entirely an application design concern.

**Forward pointer**: Covered alongside prompt injection technique work in **Module 7**, since leaked system prompts often directly enable more effective injection attacks (as shown in the chatbot worked example in the LLM Top 10 file).

### 4.3 Insecure Output Handling (LLM05)

**What it is at a glance**: Application code takes the model's generated output and feeds it directly into another sensitive system (a database query, a shell command, rendered HTML) without validating or sanitizing it.

**Why it targets the application component specifically**: This is a trust-boundary problem entirely within the application's own code -- the model did what it was manipulated into doing; the missing safety net (input validation on the *output* side) is a pure application design failure, structurally identical to classic injection vulnerabilities in traditional web apps.

**Forward pointer**: Covered in **Module 7** with concrete examples of chaining a prompt injection (get the model to generate a malicious payload) into an insecure-output-handling flaw (get that payload executed downstream).

### 4.4 Excessive Agency (LLM06)

**What it is at a glance**: An LLM or agent is granted tool access/permissions beyond what its task actually requires, so that a successful manipulation (usually via prompt injection) can cause outsized real-world damage.

**Why it targets the application component specifically**: The permissions and tool wiring are entirely decisions made by the application developer -- which functions to expose, what they're allowed to do, whether human approval is required. This is a **least-privilege** problem, structurally identical to over-permissioned service accounts in traditional systems.

**Forward pointer**: **Module 7** covers how to map an agent's full tool surface during recon and how to chain a prompt injection into an excessive-agency exploit for maximum impact (as in the SAIF worked example's auto-merge scenario).

### 4.5 RAG / Retrieval Pipeline Abuse (LLM08)

**What it is at a glance**: Exploiting the retrieval-augmented generation pipeline -- poisoning documents that later get retrieved and fed into the model's context, or exploiting weak access controls on the vector database to leak other users' data.

**Why it targets the application component specifically**: The retrieval logic (what gets searched, how access control is enforced per document, how retrieved content gets inserted into the prompt) is application-layer plumbing built around the model, not a property of the model itself.

**Forward pointer**: Covered in **Module 7**, with a focus on indirect prompt injection via poisoned documents and cross-tenant data leakage through shared vector stores.

---

## 5. Terminology Reference Table

| Term | Definition | Plain English |
|------|-----------|----------------|
| **System prompt** | Hidden developer instructions given to the model before the conversation starts. | The employee handbook the model reads before ever talking to a customer. |
| **Prompt injection** | Attacker-controlled text that overrides intended model instructions. | Slipping a fake order into the kitchen's ticket queue. |
| **Direct vs. indirect injection** | Direct = attacker types it themselves; indirect = hidden in content the model later reads. | Direct: yelling instructions at the assistant. Indirect: writing them on a note the assistant will read later. |
| **Agentic / agency** | The capability of an LLM to autonomously call tools/functions and take real-world actions. | Giving the assistant a set of keys instead of just a notepad. |
| **RAG (Retrieval-Augmented Generation)** | Fetching relevant documents from a data store and inserting them into the prompt before generation. | The model "looks things up" before answering, instead of relying only on what it memorized during training. |
| **Vector database / embedding store** | A database that stores documents as numerical vectors for similarity search. | A library catalog organized by "meaning" instead of alphabetically. |
| **Trust boundary** | The line between trusted and untrusted data/control in a system. | The velvet rope between "backstage" and "audience." |

---

## 6. Worked Example -- Scoping the Application Component of an AI Travel-Booking Agent

A travel company deploys an LLM-powered agent that can search flights, book them using the user's saved payment method, and answer travel questions using a RAG pipeline over the company's policy documents.

```
TARGET: AI Travel-Booking Agent
+--------+     +------------------------------+     +----------------------+
| User   |---->| System prompt + RAG (policy |---->| Tools:               |
| Chat   |     | docs) + conversation state   |     | - search_flights()   |
|        |     |                              |     | - book_flight($$$)   |
+--------+     +------------------------------+     +----------------------+
```

| Category | Feasibility Assessment | Notes |
|----------|------------------------|-------|
| Prompt Injection | High -- test whether a crafted message can override booking rules (e.g., booking without confirmation) | Top priority |
| System Prompt Leakage | High -- worth attempting extraction to learn exact booking-authorization logic | Directly enables better injection attacks |
| Insecure Output Handling | Medium -- check whether `book_flight()` validates parameters the LLM passes, or trusts them blindly | Test parameter injection into the tool call itself |
| Excessive Agency | High -- does this agent really need autonomous booking authority using a saved payment method, with no human confirmation step? | Classic least-privilege finding |
| RAG/Retrieval Abuse | Medium -- can a maliciously crafted "policy update" document (if the ingestion pipeline is user-influenceable) inject hidden instructions? | Depends on who can add documents to the policy knowledge base |

This mirrors the customer-support chatbot example from the LLM OWASP Top 10 file -- agentic, tool-using applications concentrate risk heavily in this component, which is exactly why Module 7 dedicates so much material to it.

---

## 7. Security Angle -- Why the Application Component Feels the Most "Familiar" -- and Why That's Dangerous

- Of the four components, the Application layer is the one that **looks most like traditional web/API security** -- and that similarity is a trap. Many organizations apply their existing web app security review process to an AI application and miss the AI-specific risks (prompt injection, excessive agency) entirely because their checklist wasn't built for this layer.
- The application component is where **most real-world, already-disclosed AI incidents have actually happened** -- far more publicized prompt-injection and jailbreak incidents exist than publicized model-inversion or data-poisoning incidents, largely because the application layer is the most exposed, most frequently updated, and easiest to reach without special access.
- This component is also the natural place where **multiple attack chains connect**: a stolen model (Model component) can be used to craft better injection payloads; a poisoned RAG document (Data-adjacent) delivers an indirect injection (Application); a successful injection then abuses excessive agency (Application) to reach the System component underneath. Thinking in components helps you see these chains rather than testing each category in isolation.
- Because agentic systems are new and evolving fast, this is also the layer where **your engagement scope needs the most explicit conversation with the client** -- "what tools can this agent call, and what happens if each one fires unexpectedly?" is a question every AI red team engagement should ask before testing begins.

---

## 8. Key Takeaways

- The **Application Component** is the interface and integration layer (prompts, APIs, plugins, RAG pipelines, orchestration logic) that mediates between users, the model, and the outside world.
- Five major LLM OWASP Top 10 categories live almost entirely in this layer: **Prompt Injection (LLM01)**, **System Prompt Leakage (LLM07)**, **Insecure Output Handling (LLM05)**, **Excessive Agency (LLM06)**, and **RAG/Retrieval Pipeline Abuse (LLM08)**.
- This is the **only component with direct exposure to both the human user and the outside world** of tools and external systems, which is why it concentrates so much real-world risk.
- Deep hands-on technique work for all five categories is covered in **Module 7**; this file is your map of what lives here and why.
- The Application layer superficially resembles traditional web/API security, which is a trap -- generic web app security reviews often miss AI-specific risks like prompt injection and excessive agency entirely.
- Attack chains frequently **cross components** through the Application layer: a stolen model informs better injection payloads, a poisoned document delivers an injection, and a successful injection abuses excessive agency to reach infrastructure underneath.

---

*Next up: System Component Attacks -- the servers, storage, cloud infrastructure, and network layer that everything else in an AI system ultimately runs on.*
