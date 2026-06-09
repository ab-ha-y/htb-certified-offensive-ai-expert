# Reinforcement Learning Algorithms

## Table of Contents

1. [What is Reinforcement Learning?](#what-is-reinforcement-learning)
2. [The Core Components](#the-core-components)
3. [The RL Loop](#the-rl-loop)
4. [Exploration vs Exploitation](#exploration-vs-exploitation)
5. [How RL Compares to Other Learning Types](#how-rl-compares-to-other-learning-types)
6. [Markov Decision Processes (MDP)](#markov-decision-processes-mdp)
7. [Key RL Algorithms Overview](#key-rl-algorithms-overview)
8. [Security and Offensive Applications](#security-and-offensive-applications)
9. [Key Terminology](#key-terminology)
10. [Strengths and Weaknesses](#strengths-and-weaknesses)
11. [Key Takeaways](#key-takeaways)

---

## What is Reinforcement Learning?

### The Dog Training Analogy

Imagine you are training a dog. You do not hand it a textbook of commands (that would be supervised learning). You do not let it sniff around and sort toys into piles on its own (that would be unsupervised learning). Instead, you let the dog **try things**, and then you give it a **treat** when it does something good and say "no" when it does something bad.

Over time, the dog learns: "Sitting when the human says 'sit' gets me a treat. Chewing the couch does not." The dog has no teacher showing it the correct answer beforehand. It learns entirely from **trial and error** and the **feedback** it receives.

**That is reinforcement learning.**

More formally: Reinforcement Learning (RL) is a type of machine learning where an **agent** learns to make decisions by performing **actions** in an **environment** and receiving **rewards** or **penalties** based on the outcomes. The agent's goal is to maximize its total reward over time.

### Why This Matters

RL is fundamentally different from other ML approaches because:

- There is no labeled dataset to learn from
- The agent must discover good strategies on its own
- Actions have consequences that play out over time (not just immediately)
- The agent must balance trying new things with sticking to what works

---

## The Core Components

Every reinforcement learning system has five essential pieces:

### 1. Agent

The **learner and decision-maker**. This is the entity that takes actions and learns from the results. Think of it as the dog in our analogy, or in a security context, an automated penetration testing tool.

### 2. Environment

The **world the agent operates in**. Everything the agent interacts with that is external to itself. For the dog, this is the house, the yard, the humans. For a security tool, this could be a target network.

### 3. State (S)

A **snapshot of the current situation**. The state captures everything the agent needs to know about its current position in the environment. For the dog: "I am in the kitchen, the human just said 'sit'." For a pentesting agent: "I have compromised Host A and have user-level access."

### 4. Action (A)

A **choice the agent can make**. The set of all possible moves available to the agent in a given state. For the dog: sit, lie down, bark, run. For a pentesting agent: scan a port, attempt an exploit, escalate privileges, move laterally.

### 5. Reward (R)

The **feedback signal**. A numerical value that tells the agent how good or bad its action was. Positive rewards encourage the behavior; negative rewards (penalties) discourage it. For the dog: +1 treat for sitting, -1 scolding for chewing shoes. For a pentesting agent: +10 for gaining access, -5 for triggering an alert.

```
+-------------------------------------------------------------+
|                    THE FIVE COMPONENTS                       |
+-------------------------------------------------------------+
|                                                             |
|   AGENT          What learns and decides                    |
|   ENVIRONMENT    The world the agent acts in                |
|   STATE          Current snapshot of the situation           |
|   ACTION         What the agent chooses to do               |
|   REWARD         Feedback: good (+) or bad (-)              |
|                                                             |
+-------------------------------------------------------------+
```

---

## The RL Loop

The interaction between agent and environment follows a continuous cycle. At each time step:

1. The agent **observes** the current state
2. The agent **selects** an action
3. The environment **transitions** to a new state
4. The environment returns a **reward**
5. The agent **learns** from this experience
6. Repeat

Here is what that loop looks like:

```
                    +---------------------------+
                    |                           |
                    |       ENVIRONMENT         |
                    |                           |
                    +--+---------------------+--+
                       |                     |
                       |  State (S_t)        |  Reward (R_t)
                       |  "Here is where     |  "Here is how
                       |   you are now"       |   you did"
                       |                     |
                       v                     v
                    +---------------------------+
                    |                           |
                    |         AGENT             |
                    |                           |
                    |   Observes state          |
                    |   Picks action            |
                    |   Learns from reward      |
                    |                           |
                    +-------------+-------------+
                                  |
                                  |  Action (A_t)
                                  |  "I choose to
                                  |   do this"
                                  |
                                  v
                    +---------------------------+
                    |                           |
                    |       ENVIRONMENT         |
                    |   (processes the action,  |
                    |    moves to new state,    |
                    |    calculates reward)     |
                    |                           |
                    +---------------------------+
                                  |
                            Cycle repeats
                            with S_(t+1)
                            and R_(t+1)
```

### A Concrete Example of One Loop Iteration

Let us say our agent is a pentesting bot:

```
Time step t:
  State S_t   = "Connected to target, port 22 open, no credentials"
  Action A_t  = "Attempt brute-force SSH login with common passwords"
  Reward R_t  = +5 (successfully logged in)
  New State   = "SSH session active, user-level shell obtained"

Time step t+1:
  State S_(t+1) = "User-level shell on target"
  Action A_(t+1) = "Run privilege escalation exploit"
  Reward R_(t+1) = +10 (gained root access)
  New State      = "Root shell obtained"
```

---

## Exploration vs Exploitation

This is one of the most important concepts in RL. The agent faces a constant dilemma:

**Exploitation**: Do what you already know works well. Stick with the action that has given you the best reward so far.

**Exploration**: Try something new. Maybe there is a better action you have not discovered yet.

### The Restaurant Analogy

You have a favorite restaurant where the food is always good (exploitation). But there are dozens of restaurants you have never tried. One of them might be even better (exploration). If you always go to your favorite, you will never find that amazing place. But if you always try new restaurants, you waste a lot of meals on bad food.

The key is **balance**.

### Epsilon-Greedy Strategy

The most common way to balance exploration and exploitation:

- Pick a small number called **epsilon** (e.g., 0.1 means 10%)
- With probability **(1 - epsilon)**: choose the best known action (exploit)
- With probability **epsilon**: choose a random action (explore)

```
                  Is random number < epsilon?
                           |
                     +-----+-----+
                     |           |
                    YES          NO
                     |           |
              +------+------+   +--------+--------+
              |  EXPLORE    |   |  EXPLOIT         |
              |  Pick a     |   |  Pick the best   |
              |  random     |   |  action you      |
              |  action     |   |  know so far     |
              +-------------+   +------------------+
```

### Why This Matters in Security

In automated penetration testing:
- **Exploitation** = Use the attack path you know works
- **Exploration** = Try a different exploit or approach that might find a new vulnerability
- Too much exploitation = you miss vulnerabilities
- Too much exploration = you waste time on dead ends and trigger more alerts

---

## How RL Compares to Other Learning Types

| Aspect | Supervised Learning | Unsupervised Learning | Reinforcement Learning |
|---|---|---|---|
| **Training data** | Labeled examples (input -> correct output) | Unlabeled data | No dataset; learns from interaction |
| **Feedback** | Correct answer provided for each example | No feedback | Reward signal (good/bad, not the correct answer) |
| **Goal** | Learn a mapping from inputs to outputs | Find hidden patterns/structure in data | Maximize cumulative reward over time |
| **Analogy** | Studying with an answer key | Sorting a pile of unsorted items | Training a dog with treats |
| **When to use** | Classification, regression, prediction | Clustering, anomaly detection | Sequential decision-making, game playing, robotics |
| **Example** | "This email is spam" / "This is not spam" | "These emails are similar to each other" | "Sending that email got a click; try similar ones" |
| **Learns from** | Past labeled data | Data structure | Trial and error |
| **Security example** | Malware classifier trained on labeled samples | Clustering network traffic to find anomalies | Agent that learns to find attack paths through a network |

### The Key Difference

Supervised and unsupervised learning typically make **one-shot predictions** -- given an input, produce an output. Reinforcement learning makes **sequential decisions** -- each action changes the world and affects what happens next. The consequences of actions unfold over time.

---

## Markov Decision Processes (MDP)

An MDP is the mathematical framework that formalizes RL problems. Do not let the name intimidate you. It is simply a way to describe decision-making situations.

### The Markov Property

The "Markov" part means: **the future depends only on the present state, not on how you got there.**

Think of chess. To decide your next move, you only need to look at the current board position. It does not matter whether you got there in 10 moves or 40 moves. The current position is all that matters.

### Components of an MDP

An MDP is defined by a tuple (S, A, P, R, gamma):

```
+---------------------------------------------------------------+
|                MARKOV DECISION PROCESS (MDP)                  |
+---------------------------------------------------------------+
|                                                               |
|  S  =  Set of all possible states                             |
|         Example: {lobby, hallway, server_room, exit}          |
|                                                               |
|  A  =  Set of all possible actions                            |
|         Example: {move_north, move_south, hack, wait}         |
|                                                               |
|  P  =  Transition probabilities                               |
|         P(s' | s, a) = probability of reaching state s'       |
|         when taking action a in state s                       |
|         Example: P(server_room | hallway, move_north) = 0.8   |
|                                                               |
|  R  =  Reward function                                        |
|         R(s, a) = reward received for taking action a          |
|         in state s                                            |
|         Example: R(server_room, hack) = +100                  |
|                                                               |
|  gamma = Discount factor (between 0 and 1)                    |
|         How much future rewards are worth compared to          |
|         immediate rewards                                     |
|         Example: gamma = 0.9 means future rewards are          |
|         worth 90% of immediate rewards                        |
|                                                               |
+---------------------------------------------------------------+
```

### The Discount Factor (gamma) Explained

Why discount future rewards? Two reasons:

1. **Uncertainty**: The further into the future, the less certain we are about outcomes
2. **Preference**: A reward now is generally more valuable than the same reward later

```
gamma = 0.9

Reward now:          100  x  0.9^0  =  100.0
Same reward in 1 step:  100  x  0.9^1  =   90.0
Same reward in 2 steps: 100  x  0.9^2  =   81.0
Same reward in 5 steps: 100  x  0.9^5  =   59.0
Same reward in 10 steps: 100  x  0.9^10 =   34.9
```

A gamma close to 1 (like 0.99) means the agent is patient and values future rewards almost as much as immediate ones. A gamma close to 0 (like 0.1) means the agent is impatient and mostly cares about immediate rewards.

### Policy and Value

Two more concepts you need to know:

**Policy (pi)**: The agent's strategy. A mapping from states to actions. "When I am in state X, I should do action Y." The goal of RL is to find the **optimal policy** -- the one that maximizes total reward.

**Value Function V(s)**: How good it is to be in a particular state. "Being in the server room is worth more than being in the lobby because I am closer to my goal."

**Action-Value Function Q(s, a)**: How good it is to take a particular action in a particular state. "Being in the hallway and choosing to move north is worth more than choosing to move south." This is what Q-Learning uses (covered in the next guide).

---

## Key RL Algorithms Overview

Here is a map of the main RL algorithms you should know:

```
                    Reinforcement Learning
                            |
              +-------------+-------------+
              |                           |
        Model-Based                  Model-Free
     (agent has a model           (agent learns from
      of the environment)          direct experience)
              |                           |
              |                 +---------+---------+
              |                 |                   |
         Dynamic            Value-Based         Policy-Based
        Programming          Methods              Methods
              |                 |                   |
              |           +-----+-----+             |
              |           |           |             |
         Value         Q-Learning   SARSA      Policy
        Iteration                              Gradient
```

For the HTB exam, focus on:
- **Q-Learning** (off-policy, value-based) -- covered in the next file
- **SARSA** (on-policy, value-based) -- covered in the third file

---

## Security and Offensive Applications

Reinforcement learning has powerful applications in offensive security. Here is how it maps:

### 1. Automated Penetration Testing

An RL agent can learn to perform penetration tests by treating the target network as an environment:

```
+-----------------------------------------------------------------+
|          RL FOR AUTOMATED PENETRATION TESTING                   |
+-----------------------------------------------------------------+
|                                                                 |
|  Agent       =  Pentesting bot                                  |
|  Environment =  Target network                                  |
|  States      =  Current access level, compromised hosts,        |
|                  discovered services                            |
|  Actions     =  Scan, exploit, pivot, escalate, exfiltrate      |
|  Rewards     =  +points for access gained                       |
|                  -points for detection/alerts triggered          |
|                                                                 |
+-----------------------------------------------------------------+
```

The agent learns optimal attack strategies without being explicitly programmed with specific exploits. It discovers which sequences of actions lead to successful compromises.

### 2. Adaptive Attacks

RL agents can adapt their attack strategies in real-time:

- If a firewall blocks port 80, the agent learns to try port 443 or other ports
- If an IDS detects a particular exploit pattern, the agent learns to use evasion techniques
- The agent adapts to the specific defenses of each target

### 3. Automated Vulnerability Discovery

RL can guide fuzzing (sending random/malformed inputs to find bugs):

- **State**: Current program state, code coverage achieved
- **Actions**: Which input to mutate, how to mutate it
- **Reward**: Positive for finding new code paths, very positive for finding crashes
- The agent learns which mutation strategies are most effective

### 4. Network Exploitation Path Discovery

Given a complex network, an RL agent can discover the optimal path from initial access to the target:

```
    +--------+        +--------+        +--------+
    |  Web   | -----> | App    | -----> | DB     |
    | Server |  +2    | Server |  +5    | Server |  +100
    +--------+        +--------+        +--------+
        |                                    ^
        |             +--------+             |
        +-----------> | File   | ------------+
                 +1   | Server |        +3
                      +--------+

    The RL agent learns: Web -> App -> DB = +107 total reward
                    vs: Web -> File -> DB = +104 total reward

    It discovers the more rewarding path automatically.
```

### 5. Evasion and Defense Bypass

RL agents can learn to evade detection systems:

- Modify attack payloads to bypass WAFs
- Time attacks to avoid anomaly detection thresholds
- Learn the detection patterns of specific security tools

### Defensive Applications (Know Both Sides)

RL is also used defensively:
- Automated incident response (learning the best response to each type of attack)
- Dynamic firewall rule adjustment
- Adaptive intrusion detection systems

---

## Key Terminology

| Term | Definition |
|---|---|
| **Agent** | The learner/decision-maker that interacts with the environment |
| **Environment** | The external system the agent interacts with |
| **State** | A representation of the current situation |
| **Action** | A choice the agent can make in a given state |
| **Reward** | Numerical feedback signal (+positive or -negative) |
| **Policy** | The agent's strategy: a mapping from states to actions |
| **Value Function** | Estimate of how good a state (or state-action pair) is |
| **Episode** | One complete run from start to terminal state |
| **Discount Factor (gamma)** | How much future rewards are worth vs immediate rewards (0 to 1) |
| **Exploration** | Trying new/random actions to discover better strategies |
| **Exploitation** | Using the best known action based on current knowledge |
| **Epsilon** | The probability of exploring (choosing a random action) |
| **MDP** | Markov Decision Process -- the formal framework for RL problems |
| **Transition** | Moving from one state to another after taking an action |
| **Terminal State** | A state where the episode ends (goal reached or failure) |
| **Cumulative Reward** | The total reward collected over an entire episode |
| **On-policy** | Learning about the policy currently being used (e.g., SARSA) |
| **Off-policy** | Learning about the optimal policy while following a different one (e.g., Q-Learning) |
| **Model-free** | Learning without a model of the environment's dynamics |
| **Model-based** | Learning with an explicit model of how the environment works |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|---|---|
| Can solve problems where no labeled data exists | Can require millions of interactions to learn (slow training) |
| Handles sequential decision-making naturally | Reward function design is critical and error-prone |
| Adapts to changing environments | Exploration can be dangerous in real-world systems |
| Can discover novel strategies humans have not considered | Difficult to debug -- hard to understand why the agent makes certain decisions |
| No need for a pre-built model of the environment (model-free methods) | Can get stuck in local optima (finds a "good enough" strategy but not the best) |
| Handles delayed rewards (actions whose effects come later) | State space explosion: too many states makes learning infeasible |
| Well-suited for game-like scenarios (attack/defense) | Requires careful tuning of hyperparameters (learning rate, discount factor, epsilon) |
| Can generalize to similar but different environments | Safety concerns: agent might find "creative" but harmful solutions |

---

## Key Takeaways

1. **RL is learning by doing.** The agent learns from trial and error, not from labeled examples or data patterns. It interacts with an environment, takes actions, and receives rewards.

2. **Five core components.** Every RL system has an Agent, Environment, States, Actions, and Rewards. Know what each one means and how to map them to security scenarios.

3. **The exploration-exploitation tradeoff is central.** The agent must balance trying new things (exploration) with using what already works (exploitation). The epsilon-greedy strategy is the most common solution.

4. **RL differs from supervised and unsupervised learning fundamentally.** RL handles sequential decisions with delayed feedback; supervised learning handles one-shot predictions with labeled data; unsupervised learning finds patterns without feedback.

5. **MDPs are the math behind RL.** States, actions, transition probabilities, rewards, and a discount factor. The Markov property says only the current state matters, not history.

6. **Security applications are powerful.** RL can automate penetration testing, discover attack paths, adapt to defenses, guide fuzzing, and learn evasion strategies. It treats offensive security as a sequential decision problem -- which it naturally is.

7. **For the exam, know Q-Learning and SARSA.** These are the two key algorithms covered in the following guides. Q-Learning is off-policy (learns the optimal action regardless of what it actually does). SARSA is on-policy (learns from what it actually does).

---

*Next: [Q-Learning](q-learning.md) -- Learning the optimal action for every situation*
