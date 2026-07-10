# Project Implementation Specification: Blackjack (Twenty-One)
## Technical Rules, State Machine Constraints, and Operational Logic Derived from *Beat the Dealer* by Edward O. Thorp

---

## 1. Architectural Overview & System Configuration

This document provides a rigorous, deterministic specification for implementing a software simulation of Blackjack based on the strict historical Nevada rules formalized by Dr. Edward O. Thorp in *Beat the Dealer*. This blueprint translates textual rules into discrete programmatic states, mathematical boundaries, and algorithmic conditions required to build an autonomous engine or an interactive application.

### 1.1 Global State Parameters
* **Deck Size ($N$):** 52 cards (Single-deck baseline). The system must support scaling to multi-deck shoes ($N \in \{52, 104, 208, 312\}$) as specified in Thorp's appendices, but default execution operates on exactly one deck.
* **Jokers:** Discarded ($0$ items).
* **Card Ranks:** $\mathcal{R} = \{2, 3, 4, 5, 6, 7, 8, 9, 10, J, Q, K, A\}$.
* **Card Suits:** $\mathcal{S} = \{\clubsuit, \diamondsuit, \heartsuit, \spadesuit\}$ (Suits carry zero mathematical weight in game logic or settlement).

### 1.2 Mathematical Card Valuation Function
Let $V(c)$ be the scoring function of card $c$ with rank $r \in \mathcal{R}$:
* For $r \in \{2, 3, 4, 5, 6, 7, 8, 9, 10\}$, $V(c) = r$.
* For $r \in \{J, Q, K\}$, $V(c) = 10$.
* For $r = A$ (Ace), $V(c) \in \{1, 11\}$. 

#### Dynamic Score Resolution Algorithm
Let a hand $H = \{c_1, c_2, \dots, c_k\}$ be an ordered set of cards. 
1. Compute the hard total: $T_{	ext{hard}} = \sum_{c \in H, r 
eq A} V(c) + \sum_{c \in H, r = A} 1$.
2. Count the number of Aces in the hand: $A_{	ext{count}} = |\{c \in H \mid r = A\}|$.
3. If $A_{	ext{count}} > 0$ and $T_{	ext{hard}} + 10 \le 21$:
   * $T_{	ext{final}} = T_{	ext{hard}} + 10$.
   * The hand status is designated as **Soft**.
4. Else:
   * $T_{	ext{final}} = T_{	ext{hard}}$.
   * The hand status is designated as **Hard**.

---

## 2. Theoretical State Machine Lifecycle

The execution sequence must follow a strict, non-reversible synchronous pipeline described by the following state transitions:

```
+------------------+     +----------------+     +------------------------+
|  1. BETTING      | --> |  2. DEALING    | --> |  3. NATURAL CHECK      |
+------------------+     +----------------+     +------------------------+
                                                            |
                                                            v
+------------------+     +----------------+     +------------------------+
|  6. SETTLEMENT   | <-- |  5. DEALER TURN| <-- |  4. PLAYER TURN        |
+------------------+     +----------------+     +------------------------+
```

### 2.1 State 1: Betting Phase (`STATE_BETTING`)
* **Inputs:** Table Minimum ($W_{\min}$), Table Maximum ($W_{\max}$), Player Bankroll ($B$).
* **Constraints:** Player must select an initial wager $W_0$ such that $W_{\min} \le W_0 \le W_{\max}$ and $W_0 \le B$.
* **State Exit Mutation:** Subtract $W_0$ from $B$ and commit it to the active hand's stake pool. Lock the table from further initial bets.

### 2.2 State 2: Dealing Phase (`STATE_DEALING`)
* **Deck State Change:** The deck must be shuffled using a pseudo-random number generator (PRNG) providing a uniform distribution over 52!. A single top card must be popped and moved to an inaccessible state (`BURN_TRACKER`) before dealing.
* **Dealing Order Sequence:** 1. Card 1 $ightarrow$ Player Hand ($H_p$, face down in traditional single-deck, face up programmatically).
  2. Card 2 $ightarrow$ Dealer Hand ($H_d$, face up—designated as **Dealer Upcard**, $u$).
  3. Card 3 $ightarrow$ Player Hand ($H_p$).
  4. Card 4 $ightarrow$ Dealer Hand ($H_d$, face down—designated as **Dealer Hole Card**, $h$).

### 2.3 State 3: Natural Identification Phase (`STATE_NATURAL_CHECK`)
* **Evaluation Condition:** Check $T_{	ext{final}}(H_p)$ and $T_{	ext{final}}(H_d)$ instantly using the initial sets where $|H_p| = 2$ and $|H_d| = 2$.
* **Conditional Branching:**
  * If $T_{	ext{final}}(H_p) = 21$ AND $T_{	ext{final}}(H_d) < 21$: Player wins a **Natural**. Route execution directly to `STATE_SETTLEMENT` with a payout coefficient of $1.5 	imes W_0$.
  * If $T_{	ext{final}}(H_d) = 21$ AND $T_{	ext{final}}(H_p) < 21$: Dealer wins a **Natural**. Route execution directly to `STATE_SETTLEMENT` with a total forfeiture of $W_0$.
  * If $T_{	ext{final}}(H_p) = 21$ AND $T_{	ext{final}}(H_d) = 21$: **Push (Tie)**. Route execution directly to `STATE_SETTLEMENT` with a payout coefficient of $1.0 	imes W_0$ (net gain = 0).
  * If neither has 21: Route execution to `STATE_PLAYER_TURN`.

---

## 3. Player Action Mechanics & Procedural Logic

During `STATE_PLAYER_TURN`, the player interacts with the engine via a set of allowed operators. If a player split pairs, each split hand is evaluated sequentially as an isolated sub-state machine.

### 3.1 Operator: Hit (`OP_HIT`)
* **Action:** Pop one card from the deck and append it to $H_p$.
* **Boundary Validation:** Compute new $T_{	ext{final}}(H_p)$.
  * If $T_{	ext{final}}(H_p) > 21$: Transition hand state to `HAND_BUSTED`. Terminate player turn for this hand immediately. Player forfeits committed wager $W$. Route sequence to `STATE_SETTLEMENT` (or evaluate subsequent split hands).
  * If $T_{	ext{final}}(H_p) \le 21$: Re-evaluate available choices. Player may call `OP_HIT` or `OP_STAND` recursively.

### 3.2 Operator: Stand (`OP_STAND`)
* **Action:** Freeze the current value of $T_{	ext{final}}(H_p)$.
* **State Transition:** Exit `STATE_PLAYER_TURN` and enter `STATE_DEALER_TURN`.

### 3.3 Operator: Double Down (`OP_DOUBLE`)
* **Pre-condition Check:** $|H_p| == 2$ (Strictly permitted on the opening two cards only). Thorp underlines that traditional Nevada rules permit this on **any** two cards, without restriction on their total value.
* **Action:** 1. Append a secondary wager $W_{	ext{double}} = W_0$ to the hand's stake pool. Total active stake becomes $2 	imes W_0$.
  2. Pop exactly one card from the deck and append to $H_p$.
* **Post-condition Check:** Compute final $T_{	ext{final}}(H_p)$.
  * If $T_{	ext{final}}(H_p) > 21$: Hand state transitions to `HAND_BUSTED`, and the combined double wager is forfeit.
* **Forced Transition:** The player is denied any subsequent actions on this hand. Force exit to `STATE_DEALER_TURN` immediately.

### 3.4 Operator: Split Pairs (`OP_SPLIT`)
* **Pre-condition Check:** $|H_p| == 2$ AND $V(c_1) == V(c_2)$. Note that card values must match, meaning a pair of 10s consisting of a Jack and a King is legally eligible for splitting under Thorp's criteria.
* **Action Process:**
  1. Isolate $c_1$ into Hand $A$ ($H_{pA} = \{c_1\}$) and $c_2$ into Hand $B$ ($H_{pB} = \{c_2\}$).
  2. Deduct an additional wager equal to $W_0$ from the bankroll $B$ and assign it to $H_{pB}$.
  3. Pop a card from the deck to complete $H_{pA}$ so that $|H_{pA}| = 2$.
  4. Allow the player to play out $H_{pA}$ entirely (Hitting, Standing, or Doubling) until termination.
  5. Once $H_{pA}$ is resolved, pop a card from the deck to complete $H_{pB}$ so that $|H_{pB}| = 2$.
  6. Allow the player to play out $H_{pB}$ entirely.
* **The Ace Splitting Edge Case (Mandatory Implementation):**
  * If the split rank is an Ace ($r == A$), the dealer appends exactly one card to $H_{pA}$ and exactly one card to $H_{pB}$.
  * The player is completely locked out from calling `OP_HIT` or `OP_DOUBLE` on either hand.
  * If the newly drawn card is a 10-value card, the hand score evaluates to a hard total of 21. **This does not constitute a Natural.** If it wins against the dealer, it yields a standard 1:1 payout, not a 3:2 payout.

### 3.5 Operator: Insurance (`OP_INSURANCE`)
* **Trigger Window:** Executes immediately inside `STATE_NATURAL_CHECK` prior to standard player choices, if and only if $V(u) == 11$ (Dealer upcard is an Ace).
* **Action:** The player can option a side bet $W_{	ext{ins}}$ where $0 < W_{	ext{ins}} \le 0.5 	imes W_0$.
* **Evaluation Matrix:**
  * If $V(h) == 10$ (Dealer has a hidden 10-value card, forming a Blackjack): The side bet wins. Payout = $2 	imes W_{	ext{ins}} + W_{	ext{ins}}$. The main wager $W_0$ is simultaneously lost to the house (unless the player also has a Natural, causing a main wager push). Break sequence and transition instantly to `STATE_SETTLEMENT`.
  * If $V(h) 
eq 10$ (Dealer does not have a Blackjack): The side bet $W_{	ext{ins}}$ is forfeited immediately to the house track. Clear the insurance state and resume standard play within `STATE_PLAYER_TURN`.

---

## 4. Dealer Automation Protocol (Fixed Rule Engine AI)

The dealer's AI requires zero heuristics or neural processing. It is executed via a deterministic loop upon entry to `STATE_DEALER_TURN`.

### 4.1 Automated Script Logic
1. Flip the Dealer Hole Card ($h$) face up to expose the full dealer set $H_d$.
2. Calculate $T_{	ext{final}}(H_d)$ dynamically using the standard score resolution algorithm.
3. While $T_{	ext{final}}(H_d) < 17$:
   * Pop a card from the deck and append to $H_d$.
   * Re-evaluate $T_{	ext{final}}(H_d)$.
4. If $T_{	ext{final}}(H_d) > 21$:
   * Set dealer status to `DEALER_BUSTED`.
   * Exit loop.
5. If $17 \le T_{	ext{final}}(H_d) \le 21$:
   * Freeze dealer score at $T_{	ext{final}}(H_d)$.
   * Exit loop.

### 4.2 Thorp's Stand-on-All-17s Rule Constraint
* **Crucial Edge Case:** If $H_d$ evaluates to a **Soft 17** (e.g., $H_d = \{A, 6\}$ or $\{A, 2, 3, 1\}$ calculating to 11 + 6 = 17), the dealer **MUST STAND**. 
* Do not allow the script to hit a soft 17. Thorp’s mathematical strategy models assume the historical Nevada house standard where the dealer treats hard 17s and soft 17s identically as standing thresholds.

---

## 5. Settlement Logic & Financial Payout Matrix

Once both player and dealer actions are concluded, active wagers are resolved during `STATE_SETTLEMENT`. The processing engine must execute resolution matching the prioritized conditional framework below:

| Sequence Priority | Condition Check Expression | Outcome State | Financial Settlement Resolution |
| :---: | :--- | :--- | :--- |
| **1** | $T_{	ext{final}}(H_p) > 21$ | `PLAYER_BUST` | Player loses wager $W$. House retains funds. |
| **2** | $|H_p| == 2$ and $T_{	ext{final}}(H_p) == 21$ and $T_{	ext{final}}(H_d) 
eq 21$ | `NATURAL_WIN` | Player is paid **3 to 2** ($1.5 	imes W$). Return original stake. |
| **3** | $T_{	ext{final}}(H_p) \le 21$ and $T_{	ext{final}}(H_d) > 21$ | `DEALER_BUST` | Player wins. Paid **1 to 1** ($1.0 	imes W$). Return original stake. |
| **4** | $T_{	ext{final}}(H_p) \le 21$ and $T_{	ext{final}}(H_p) > T_{	ext{final}}(H_d)$ | `PLAYER_HIGH_SCORE` | Player wins. Paid **1 to 1** ($1.0 	imes W$). Return original stake. |
| **5** | $T_{	ext{final}}(H_d) \le 21$ and $T_{	ext{final}}(H_p) < T_{	ext{final}}(H_d)$ | `DEALER_HIGH_SCORE` | Player loses wager $W$. House retains funds. |
| **6** | $T_{	ext{final}}(H_p) == T_{	ext{final}}(H_d)$ | `PUSH` | Tie game. Paid **0** ($0.0 	imes W$). Return original stake to player bankroll. |

### 5.1 The Order of Elimination Principle
A common flaw in faulty state machine design is ignoring the timeline of player busts. If the player calls `OP_HIT` and passes 21, their hand is flagged as `PLAYER_BUST` and their bet is deducted in `STATE_PLAYER_TURN`. If the dealer subsequently enters `STATE_DEALER_TURN` and passes 21, the previously busted player does **NOT** win. The player's loss occurs immediately upon busting, terminating their asset's validity prior to the dealer's lifecycle execution.

---

## 6. Implementation Reference: Thorp's Complete Strategy Indexes

To enable an autonomous agent to run simulated rounds or validate a game player's choices against Thorp's mathematical system, your engine can embed the following discrete logic matrices for hard and soft card totals.

### 6.1 Hard Total Strategy Table
When the player holds a hard total, look up the optimal action based on the visible Dealer Upcard ($u$):

```
Player Total | 2   3   4   5   6   7   8   9   10  A  
-------------|---------------------------------------
17+          | S   S   S   S   S   S   S   S   S   S  
16           | S   S   S   S   S   H   H   H   H   H  
15           | S   S   S   S   S   H   H   H   H   H  
14           | S   S   S   S   S   H   H   H   H   H  
13           | S   S   S   S   S   H   H   H   H   H  
12           | H   H   S   S   S   H   H   H   H   H  
11           | D   D   D   D   D   D   D   D   D   H  
10           | D   D   D   D   D   D   D   D   H   H  
9            | H   D   D   D   D   H   H   H   H   H  
8-           | H   H   H   H   H   H   H   H   H   H  

Legend: H = Hit, S = Stand, D = Double Down (If not permitted, Hit)
```

### 6.2 Soft Total Strategy Table
When the player holds an active Ace valued at 11 points, look up the optimal action against the visible Dealer Upcard ($u$):

```
Player Total | 2   3   4   5   6   7   8   9   10  A  
-------------|---------------------------------------
A,9 (20)     | S   S   S   S   S   S   S   S   S   S  
A,8 (19)     | S   S   S   S   S   S   S   S   S   S  
A,7 (18)     | S   D   D   D   D   S   S   H   H   H  
A,6 (17)     | H   D   D   D   D   H   H   H   H   H  
A,5 (16)     | H   H   D   D   D   H   H   H   H   H  
A,4 (15)     | H   H   D   D   D   H   H   H   H   H  
A,3 (14)     | H   H   H   D   D   H   H   H   H   H  
A,2 (13)     | H   H   H   D   D   H   H   H   H   H  

Legend: H = Hit, S = Stand, D = Double Down (If not permitted, Hit)
```

### 6.3 Pair Splitting Strategy Table
When the player holds an opening pair of equal card ranks, look up the optimal operational choice against the visible Dealer Upcard ($u$):

```
Player Pair  | 2   3   4   5   6   7   8   9   10  A  
-------------|---------------------------------------
A,A          | Sp  Sp  Sp  Sp  Sp  Sp  Sp  Sp  Sp  Sp 
10,10        | M   M   M   M   M   M   M   M   M   M  
9,9          | Sp  Sp  Sp  Sp  Sp  M   Sp  Sp  M   M  
8,8          | Sp  Sp  Sp  Sp  Sp  Sp  Sp  Sp  Sp  Sp 
7,7          | Sp  Sp  Sp  Sp  Sp  Sp  H   H   H   H  
6,6          | Sp  Sp  Sp  Sp  Sp  H   H   H   H   H  
5,5          | M   M   M   M   M   M   M   M   M   M  
4,4          | H   H   H   Sp  Sp  H   H   H   H   H  
3,3          | Sp  Sp  Sp  Sp  Sp  Sp  H   H   H   H  
2,2          | Sp  Sp  Sp  Sp  Sp  Sp  H   H   H   H  

Legend: Sp = Split the Pair, M = Do Not Split (Use standard Hard Total action), H = Hit
```
