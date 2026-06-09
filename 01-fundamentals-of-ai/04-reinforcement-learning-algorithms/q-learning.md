# Q-Learning

## Table of Contents

1. [What is Q-Learning?](#what-is-q-learning)
2. [The Q-Table](#the-q-table)
3. [The Q-Learning Formula](#the-q-learning-formula)
4. [Hyperparameters Explained](#hyperparameters-explained)
5. [Worked Example: Grid World](#worked-example-grid-world)
6. [Step-by-Step Q-Table Updates](#step-by-step-q-table-updates)
7. [The Complete Algorithm](#the-complete-algorithm)
8. [Security and Offensive Applications](#security-and-offensive-applications)
9. [Key Terminology](#key-terminology)
10. [Strengths and Weaknesses](#strengths-and-weaknesses)
11. [Key Takeaways](#key-takeaways)

---

## What is Q-Learning?

### The Cheat Sheet Analogy

Imagine you are playing a video game for the first time. You have no walkthrough, no guide. You just start playing and dying. A lot.

But you are smart. You start keeping a **cheat sheet**. Every time you are in a particular situation (state) and try a particular move (action), you write down how it turned out. Over many attempts, your cheat sheet becomes very accurate:

```
"In the dark room, going left leads to a trap (-10)"
"In the dark room, going right leads to treasure (+50)"
"At the bridge, jumping gets you across (+5)"
"At the bridge, walking makes you fall (-20)"
```

After playing enough times, you can look at your cheat sheet for any situation and immediately know the best move. **That cheat sheet is the Q-Table, and the process of building it is Q-Learning.**

### Formal Definition

Q-Learning is a **model-free, off-policy** reinforcement learning algorithm that learns the value of taking each action in each state. The "Q" stands for "quality" -- it measures the quality of a state-action pair.

- **Model-free**: The agent does not need to know how the environment works internally. It just learns from experience.
- **Off-policy**: The agent learns about the best possible action (the optimal policy) even while it takes non-optimal actions during exploration. It separates "what I do" from "what I learn about."

---

## The Q-Table

The Q-Table is the core data structure of Q-Learning. It is simply a lookup table with:
- **Rows** = every possible state
- **Columns** = every possible action
- **Cell values** = the estimated "quality" (expected future reward) of taking that action in that state

### Initial Q-Table (All Zeros)

When the agent starts learning, it knows nothing. The Q-Table starts with all zeros:

```
+------------------+--------+--------+--------+--------+
|    State \ Action|  Up    | Down   | Left   | Right  |
+------------------+--------+--------+--------+--------+
| (0,0) Start      |  0.00  |  0.00  |  0.00  |  0.00  |
| (0,1)            |  0.00  |  0.00  |  0.00  |  0.00  |
| (0,2)            |  0.00  |  0.00  |  0.00  |  0.00  |
| (1,0)            |  0.00  |  0.00  |  0.00  |  0.00  |
| (1,1) Trap       |  0.00  |  0.00  |  0.00  |  0.00  |
| (1,2)            |  0.00  |  0.00  |  0.00  |  0.00  |
| (2,0)            |  0.00  |  0.00  |  0.00  |  0.00  |
| (2,1)            |  0.00  |  0.00  |  0.00  |  0.00  |
| (2,2) Goal       |  0.00  |  0.00  |  0.00  |  0.00  |
+------------------+--------+--------+--------+--------+
```

### After Training (Learned Values)

After many episodes of exploration, the Q-Table is filled with learned values. The agent can then simply look up its current state, find the action with the highest Q-value, and take that action.

```
+------------------+--------+--------+--------+--------+
|    State \ Action|  Up    | Down   | Left   | Right  |
+------------------+--------+--------+--------+--------+
| (0,0) Start      |  0.00  |  3.20  |  0.00  |  5.10  |
| (0,1)            |  0.00  |  2.10  | -1.00  |  7.30  |
| (0,2)            |  0.00  | -8.50  |  4.20  |  0.00  |
| (1,0)            |  2.50  |  6.80  |  0.00  |  0.50  |
| ...              |  ...   |  ...   |  ...   |  ...   |
+------------------+--------+--------+--------+--------+

For state (0,0): Best action is "Right" (Q = 5.10)
For state (0,1): Best action is "Right" (Q = 7.30)
```

---

## The Q-Learning Formula

Here is the update rule -- the heart of Q-Learning:

```
Q(s, a) <-- Q(s, a) + alpha * [R + gamma * max Q(s', a') - Q(s, a)]
```

This looks intimidating. Let us break it down piece by piece.

### Term-by-Term Breakdown

```
Q(s, a)    =  Current Q-value for state s and action a
               "What I currently think this state-action pair is worth"

alpha      =  Learning rate (0 to 1)
               "How quickly I update my beliefs"

R          =  Reward received after taking action a in state s
               "What I just got"

gamma      =  Discount factor (0 to 1)
               "How much I value future rewards vs immediate rewards"

max Q(s', a')  =  Maximum Q-value in the NEXT state s' across ALL actions
                   "The best possible future from where I land"

[R + gamma * max Q(s', a') - Q(s, a)]  =  The "TD Error" (Temporal Difference)
               "The difference between what I got + best future
                and what I expected"
```

### The Formula in Plain English

```
New estimate = Old estimate + learning_rate * (reality - old_estimate)
```

Or even simpler:

```
"Nudge my old belief a little bit toward what actually happened."
```

### A Numeric Example of One Update

Suppose:
- Current Q(s, a) = 5.0 (our current estimate)
- alpha = 0.1 (learning rate: update slowly)
- R = 10 (we got a reward of 10)
- gamma = 0.9 (we care about future rewards)
- max Q(s', a') = 8.0 (best Q-value in the next state)

```
Q(s, a) = 5.0 + 0.1 * [10 + 0.9 * 8.0 - 5.0]
        = 5.0 + 0.1 * [10 + 7.2 - 5.0]
        = 5.0 + 0.1 * [12.2]
        = 5.0 + 1.22
        = 6.22
```

The Q-value moved from 5.0 toward the "better" estimate of 6.22. Not all the way there -- just a nudge (controlled by the learning rate).

---

## Hyperparameters Explained

### Learning Rate (alpha)

**What it does**: Controls how much each new experience changes the Q-value.

```
alpha = 0.0  -->  Agent never learns (ignores all new information)
alpha = 0.1  -->  Agent learns slowly (small updates, stable)
alpha = 0.5  -->  Agent learns moderately
alpha = 1.0  -->  Agent learns immediately (completely replaces old value)
```

**Analogy**: If someone tells you a restaurant is great, do you immediately change your opinion (alpha = 1.0), or do you wait until multiple people confirm it (alpha = 0.1)?

**Typical value**: 0.1 to 0.5

### Discount Factor (gamma)

**What it does**: Controls how much the agent values future rewards relative to immediate rewards.

```
gamma = 0.0  -->  "I only care about the next reward" (very short-sighted)
gamma = 0.5  -->  "Future rewards are worth half as much"
gamma = 0.9  -->  "I am patient; future rewards matter a lot"
gamma = 1.0  -->  "Future rewards are worth exactly as much as present ones"
```

**Analogy**: Would you rather have $100 today or $110 in a year? Your answer reflects your personal discount factor.

**Typical value**: 0.9 to 0.99

### Epsilon (for Epsilon-Greedy Exploration)

**What it does**: Controls how often the agent explores (random action) vs exploits (best known action).

```
epsilon = 0.0  -->  Always exploit (never try anything new)
epsilon = 0.1  -->  Explore 10% of the time (typical)
epsilon = 0.5  -->  Explore half the time (very exploratory)
epsilon = 1.0  -->  Always explore (pure random behavior)
```

**Common strategy -- Epsilon Decay**: Start with high epsilon (lots of exploration) and gradually decrease it over time (shift to exploitation as the agent learns).

```
Episode 1-100:     epsilon = 1.0   (explore everything)
Episode 101-500:   epsilon = 0.5   (explore half the time)
Episode 501-1000:  epsilon = 0.1   (mostly exploit, little exploration)
Episode 1001+:     epsilon = 0.01  (almost always exploit)
```

---

## Worked Example: Grid World

Let us walk through Q-Learning on a simple 4x4 grid. This is the best way to understand the algorithm.

### The Grid

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

Agent starts at (0,0)
Goal is at (3,3) with reward +100
Traps at (0,2) and (1,3) with reward -10
All other moves give reward -1 (small penalty to encourage efficiency)
Available actions: Up, Down, Left, Right
If the agent tries to move off the grid, it stays in place and gets -1
```

### Setup

```
Hyperparameters:
  alpha  = 0.5   (learning rate)
  gamma  = 0.9   (discount factor)
  epsilon = 0.3  (exploration rate -- 30% random actions)

Q-Table starts with all zeros.
```

---

## Step-by-Step Q-Table Updates

We will trace through part of the agent's first episode, showing each Q-value update.

### Step 1: Start at (0,0), Choose Action

```
State: (0,0)
Available actions: Up, Down, Left, Right
All Q-values are 0, so any action is equally good.
Agent randomly selects: RIGHT
```

The agent moves to (0,1). The reward is -1 (step penalty).

**Q-Update for Q( (0,0), Right ):**

```
Q(s, a) = Q(s, a) + alpha * [R + gamma * max Q(s', a') - Q(s, a)]

Q( (0,0), Right ) = 0 + 0.5 * [-1 + 0.9 * max(0, 0, 0, 0) - 0]
                   = 0 + 0.5 * [-1 + 0.9 * 0 - 0]
                   = 0 + 0.5 * [-1]
                   = -0.5
```

```
Q-Table after Step 1 (only changed cell shown):
+------------------+--------+--------+--------+--------+
|    State \ Action|  Up    | Down   | Left   | Right  |
+------------------+--------+--------+--------+--------+
| (0,0) Start      |  0.00  |  0.00  |  0.00  | -0.50  |
| (all others)     |  0.00  |  0.00  |  0.00  |  0.00  |
+------------------+--------+--------+--------+--------+
```

### Step 2: At (0,1), Choose Action

```
State: (0,1)
All Q-values are 0 for this state.
Agent randomly selects: DOWN
```

The agent moves to (1,1). Reward is -1.

**Q-Update for Q( (0,1), Down ):**

```
Q( (0,1), Down ) = 0 + 0.5 * [-1 + 0.9 * max(0, 0, 0, 0) - 0]
                 = 0 + 0.5 * [-1]
                 = -0.5
```

### Step 3: At (1,1), Choose Action

```
State: (1,1)
All Q-values are 0.
Agent randomly selects: RIGHT
```

The agent moves to (1,2). Reward is -1.

**Q-Update for Q( (1,1), Right ):**

```
Q( (1,1), Right ) = 0 + 0.5 * [-1 + 0.9 * 0 - 0]
                   = -0.5
```

### Step 4: At (1,2), Choose Action

```
State: (1,2)
Agent randomly selects: RIGHT
```

The agent moves to (1,3) -- a TRAP! Reward is -10. Episode might continue, but this hurts.

**Q-Update for Q( (1,2), Right ):**

```
Q( (1,2), Right ) = 0 + 0.5 * [-10 + 0.9 * 0 - 0]
                   = 0 + 0.5 * [-10]
                   = -5.0
```

Now the agent has learned: being at (1,2) and going Right is bad (Q = -5.0).

### Fast Forward: After Many Episodes

After hundreds of episodes, the Q-table converges. Here is what the learned Q-values might look like (showing only the best action per state for clarity):

```
+------+------+------+------+
| DOWN | DOWN | DOWN |  *   |
| 5.10 | 4.20 |-3.80 | TRAP |
+------+------+------+------+
| DOWN |RIGHT | DOWN |  *   |
| 6.80 | 7.30 | 5.60 | TRAP |
+------+------+------+------+
| DOWN |RIGHT |RIGHT | DOWN |
| 8.50 | 9.20 |10.50 |12.00 |
+------+------+------+------+
|RIGHT |RIGHT |RIGHT | GOAL |
| 9.80 |11.00 |15.00 | 100  |
+------+------+------+------+

The agent has learned a path that avoids traps:
(0,0) -> (1,0) -> (2,0) -> (2,1) -> (2,2) -> (2,3) -> (3,3)
                         or a more direct path via the right side
```

### Key Observations from the Training

1. **Q-values near the goal are highest** because they lead directly to the reward
2. **Q-values propagate backward** from the goal through the Bellman equation -- states far from the goal get smaller values because of the discount factor
3. **Traps have negative Q-values** around them, so the agent learns to avoid those areas
4. **Multiple good paths emerge** -- the agent knows several ways to reach the goal

---

## The Complete Algorithm

Here is the full Q-Learning algorithm in pseudocode:

```
ALGORITHM: Q-Learning
=====================

1. Initialize Q-table with zeros (all states x all actions)
2. Set hyperparameters: alpha, gamma, epsilon

3. For each episode:
   a. Reset to start state s
   
   b. While s is not a terminal state:
      
      i.   CHOOSE action a using epsilon-greedy:
           - Generate random number r between 0 and 1
           - If r < epsilon:
               a = random action          (EXPLORE)
           - Else:
               a = action with highest Q(s, a)  (EXPLOIT)
      
      ii.  TAKE action a
           - Observe reward R
           - Observe new state s'
      
      iii. UPDATE Q-table:
           Q(s, a) = Q(s, a) + alpha * [R + gamma * max Q(s', a') - Q(s, a)]
                                                    ^^^^^^^^^^^^^^^^^
                                                    This is the KEY part:
                                                    we use the MAX Q-value
                                                    of the next state
                                                    (best possible action)
      
      iv.  SET s = s'  (move to new state)
   
   c. Optionally decay epsilon (reduce exploration over time)

4. Q-table now contains learned values. Use it to pick the best action in any state.
```

### Why "Off-Policy"?

Notice step (b.iii): when updating, we use `max Q(s', a')` -- the value of the **best** action in the next state. But in step (b.i), we might not actually take that best action (we might explore instead).

This separation is why Q-Learning is called **off-policy**: it learns about the optimal policy (always taking the best action) even while following a different policy (epsilon-greedy with exploration).

This is the key difference from SARSA, which we will cover in the next guide.

---

## Security and Offensive Applications

### 1. Automated Network Exploitation

Q-Learning maps naturally to network penetration testing:

```
+----------------------------------------------------------------+
|              Q-LEARNING FOR NETWORK EXPLOITATION               |
+----------------------------------------------------------------+
|                                                                |
|  States:                                                       |
|    - Network topology knowledge (known hosts, services)        |
|    - Current access level on each host                         |
|    - Credentials discovered                                    |
|    - Security controls detected                                |
|                                                                |
|  Actions:                                                      |
|    - Port scan a host                                          |
|    - Run a specific exploit                                    |
|    - Attempt credential reuse                                  |
|    - Escalate privileges                                       |
|    - Move laterally to another host                            |
|    - Exfiltrate data                                           |
|                                                                |
|  Rewards:                                                      |
|    +1   Discovered new host or service                         |
|    +5   Gained initial access to a host                        |
|    +10  Escalated to admin/root                                |
|    +50  Reached the target asset                               |
|    -5   Triggered an IDS alert                                 |
|    -20  Got blocked by a firewall                              |
|    -1   Each time step (encourages speed)                      |
|                                                                |
+----------------------------------------------------------------+
```

### 2. Finding Optimal Attack Paths

Consider a network with multiple paths to the target:

```
                        INTERNET
                            |
                    +-------+-------+
                    |  Firewall     |
                    +---+-------+---+
                        |       |
                   +----+--+ +--+----+
                   | Web   | | Mail  |
                   | Server| | Server|
                   +----+--+ +--+----+
                        |       |
                   +----+--+ +--+----+
                   | App   | | File  |
                   | Server| | Server|
                   +----+--+ +--+----+
                        |       |
                        +---+---+
                            |
                    +-------+-------+
                    |   DATABASE    |
                    |   (TARGET)    |
                    +---------------+

Q-Table for this scenario:
+--------------------+-----------+-----------+-----------+-----------+
| State              | Scan      | Exploit   | Pivot     | Escalate  |
+--------------------+-----------+-----------+-----------+-----------+
| Internet_only      |  8.5      |  0.0      |  0.0      |  0.0      |
| Web_user_access    |  2.1      |  6.3      |  7.8      |  5.2      |
| Mail_user_access   |  1.5      |  3.2      |  4.1      |  2.0      |
| App_root_access    |  1.0      |  2.0      | 12.5      |  0.0      |
| DB_user_access     |  0.5      |  0.0      |  0.0      | 15.0      |
+--------------------+-----------+-----------+-----------+-----------+

The agent learns: Scan -> Exploit Web -> Pivot to App -> Pivot to DB -> Escalate
This is the optimal attack path based on learned Q-values.
```

### 3. Adaptive Exploit Selection

The Q-table tells the agent which exploit to use against each service:

```
+-----------------+----------+----------+----------+----------+
| Service Found   | Exploit1 | Exploit2 | Exploit3 | No-op    |
+-----------------+----------+----------+----------+----------+
| Apache 2.4.49   |   8.2    |   1.5    |   0.3    |   0.0    |
| OpenSSH 7.6     |   0.5    |   6.7    |   2.1    |   0.0    |
| SMB v1          |   0.2    |   0.1    |   9.5    |   0.0    |
| MySQL 5.7       |   4.3    |   5.8    |   0.9    |   0.0    |
+-----------------+----------+----------+----------+----------+

Agent learns: Use Exploit1 against Apache, Exploit2 against SSH, etc.
```

### 4. Why Q-Learning Specifically for Offensive Security

Q-Learning's off-policy nature is valuable here: the agent can learn the optimal attack strategy even while exploring suboptimal paths. It does not need to actually follow the best path to learn about it. This means:

- During training, the agent can safely explore many attack variations
- The learned Q-table represents the most aggressive/optimal attack path
- Even failed exploits contribute to learning (negative rewards teach what to avoid)

---

## Key Terminology

| Term | Definition |
|---|---|
| **Q-Value** | The estimated total future reward for taking action a in state s; "quality" |
| **Q-Table** | Lookup table storing Q-values for all state-action pairs |
| **Off-policy** | Learning about the optimal policy while following a different (exploration) policy |
| **Bellman Equation** | The recursive equation that Q-Learning is based on; relates current value to future values |
| **TD Error** | Temporal Difference error; the gap between expected and actual reward |
| **Learning Rate (alpha)** | How quickly new information overrides old Q-values (0 to 1) |
| **Discount Factor (gamma)** | How much future rewards are valued vs immediate rewards (0 to 1) |
| **Epsilon-Greedy** | Strategy that explores randomly with probability epsilon, otherwise exploits |
| **Epsilon Decay** | Gradually reducing epsilon over time to shift from exploration to exploitation |
| **Convergence** | When Q-values stabilize and stop changing significantly |
| **Episode** | One complete run from start state to terminal state |
| **Greedy Policy** | Always choosing the action with the highest Q-value (no exploration) |
| **max Q(s', a')** | The maximum Q-value achievable from the next state -- the hallmark of Q-Learning |

---

## Strengths and Weaknesses

| Strengths | Weaknesses |
|---|---|
| Simple to understand and implement | Q-Table grows exponentially with state/action space |
| Guaranteed to converge to optimal policy (with enough exploration) | Cannot handle continuous state spaces (need Deep Q-Learning for that) |
| Off-policy: learns optimal strategy while exploring freely | Requires many episodes to learn in complex environments |
| No model of the environment needed | Overestimates Q-values in stochastic environments |
| Works well for discrete, small-to-medium state spaces | Memory intensive: must store Q-value for every state-action pair |
| Easy to inspect and debug (just look at the Q-table) | Does not scale to real-world state spaces without function approximation |
| Well-suited for automated attack path discovery | Initial random exploration can be wasteful |
| Learns from both successes and failures | Reward shaping is critical and hard to get right |

---

## Key Takeaways

1. **Q-Learning builds a cheat sheet (Q-Table) that tells you the best action in every situation.** It does this through trial and error, updating values based on actual experience.

2. **The Q-Learning formula nudges old estimates toward new reality.** `Q(s,a) = Q(s,a) + alpha * [reward + gamma * best_future - Q(s,a)]`. Each term has a clear meaning.

3. **Three hyperparameters control behavior.** Alpha (learning speed), gamma (future vs present preference), epsilon (exploration vs exploitation balance). Know what each does and typical values.

4. **Q-Learning is off-policy.** It uses `max Q(s', a')` in its update -- it learns about the best possible action even when it takes a different action. This is the key difference from SARSA.

5. **Q-values propagate backward from rewards.** States near the goal get high values first; over many episodes, these values spread to earlier states, creating a gradient the agent follows.

6. **Security application: attack path optimization.** Q-Learning naturally maps to finding the best sequence of actions (scans, exploits, pivots) to reach a target in a network. The Q-table tells you which exploit to use against which service.

7. **Main limitation: scalability.** Q-tables do not scale to large state spaces. Real-world security applications use Deep Q-Networks (DQN) which replace the table with a neural network.

---

*Previous: [Reinforcement Learning Algorithms](reinforcement-learning-algorithms.md)*
*Next: [SARSA](sarsa.md) -- A safer, on-policy alternative*
