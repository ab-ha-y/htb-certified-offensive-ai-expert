# SARSA (State-Action-Reward-State-Action)

## Table of Contents

1. [What is SARSA?](#what-is-sarsa)
2. [SARSA vs Q-Learning: The Core Difference](#sarsa-vs-q-learning-the-core-difference)
3. [The SARSA Update Rule](#the-sarsa-update-rule)
4. [On-Policy vs Off-Policy Explained](#on-policy-vs-off-policy-explained)
5. [Worked Example: Grid World (Same Grid, Different Behavior)](#worked-example-grid-world)
6. [Step-by-Step SARSA Updates](#step-by-step-sarsa-updates)
7. [Comparing the Learned Paths](#comparing-the-learned-paths)
8. [The Complete SARSA Algorithm](#the-complete-sarsa-algorithm)
9. [When to Use SARSA vs Q-Learning](#when-to-use-sarsa-vs-q-learning)
10. [Security and Offensive Applications](#security-and-offensive-applications)
11. [Key Terminology](#key-terminology)
12. [Strengths and Weaknesses](#strengths-and-weaknesses)
13. [Key Takeaways](#key-takeaways)

---

## What is SARSA?

### The Name Says It All

SARSA stands for **State-Action-Reward-State-Action**. The name literally describes the five pieces of information it uses in each update:

```
S   -->  Current State         "Where am I?"
A   -->  Action Taken          "What did I do?"
R   -->  Reward Received       "What did I get?"
S'  -->  Next State            "Where did I end up?"
A'  -->  Next Action (chosen)  "What will I ACTUALLY do next?"
                                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                THIS is what makes SARSA different
                                from Q-Learning
```

### The Cautious Traveler Analogy

Imagine two people navigating a city with both safe streets and dangerous alleyways:

**Q-Learning person**: Plans the fastest route on the map, assuming they will always take the optimal path. They chart a route that cuts through a dark alley because it is the shortest path. "The best move from that corner is to go through the alley."

**SARSA person**: Plans a route based on how they **actually** walk -- including their own tendency to sometimes wander randomly (explore). Since they know they might accidentally stumble into a dangerous alley (due to exploration), they learn to stay further away from alleyways altogether. They take a longer but safer route.

Both eventually reach their destination, but SARSA learns a **safer** path because it accounts for the fact that the agent is imperfect and sometimes makes random moves.

---

## SARSA vs Q-Learning: The Core Difference

The difference comes down to a single change in the update formula:

```
Q-LEARNING UPDATE:
Q(s, a) = Q(s, a) + alpha * [R + gamma * max Q(s', a') - Q(s, a)]
                                          ^^^
                                     Uses the BEST possible
                                     action in the next state
                                     (even if agent does not
                                      actually take it)


SARSA UPDATE:
Q(s, a) = Q(s, a) + alpha * [R + gamma * Q(s', a') - Q(s, a)]
                                          ^^^^^^^^
                                     Uses the ACTUAL action
                                     the agent will take in
                                     the next state
                                     (including random exploration
                                      moves)
```

That is it. One term is different. But it changes the behavior dramatically.

### ASCII Comparison Diagram

```
TIME STEP t          TIME STEP t+1

Q-LEARNING:
+---------+          +---------+
| State s |--act a-->| State s'|---> What is the BEST action?
+---------+  get R   +---------+     (max over all Q-values)
                                     Agent might not take it.
                                     "What SHOULD I do?"


SARSA:
+---------+          +---------+
| State s |--act a-->| State s'|---> What action WILL I take?
+---------+  get R   +---------+     (using epsilon-greedy, same
                                      as how I chose action a)
                                     Agent WILL take this action.
                                     "What WILL I do?"
```

### Why This Matters

When the agent explores (takes random actions), those random actions might lead to bad outcomes (falling into traps). Q-Learning ignores this risk in its updates -- it assumes the best action will always be taken in the future. SARSA includes this risk -- it knows the agent will sometimes take random (and potentially bad) actions.

This makes SARSA more **conservative**. It learns to avoid states where a random exploration move could be disastrous, even if the optimal move from that state is fine.

---

## The SARSA Update Rule

```
Q(s, a) = Q(s, a) + alpha * [R + gamma * Q(s', a') - Q(s, a)]
```

### Term-by-Term Breakdown

```
Q(s, a)     =  Current Q-value for state s, action a
                "What I think this state-action pair is worth"

alpha       =  Learning rate (0 to 1)
                "How fast I update my estimates"

R           =  Reward received after taking action a in state s
                "What I just got"

gamma       =  Discount factor (0 to 1)
                "How much I care about future rewards"

Q(s', a')   =  Q-value for the NEXT state s' and the NEXT action a'
                that the agent WILL ACTUALLY TAKE
                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                NOT the best action. The action actually chosen
                (which might be a random exploration action).

[R + gamma * Q(s', a') - Q(s, a)]  =  TD Error
                "How different reality is from my expectation"
```

### The Critical Difference in One Line

| Algorithm | What goes in the update | Name for this |
|---|---|---|
| Q-Learning | `max Q(s', a')` -- best possible next action | Off-policy |
| SARSA | `Q(s', a')` -- actual next action chosen | On-policy |

---

## On-Policy vs Off-Policy Explained

This is a concept that trips up many beginners. Here is a clear explanation:

### Off-Policy (Q-Learning)

The agent learns about **one policy** (the optimal policy -- always take the best action) while **following a different policy** (epsilon-greedy with random exploration).

Think of it like this: "I explore randomly to gather data, but when I update my notes, I assume I will always make the perfect move going forward."

### On-Policy (SARSA)

The agent learns about **the same policy it is actually following**. It updates its Q-values based on what it will actually do, including random exploration moves.

Think of it like this: "I explore randomly to gather data, and when I update my notes, I honestly account for the fact that I will sometimes make random moves going forward."

```
+-------------------------------------------------------------------+
|                                                                   |
|   OFF-POLICY (Q-Learning)        ON-POLICY (SARSA)                |
|                                                                   |
|   "Learn the ideal behavior"     "Learn about MY behavior"        |
|                                                                   |
|   Behavior policy: epsilon-greedy Behavior policy: epsilon-greedy |
|   Target policy:   greedy (best)  Target policy:   epsilon-greedy |
|                    ^different                      ^same           |
|                                                                   |
|   Optimistic about the future     Realistic about the future      |
|   Finds the optimal path          Finds a safe path               |
|   Risk-seeking                    Risk-averse                     |
|                                                                   |
+-------------------------------------------------------------------+
```

---

## Worked Example: Grid World

We use the **same grid** as the Q-Learning guide to show how SARSA behaves differently.

### The Grid (Same as Before)

```
+------+------+------+------+
|      |      |      |      |
| (0,0)|      | (0,2)| (0,3)|
| START|      | TRAP |      |
|      |      | -10  |      |
+------+------+------+------+
|      |      |      |      |
| (1,0)|      | (1,2)| (1,3)|
|      |      |      | TRAP |
|      |      |      | -10  |
+------+------+------+------+
|      |      |      |      |
| (2,0)| (2,1)| (2,2)| (2,3)|
|      |      |      |      |
|      |      |      |      |
+------+------+------+------+
|      |      |      |      |
| (3,0)| (3,1)| (3,2)| (3,3)|
|      |      |      | GOAL |
|      |      |      | +100 |
+------+------+------+------+

Same setup:
  Goal at (3,3): reward +100
  Traps at (0,2) and (1,3): reward -10
  All other moves: reward -1
  Actions: Up, Down, Left, Right

Hyperparameters:
  alpha   = 0.5
  gamma   = 0.9
  epsilon = 0.3
```

---

## Step-by-Step SARSA Updates

The key difference from Q-Learning: we must choose the next action BEFORE doing the update.

### Step 1: Start at (0,0), Choose First Action

```
State: (0,0)
All Q-values are 0.
Using epsilon-greedy (epsilon = 0.3):
  Random number = 0.45 (> 0.3, so EXPLOIT)
  All Q-values are 0, so pick randomly among tied values.
  Action chosen: RIGHT
```

### Step 2: Take Action, Observe Result, Choose NEXT Action

```
Take action RIGHT from (0,0):
  New state s' = (0,1)
  Reward R = -1

NOW (before updating), choose the NEXT action a' from (0,1):
  Using epsilon-greedy (epsilon = 0.3):
  Random number = 0.15 (< 0.3, so EXPLORE)
  Random action chosen: DOWN

  a' = DOWN   <-- This is what SARSA uses in the update
                   Q-Learning would use max Q(s', a') instead
```

### Step 3: SARSA Update

```
Q(s, a) = Q(s, a) + alpha * [R + gamma * Q(s', a') - Q(s, a)]

Q( (0,0), Right ) = 0 + 0.5 * [-1 + 0.9 * Q( (0,1), Down ) - 0]
                   = 0 + 0.5 * [-1 + 0.9 * 0 - 0]
                   = 0 + 0.5 * [-1]
                   = -0.5

So far, identical to Q-Learning because all Q-values are still 0.
The difference emerges as values diverge.
```

### Step 4: Move to Next State, Continue

```
Now we are at (0,1) and we ALREADY know our action: DOWN
(We chose it in Step 2 for the update.)

Take action DOWN from (0,1):
  New state s' = (1,1)
  Reward R = -1

Choose NEXT action a' from (1,1):
  epsilon-greedy: random number = 0.72 (EXPLOIT)
  All Q-values are 0, random among ties.
  a' = RIGHT

SARSA Update:
Q( (0,1), Down ) = 0 + 0.5 * [-1 + 0.9 * Q( (1,1), Right ) - 0]
                 = 0 + 0.5 * [-1 + 0 - 0]
                 = -0.5
```

### Where the Difference Emerges: Near Traps

Let us jump ahead to a more interesting scenario. Suppose after many episodes, the agent is at state **(1,2)**, which is adjacent to a trap at (1,3).

**With Q-Learning:**
```
State: (1,2)
Q-Learning update uses: max Q(s', a')
The best action from (1,2) might be DOWN (toward the goal path).
Q-Learning says: "The best thing from (1,2) is to go DOWN. 
                  Q( (1,2), Down ) is pretty good."
Q-Learning is OKAY being at (1,2) because the OPTIMAL move avoids the trap.
```

**With SARSA:**
```
State: (1,2)
SARSA update uses: Q(s', a') where a' is the ACTUAL next action
With epsilon = 0.3, there is a 30% chance a' = random action.
If random action = RIGHT, agent goes to (1,3) TRAP! Reward = -10.
SARSA includes this risk in its Q-value estimate.
So Q-values near traps are LOWER with SARSA than with Q-Learning.
The agent learns: "Being at (1,2) is risky because I might 
                   accidentally step into the trap."
```

### Numeric Comparison at (1,2)

Suppose after training, the Q-values at (1,2) are:

```
Q-LEARNING Q-values at (1,2):
  Up: -2.0  |  Down: 6.5  |  Left: 3.2  |  Right: -5.0
  Best action: DOWN (Q = 6.5)
  Agent WILL pass through (1,2) on its optimal path.

SARSA Q-values at (1,2):
  Up: -3.5  |  Down: 2.8  |  Left: 1.5  |  Right: -7.2
  Best action: DOWN (Q = 2.8) -- but this is lower than alternatives
  Agent AVOIDS (1,2) entirely because all values are lower.
  It takes a wider path around the traps.
```

---

## Comparing the Learned Paths

After training with both algorithms on the same grid:

```
Q-LEARNING LEARNED PATH:           SARSA LEARNED PATH:
(takes the shortest/optimal path,  (takes a wider/safer path,
 passing near traps)                avoiding areas near traps)

+------+------+------+------+      +------+------+------+------+
|  *   |      | TRAP |      |      |  *   |      | TRAP |      |
| (0,0)|      |      |      |      | (0,0)|      |      |      |
|  |   |      |      |      |      |  |   |      |      |      |
+--+---+------+------+------+      +--+---+------+------+------+
|  v   |      |      | TRAP |      |  v   |      |      | TRAP |
| (1,0)|      |      |      |      | (1,0)|      |      |      |
|  |   |      |      |      |      |  |   |      |      |      |
+--+---+------+------+------+      +--+---+------+------+------+
|  v   |      |      |      |      |  v   |      |      |      |
| (2,0)| ---> | ---> | ---> |      | (2,0)| ---> |(2,2) |      |
|      |(2,1) |(2,2) |(2,3) |      |      |(2,1) |  |   |      |
+------+------+------+--+---+      +------+------+--+---+------+
|      |      |      |  v   |      |      |      |  v   |      |
|      |      |      | GOAL |      |      |      | (3,2)| ---> |
|      |      |      |      |      |      |      |      | GOAL |
+------+------+------+------+      +------+------+------+------+

Q-Learning: (0,0)->(1,0)->(2,0)    SARSA: (0,0)->(1,0)->(2,0)
  ->(2,1)->(2,2)->(2,3)->(3,3)      ->(2,1)->(2,2)->(3,2)->(3,3)

Q-Learning passes through (2,3)     SARSA avoids (2,3) -- too close
which is directly below the trap    to the trap at (1,3). One random
at (1,3). Optimal but risky if      move up from (2,3) = disaster.
the agent explores.                 Safer but slightly longer path.
```

### Why the Paths Differ

Q-Learning's path through (2,3) is technically optimal (shortest to goal). But (2,3) is directly below the trap at (1,3). If the agent is still exploring (epsilon > 0) and randomly moves UP from (2,3), it hits the trap.

SARSA knows this. Because it uses the actual next action (which might be random) in its updates, it has learned that (2,3) is dangerous during exploration. So it routes around it.

**Once epsilon reaches 0 (no more exploration), both algorithms converge to the same optimal path.** The difference only matters during training or when epsilon > 0.

---

## The Complete SARSA Algorithm

```
ALGORITHM: SARSA
================

1. Initialize Q-table with zeros (all states x all actions)
2. Set hyperparameters: alpha, gamma, epsilon

3. For each episode:
   a. Reset to start state s
   b. Choose action a using epsilon-greedy from Q(s, .)
      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      NOTE: We choose the FIRST action before entering the loop
   
   c. While s is not a terminal state:
      
      i.   TAKE action a
           - Observe reward R
           - Observe new state s'
      
      ii.  CHOOSE next action a' using epsilon-greedy from Q(s', .)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
           KEY STEP: Choose the next action NOW, before updating
      
      iii. UPDATE Q-table:
           Q(s, a) = Q(s, a) + alpha * [R + gamma * Q(s', a') - Q(s, a)]
                                                     ^^^^^^^^
                                                     Uses Q-value of
                                                     the ACTUAL next
                                                     action a' (not max)
      
      iv.  SET s = s', a = a'
           ^^^^^^^^^^^^^^^^^^
           Both state AND action carry forward
   
   d. Optionally decay epsilon

4. Q-table now contains learned values.
```

### Side-by-Side Algorithm Comparison

```
Q-LEARNING                           SARSA
=========                            =====

For each episode:                    For each episode:
  s = start state                      s = start state
                                       a = epsilon_greedy(s)  <-- extra
  While not terminal:                  While not terminal:
    a = epsilon_greedy(s)                (a already chosen)
    Take a, get R, s'                    Take a, get R, s'
                                         a' = epsilon_greedy(s')
    UPDATE with max Q(s',.)              UPDATE with Q(s', a')
    s = s'                               s = s', a = a'
```

---

## When to Use SARSA vs Q-Learning

| Factor | Q-Learning | SARSA |
|---|---|---|
| **Policy type** | Off-policy | On-policy |
| **Update uses** | Best possible next action | Actual next action |
| **Risk tolerance** | Risk-neutral (assumes optimal future) | Risk-averse (accounts for exploration mistakes) |
| **Path found** | Shortest/optimal path | Safer path (avoids risky areas) |
| **Near dangerous states** | Will pass close if it is optimal | Avoids areas where random moves are costly |
| **Convergence speed** | Often faster (targets optimal directly) | Can be slower (conservative updates) |
| **During exploration** | Can suffer large penalties near traps | Avoids penalties by staying away |
| **After training (epsilon=0)** | Optimal path | Also converges to optimal path |
| **Best for** | Finding the absolute best strategy | Finding a strategy that is safe during learning |
| **Security use case** | Maximizing exploitation efficiency | Avoiding crashes and detection during testing |
| **When actions are costly** | Might cause damage while learning | Learns cautiously, avoids costly mistakes |
| **Implementation** | Slightly simpler (no need to pre-choose next action) | Slightly more complex (must choose next action before update) |

### Quick Decision Guide

```
Use Q-LEARNING when:
  - You want the absolute optimal strategy
  - The environment is a simulation (mistakes are cheap)
  - You can afford to trigger alarms during training
  - You want faster convergence to the best policy
  - You are training in a lab, deploying in the real world

Use SARSA when:
  - Safety during learning matters
  - Mistakes are expensive or irreversible
  - You want to avoid dangerous states during training
  - The agent is learning in a live/production environment
  - Triggering alerts or crashing systems is unacceptable
  - You want a policy that is robust to occasional random actions
```

---

## Security and Offensive Applications

### SARSA for Cautious Automated Security Testing

The key advantage of SARSA in security: **it learns strategies that are safe even when the agent occasionally makes random mistakes.** This is critical when:

1. Testing production systems that must stay online
2. Performing security assessments where detection must be minimized
3. Situations where a wrong action could crash a critical service

### Scenario: Testing a Production Web Application

```
+-------------------------------------------------------------------+
|           SARSA FOR PRODUCTION SECURITY TESTING                   |
+-------------------------------------------------------------------+
|                                                                   |
|  States:                                                          |
|    - Current page/endpoint being tested                           |
|    - Authentication level                                         |
|    - Number of alerts triggered so far                            |
|    - Server response time (is it slowing down?)                   |
|                                                                   |
|  Actions:                                                         |
|    - Send benign probe request                                    |
|    - Send SQL injection payload (mild)                            |
|    - Send SQL injection payload (aggressive)                      |
|    - Send XSS payload                                             |
|    - Attempt authentication bypass                                |
|    - Wait/back off                                                |
|                                                                   |
|  Rewards:                                                         |
|    +10   Found a vulnerability                                    |
|    +2    Got interesting error message                             |
|    -1    Normal response (no finding)                              |
|    -15   Triggered WAF/IDS alert                                  |
|    -50   Server became unresponsive                               |
|    -100  Server crashed                                           |
|                                                                   |
+-------------------------------------------------------------------+
```

### How SARSA Behaves Differently Here

```
Scenario: Agent is testing endpoint /api/users

Q-LEARNING behavior:
  "The best action is aggressive SQL injection -- it has the highest
   Q-value because it finds vulns fastest."
  
  But with epsilon = 0.1, 10% of the time the agent will take a
  random action. If it accidentally sends an aggressive payload
  to /api/admin/shutdown, the server crashes (-100).
  
  Q-Learning does NOT account for this risk in its Q-values.
  It still rates aggressive payloads highly near dangerous endpoints.

SARSA behavior:
  "Aggressive SQL injection finds vulns, BUT sometimes I take
   random actions. If I am near a dangerous endpoint and take
   a random aggressive action, the server could crash."
  
  SARSA accounts for this risk. Near dangerous endpoints, it
  learns to use milder payloads. The Q-value for aggressive
  actions near sensitive endpoints is LOWER because SARSA
  factors in the possibility of accidental aggressive actions.

Result:
  Q-Learning: Faster testing, but might crash the server
  SARSA:      Slower testing, but keeps the server running
```

### SARSA for Stealth

SARSA's risk-aversion maps naturally to stealthy testing:

```
+-----------------------------------------------------------------+
|                  SARSA LEARNS STEALTH                            |
+-----------------------------------------------------------------+
|                                                                 |
|  States near detection threshold:                               |
|                                                                 |
|  Q-Learning:     "Best action is another scan. Do it."          |
|                  (Ignores risk of epsilon-random burst scan      |
|                   that triggers the IDS)                        |
|                                                                 |
|  SARSA:          "I am near the detection threshold. Even       |
|                   though the best action is to scan, my         |
|                   random actions might trigger a burst.          |
|                   I will back off and wait."                    |
|                                                                 |
+-----------------------------------------------------------------+
```

### Comparison Table: Security Context

| Scenario | Q-Learning | SARSA |
|---|---|---|
| Lab/CTF environment | Preferred (find vulns fast) | Works but unnecessary caution |
| Production pentest | Risky (might cause outages) | Preferred (safer testing) |
| Red team with stealth requirement | May trigger alerts | Naturally avoids detection |
| Automated vulnerability scanner | Good for thorough coverage | Good for stable scanning |
| Fuzzing a critical service | Might crash the target | Adapts to avoid crashes |
| Network enumeration | Aggressive, fast | Throttled, careful |

### Real-World Application: Safe Automated Penetration Testing Framework

```
TRAINING PHASE (in a lab copy of the target):
  - Use Q-Learning to find all possible attack paths (aggressive)
  - Use SARSA to find safe attack paths (conservative)
  - Compare the two sets of paths

DEPLOYMENT PHASE (against real target):
  - Start with SARSA-learned paths (safe)
  - Monitor system health continuously
  - If system is stable, gradually introduce Q-Learning paths
  - If system shows stress, fall back to SARSA paths

This hybrid approach gets the best of both worlds:
  Q-Learning provides the complete attack surface map
  SARSA provides the safe execution plan
```

---

## Key Terminology

| Term | Definition |
|---|---|
| **SARSA** | State-Action-Reward-State-Action; an on-policy RL algorithm |
| **On-policy** | The agent learns about the policy it is actually following |
| **Off-policy** | The agent learns about a different (usually optimal) policy |
| **Q(s', a')** | Q-value of the actual next action chosen (used in SARSA update) |
| **max Q(s', a')** | Q-value of the best next action (used in Q-Learning update) |
| **Risk-averse** | Tendency to avoid situations where bad outcomes are possible |
| **Conservative policy** | A strategy that favors safe actions over optimal-but-risky ones |
| **TD(0)** | Temporal Difference learning with one-step lookahead (both SARSA and Q-Learning are TD(0)) |
| **Expected SARSA** | Variant that uses the expected Q-value over all possible next actions (weighted by policy probabilities) |
| **Behavior policy** | The policy the agent actually follows to select actions |
| **Target policy** | The policy the agent is trying to learn about |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|---|---|
| Safer during training (accounts for exploration risk) | Can be slower to converge than Q-Learning |
| Better for real-world/production environments | Finds suboptimal paths when epsilon > 0 |
| Naturally avoids dangerous states | More sensitive to the choice of epsilon |
| Accounts for the agent's own imperfect behavior | Slightly more complex to implement |
| Good for online learning (learning while doing) | Learned policy depends on exploration strategy |
| Robust to occasional random actions | May be overly cautious in safe environments |
| Suitable for risk-sensitive domains (security testing) | Same scalability limitations as Q-Learning (table-based) |
| Consistent: what it learns matches what it does | Needs more episodes to find optimal policy |

---

## Key Takeaways

1. **SARSA = State-Action-Reward-State-Action.** It uses the actual next action in its update, not the best possible next action. This single change makes it on-policy and more conservative.

2. **On-policy means learning about what you actually do.** SARSA's Q-values reflect the policy the agent is actually following (including random exploration moves). Q-Learning's Q-values reflect the optimal policy (ignoring exploration mistakes).

3. **SARSA is the cautious cousin of Q-Learning.** It avoids states where random exploration could lead to disaster. Q-Learning will happily walk near a cliff because the optimal move is safe; SARSA stays away because it might randomly step off the edge.

4. **The formulas differ by one term.** Q-Learning: `max Q(s', a')`. SARSA: `Q(s', a')`. Everything else is identical. Know this difference for the exam.

5. **When epsilon = 0, both converge to the same policy.** The difference only matters during training or when the agent still explores. In pure exploitation mode, SARSA and Q-Learning behave identically.

6. **Security application: safe production testing.** SARSA is ideal when you cannot afford to crash systems, trigger alerts, or cause damage during the learning process. Use Q-Learning in labs for finding all attack paths; use SARSA in production for safe execution.

7. **Hybrid approach is powerful.** Train with Q-Learning to discover all possible strategies, then use SARSA for safe deployment. This gives you both comprehensive coverage and operational safety.

---

*Previous: [Q-Learning](q-learning.md) -- The off-policy algorithm for finding optimal strategies*
*Start: [Reinforcement Learning Algorithms](reinforcement-learning-algorithms.md) -- Overview of RL*
