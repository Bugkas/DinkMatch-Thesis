# DinkMatch
### *Local-First Multi-Objective Court Scheduling and Zero-Sum Rating Calibration Engine for Multi-Sport Racket Facilities*

---

## 1. Project Context & Brand Architecture

To meet academic evaluation standards and establish a commercialization path, this project maintains a separation of naming conventions:

*   **Academic Subtitle:** *Local-First Multi-Objective Court Scheduling and Zero-Sum Rating Calibration Engine for Multi-Sport Racket Facilities*
*   **Commercial Brand Name:** **DinkMatch** (Dynamic Intelligent Network for Kinetic Open-Play Matchmaking)
*   **Academic Institution:** **Don Mariano Marcos Memorial State University (DMMMSU)**
*   **Target Sports:** Racket and net-based sports featuring drop-in/open-play formats, including **Pickleball**, **Badminton**, **Table Tennis**, **Lawn Tennis**, and **Padel**.

---

## 2. System Overview

### The Core Problem
Traditional physical queuing boards (like whiteboard paddle stacking) result in extreme skill disparities (advanced players mixed with beginners), lobby congestion, and queue manipulation (players shifting paddles to pick better games or dodge weaker partners).

### The DinkMatch Solution
DinkMatch replaces manual whiteboard queuing with an automated, local-first touchscreen kiosk that balances temporal fairness (wait times) against competitive skill parity (minimizing rating variance within matches) in real-time. Operating 100% offline-first, it dynamically matches players and calibrates skill ratings post-match without relying on continuous cloud connectivity.

---

## 3. Core Technical Architecture & Workflows

### Local-First Sync Workflow
The touchscreen kiosk runs local state management in Vue 3/Quasar and persists data using LocalStorage. The Likha ERP Sync Agent syncs local databases asynchronously to the cloud.

```
[ Touchscreen Kiosk Terminal (Vue 3 / Quasar PWA) ]
                       |
                       +---> LocalStorage / SQLite (Local State & Ratings updates)
                       |
             [ Likha ERP Sync Agent ]
                       | (Asynchronous Background WebSockets / REST API Sync)
                       v
         [ Cloud-Hosted Likha ERP Backend ] ---> [ Global Leaderboard / Web Portal ]
```

### State Consistency & Concurrency Constraints
To maintain a robust queue status across multi-admin syncs, the system enforces the following constraints during state serialization (`saveState`):
*   **One Match Per Player:** Enforces that no player appears in more than one active match. Conflicting matches are tombstoned, and players are returned to their original queue positions.
*   **One Match Per Court:** Ensures each physical court has at most one in-progress match.
*   **Concurrency Cap:** Automatically demotes excess in-progress matches to `waiting` state if they exceed the available court capacity (`availableCourts`).
*   **Tombstone Propagation:** Deleted queue entries and player profiles are tracked via timestamps (`deletedAt`) to resolve conflicts using Last-Write-Wins (LWW) semantics.

---

## 4. Matchmaking Engine Implementation

Rather than evaluating combinations across the entire waiting list, the matchmaking engine performs a **two-phase priority-drafting workflow**:

### Phase 1: Prioritized Queue Drafting
The system selects exactly $2 \times \text{Team Size}$ players (4 for doubles) from the queue based on the active queue settings and bracket rules:
1.  **Strict Balance Mode:** Ignores brackets and drafts the closest-rated players from the top of the rating-sorted queue.
2.  **Bracket Priority Mode:** Segregates players into `GENERAL`, `WINNERS`, and `LOSERS` queues:
    *   Drafts a full match from `GENERAL` if available.
    *   Permits leftover `GENERAL` players to overflow and merge with `LOSERS`.
    *   Alternates between `WINNERS` and `LOSERS` brackets using a priority tiebreaker (fewer matches played first, then earlier queue entry time, followed by a toggling alternator flag).

### Phase 2: Combinatorial Pairing Evaluation
Once the 4 players are drafted, the engine evaluates all 3 possible 2v2 pairings (combinations of the 4 players) to form Team A and Team B. Matchups are evaluated according to the selected mode:

*   **`strict_balance` / `fair_balance`:** Minimizes the rating difference (`expectedDifference`) between Team A and Team B.
*   **`balance_first` / `balanced_variety`:** Minimizes a combined cost function incorporating rating difference and partner/opponent repetition penalties:
    $$\text{Cost} = \text{ExpectedDifference} + (w_{\text{partner}} \times \text{PartnerRepeats}) + (w_{\text{opponent}} \times \text{OpponentRepeats})$$
    *   *Settings:* `balance_first` ($w_{\text{partner}}=50, w_{\text{opponent}}=15$); `balanced_variety` ($w_{\text{partner}}=25, w_{\text{opponent}}=8$).
*   **`variety_first`:** Prioritizes matching players who have played together/against each other the least, sorting by fewest partner repeats, then opponent repeats, and finally rating difference.

### Team Rating Metric (Softened Harmonic Mean)
To balance teams, the engine calculates team ratings using a weighted combination of the **Harmonic Mean** (which heavily weights the weakest player to reflect the "weakest link" dynamics of doubles racket sports) and the **Arithmetic Mean**:
$$R_{\text{team}} = 0.6 \times \text{HarmonicMean}(R_{\text{players}}) + 0.4 \times \text{ArithmeticMean}(R_{\text{players}})$$
$$\text{HarmonicMean}(R) = \frac{n}{\sum_{i=1}^n \frac{1}{R_i}}$$

---

## 5. Stateless Point-Margin Elo (PM-Elo) Engine

To avoid database bloat and eliminate historical query dependencies, the rating engine calculates adjustments based purely on current seed ratings and immediate match scores.

### 5.1 Win Expectancy & Rating Pool
For rating adjustments, team strengths are evaluated using the simple arithmetic average of teammate ratings:
$$R_{T1} = \frac{R_{\text{Player1}} + R_{\text{Player2}}}{2}, \quad R_{T2} = \frac{R_{\text{Player3}} + R_{\text{Player4}}}{2}$$

The expected win share of the winning team ($E_W$) is derived using a logistic curve:
$$E_W = \frac{1}{1 + 10^{(R_L - R_W)/400}}$$

A single, integer-rounded rating pool is transferred from the losing team to the winning team, scaled by the score margin and team expectation:
$$\text{Pool} = \text{Round}\left( K \times \left(1 + \lambda \ln(1 + \text{Margin})\right) \times (1 - E_W) \right)$$
*   *Constants:* $K_{\text{doubles}} = 64$, $K_{\text{singles}} = 36$, and Margin Weight ($\lambda$) = $0.15$.

### 5.2 Zero-Sum Distribution & Constraints
The pool points are distributed proportionally among teammates using the **largest-remainder method** to ensure updates remain strictly zero-sum:

*   **Winning Team Allocation (Anti-Carry Penalty):**
    Each winning player's allocation weight ($W_i$) is calculated based on their individual win expectancy, discounted if they carry a much weaker partner:
    $$W_i = \text{BaseCredit}_i \times \text{Penalty}_i$$
    $$\text{BaseCredit}_i = 1 - \frac{1}{1 + 10^{(R_L - R_i)/400}}$$
    $$\text{Penalty}_i = \begin{cases} \max\left(0.1, 1 - \frac{0.5 \times \text{PartnerGap}}{400}\right) & \text{if } R_i < R_{\text{partner}} \\ 1 & \text{otherwise} \end{cases}$$
*   **Losing Team Allocation (Underdog Blame):**
    The blame distribution allocates losses based on expected win probability:
    $$W_j = \frac{1}{1 + 10^{(R_W - R_j)/400}}$$
*   **Partner Ratio Cap:**
    To prevent extreme skill gaps from producing massive rating changes, teammate weights are capped at a maximum ratio of $2.0$:
    $$\text{CappedWeight}_i = \max\left(W_i, \frac{\max(W_{\text{team}})}{2.0}\right)$$

---

## 6. Repository Diagram Assets

All system architecture diagrams are located in the `graphs-and-charts/` directory:

| Diagram Name | Asset Path | Description |
| :--- | :--- | :--- |
| **DFD Level 0 (Context)** | `graphs-and-charts/pngs/01_DFD_lvl0.jpg` | Context diagram showing kiosk boundary and external entity interfaces. |
| **DFD Level 1** | `graphs-and-charts/pngs/01_DFD_lvl1.png` | Decomposes the kiosk into 6 core processes and 4 local data stores. |
| **DFD Level 2 (Matchmaking)**| `graphs-and-charts/pngs/01_DFD_lvl2 Process 2.0.png` | Deep-dive decomposition of the Matchmaking Optimization Solver (Process 2.0). |
| **DFD Level 2 (Video Fault)**| `graphs-and-charts/pngs/01_DFD_lvl2 Process 7.0 .png` | Exception handling and telemetry workflows for hardware/video faults. |
| **Structured Chart** | `graphs-and-charts/pngs/02_Structured_Chart.png` | High-level architectural partition of execution modules. |
| **HIPO VTOC Diagram** | `graphs-and-charts/pngs/03_HIPO Diagram.png` | Visual Table of Contents mapping hierarchical functional components. |
| **Entity-Relationship (ERD)** | `graphs-and-charts/pngs/06_ERD.png` | Database schemas and relation mappings for local and cloud data structures. |

---

## 7. Empirical Validation Protocol

To validate rating calibration accuracy and matchmaking convergence, a comparative **organizer-versus-algorithm** validation framework is applied.

1.  **Expert Ground Truth ($R_{\text{expert}}$):** Before play sessions begin, the resident club organizer manually ranks the participant cohort ($N \approx 100$) based on historical observations and tournament results.
2.  **Algorithm Leaderboard ($R_{\text{system}}$):** Players run games scheduled by the matchmaking kiosk, and system-calibrated ratings are extracted to produce a sorted leaderboard.
3.  **Spearman's Rank Correlation Coefficient:**
    $$\rho = 1 - \frac{6 \sum d_i^2}{n(n^2 - 1)}$$
    *(where $d_i = R_{\text{expert}, i} - R_{\text{system}, i}$)*
4.  **Target Success Metrics:**
    *   At $N = 100$ and significance level $\alpha = 0.05$, the critical correlation value is $\rho_{\text{crit}} = 0.197$ to reject the null hypothesis $H_0$.
    *   The project defines a success target of **$\rho \ge 0.75$**, indicating highly reliable alignment with expert ground-truth rankings.

---

## 8. Research & Development Team

**Student Researchers (DMMMSU College of Computer Science):**

*   **Boado, Jericho T.** - Student Leader / Representative
*   **Agbunag, Joana Andrae J.** - Student Researcher
*   **Boac, Chelsie Assyria P.** - Student Researcher
*   **Boadilla, Maerose Joscel Czarinah V.** - Student Researcher
*   **Geneta, Karina Joyce B.** - Student Researcher
*   **Sison, Darmie A.** - Student Researcher

**Thesis Advisor:**
*   **Charlie S. Marzan, PhD** - Dean, CCS / Thesis Committee Chair

**Industry Collaborators & Technical Advisors (ZyberLab Solutions Inc.):**
*   **Trinmar Boado** - Chief Technology Officer (CTO) / Technical Mentor & Collaborator
*   **Lovely Boado** - Chief Financial Officer (CFO) / Project Consultant & Advisor
