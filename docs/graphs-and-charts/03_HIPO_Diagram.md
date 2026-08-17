# Hierarchical Input Process Output (HIPO) Diagram - DinkMatch

> [!NOTE]
> **Diagram Notation Used:** **IBM Standard HIPO Visual Table of Contents (VTOC)**
> - **Functional Blocks:** Rectangles represent functional modules or procedures.
> - **Hierarchical Index:** Numeric prefixes (e.g., 0.0, 1.0, 1.1) define the module level and relate directly to their physical Input-Process-Output (IPO) specification sheets.
> - **Structure:** A top-down hierarchical tree structure displaying software logic decomposition.

This document provides the **HIPO (Hierarchical Input Process Output)** specification for the **DinkMatch** system, including the *Matchmaking Engine* subsystem, *External Camera* device integration, *Line & Foot Fault Detection*, and *Club/Profile management*.

---

## 1. Visual Table of Contents (VTOC)

The VTOC shows the hierarchy of the system functions down to the sub-module level. Each module is assigned a reference number matching the IPO sheets.

```mermaid
graph LR
    classDef root fill:#1e293b,stroke:#00d2ff,stroke-width:2px,color:#fff;
    classDef module fill:#104C81,stroke:#fff,stroke-width:1px,color:#fff;
    classDef sub fill:#3b82f6,stroke:#fff,stroke-width:1px,color:#fff;

    Root["0.0 DinkMatch Kiosk Coordinator"]:::root

    M1["1.0 Profile CheckIn"]:::module
    M2["2.0 Match Solver"]:::module
    M3["3.0 Court Manager"]:::module
    M4["4.0 Score Rating"]:::module
    M6["6.0 DinkMatch AI"]:::module

    Root --> M1
    Root --> M2
    Root --> M3
    Root --> M4
    Root --> M6

    M1 --> M1_1["1.1 Auth Agent"]:::sub
    M1 --> M1_2["1.2 Club Joiner"]:::sub
    M1 --> M1_3["1.3 Profile Editor"]:::sub
    M1 --> M1_4["1.4 CheckIn Validator"]:::sub

    M2 --> M2_1["2.1 Window"]:::sub
    M2 --> M2_2["2.2 Cost"]:::sub
    M2 --> M2_3["2.3 Mode Aging"]:::sub

    M3 --> M3_1["3.1 Monitor Occupancy"]:::sub
    M3 --> M3_2["3.2 TTS Call Announce"]:::sub

    M4 --> M4_1["4.1 Parse Scores"]:::sub
    M4 --> M4_2["4.2 Compute Expectancy"]:::sub
    M4 --> M4_3["4.3 AntiCarry & Capping"]:::sub
    M4 --> M4_4["4.4 Rank Promotion Validator"]:::sub

    M6 --> M6_1["6.1 Stream Interface"]:::sub
    M6 --> M6_2["6.2 CV Analyzer"]:::sub
    M6 --> M6_3["6.3 Recap & MVP"]:::sub
    M6 --> M6_4["6.4 Alert Dispatcher"]:::sub
```

---

## 2. Input-Process-Output (IPO) Sheets

Below are the IPO sheets for the four most critical processing modules of the DinkMatch system.

### IPO Sheet 1.0: Profile & Check-In Controller
*   **Module Name:** `Profile & Check-In Controller`
*   **Module ID:** `1.0`
*   **Purpose:** Manages registration/auth status, profile alterations, club join forms, and check-in/out logic.

| Inputs | Processes | Outputs |
| :--- | :--- | :--- |
| **From Player (UI Form):**<br>- PIN, password, or login credentials<br>- Profile edit fields (Display name, avatar, DUPR ID)<br>- Club UUID to Join<br>- Check-in/out triggers<br><br>**From Database (D2):**<br>- Cached Profile | 1. Capture credentials to authenticate. Run `1.1 Registration & Auth Agent`. If failed, display error.<br>2. Process profile edits via `1.3 Profile Editor`. Save changes locally and flag for background synchronization.<br>3. Process club requests via `1.2 Club Membership Manager`. Connect player profile with selected club UUID.<br>4. Run `1.4 Check-In Validator` to verify active status. If valid, insert player into the active waitlist queue (`D1: Active Queue`). | **To Local Database (D1 / D2):**<br>- Created / edited player profile<br>- Updated club mapping relations<br>- Queue entry added<br><br>**To User Interface (UI):**<br>- Login validation state<br>- Profile changes confirmation toast |

---

### IPO Sheet 2.0: Matchmaking Optimization Scheduler
*   **Module Name:** `Matchmaker Solver`
*   **Module ID:** `2.0`
*   **Purpose:** Automatically builds balanced foursomes for doubles matches from waitlisted players.

| Inputs | Processes | Outputs |
| :--- | :--- | :--- |
| **From Queue Store (D1):**<br>- List of checked-in players ordered by arrival time<br><br>**From Profiles Cache (D2):**<br>- Individual skill ratings<br><br>**From Court Status (D3):**<br>- Count of idle courts<br><br>**From Coordinator Settings:**<br>- Operating Mode (Balanced FIFO, Peak Parity, King of the Court)<br>- Weights ($\alpha$, $\beta$) for cost function | 1. Monitor `D3: Court Status` for vacant courts. If none are idle, halt optimization execution.<br>2. Fetch the top $N$ longest-waiting players from the queue (`2.1 Fetch Queue Sliding Window`).<br>3. Calculate the rating expansion window ($\delta_i$) for each player using logarithmic wait-time aging: $\delta_i = \delta_{base} + \gamma \ln(1 + W_i)$.<br>4. Generate all possible four-player combinations $C(N, 4)$ within the sliding window.<br>5. For each combination, evaluate the joint cost function: $J(M) = \alpha \cdot \sigma^2(R_M) - \beta \cdot \sum W_i$.<br>6. Filter out quartets that violate the active mode constraints (e.g. Peak Parity rating limits).<br>7. Pick the quartet that minimizes the joint cost $J(M)$ and assign them to the vacant court.<br>8. Evict the selected players from `D1: Active Queue`. | **To Local Database (D1):**<br>- Purged Queue Entries (4 players removed)<br><br>**To Court Status (D3):**<br>- Updated Court State (occupancy status set to active, occupant names mapped)<br><br>**To Announcer (3.2):**<br>- Audio TTS announcement trigger payload |

---

### IPO Sheet 4.0: Score & Rating Processor
*   **Module Name:** `Score & Rating Processor`
*   **Module ID:** `4.0`
*   **Purpose:** Calibrates player skill ratings statelessly based on score margins and team dynamics immediately after a match finishes.

| Inputs | Processes | Outputs |
| :--- | :--- | :--- |
| **From Scoreboard UI:**<br>- Match ID<br>- Team 1 Score ($S_1$)<br>- Team 2 Score ($S_2$)<br><br>**From Profiles Cache (D2):**<br>- Teammate and opponent ratings<br><br>**From Peer Feedback UI:**<br>- Sportsmanship badges / Conduct tags | 1. Capture the final scores $S_1$ and $S_2$.<br>2. Calculate point shares: $S'_{T1} = S_1/(S_1 + S_2)$ and $S'_{T2} = S_2/(S_1 + S_2)$.<br>3. Calculate average ratings for Team 1 ($R_{T1}$) and Team 2 ($R_{T2}$).<br>4. Compute logistic win expectancies: $E_{T1} = 1/(1 + 10^{(R_{T2}-R_{T1})/400})$ and $E_{T2} = 1 - E_{T1}$.<br>5. Compute the raw Elo change: $\Delta = K \cdot (S'_T - E_T)$.<br>6. Apply the teammate anti-carry discount: if rating gap exists, scale down the rating gain of the higher-skilled player carrying the match.<br>7. Apply partner adjustment capping to ensure that teammate rating adjustments do not exceed a maximum variance ratio (e.g., 70/30 distribution).<br>8. Apply high K-value calibration factor if a player has "No Rating" status.<br>9. Run `4.4 Rank Promotion Validator` to evaluate whether the new rating crosses any skill category boundaries (1400, 1700, 1900, 2100). If so, update player's `currentRank`, `rankUpdatedAt`, and trigger a promotion/tier-change alert.<br>10. Commit updated skill ratings and ranks to `D2: Player Profiles` and write match logs to `D4: Match History Logs`.<br>11. Vacate the court in `D3: Court Status` and return players to queue. | **To Local Database (D2):**<br>- Updated calibrated ratings and ranks<br><br>**To Match Logs (D4):**<br>- Written Match Record (scores, rating deltas, peer badges, sync flag = false)<br><br>**To Court Status (D3):**<br>- Vacated Court State (occupancy set to idle)<br><br>**To User Interface (UI) / Audio Speakers:**<br>- Live dashboard updates (Leaderboard standings)<br>- Rank Promotion Alert (visual popup card & TTS announcement) |

---

### IPO Sheet 6.0: DinkMatch AI Subsystem Controller
*   **Module Name:** `DinkMatch AI Controller`
*   **Module ID:** `6.0`
*   **Purpose:** Captures video stream frames to identify foot/line violations, and generates session summaries, social media promotional captions, and MVP rankings from session match logs.

| Inputs | Processes | Outputs |
| :--- | :--- | :--- |
| **From External Camera:**<br>- Real-time raw video frames stream<br><br>**From Court Status (D3):**<br>- Active court boundary mapping metrics (kitchen line / baseline coordinates)<br><br>**From Match History Logs (D4):**<br>- Completed match logs array for the active session | 1. Capture stream using `6.1 Camera Stream Interface`. Write raw frame images to cache `D5`.<br>2. Run edge-detection algorithms via `6.2 Computer Vision Analyzer` to identify court lines.<br>3. Track player foot coordinates relative to active court lines. If violation occurs, trigger `6.4 Fault Alert Dispatcher` to flash overlay and play sound alert.<br>4. Trigger `6.3 Session Recap & MVP Generator` at the end of the session to read matches from `D4`. Calculate MVP standings and generate copy-ready social media captions. | **To Local Database (D4):**<br>- Saved Fault Log (Timestamp, Match ID, Fault Type)<br>- Saved AI Session Recap markdown text and MVP stats table<br><br>**To User Interface (UI) / Audio Speakers:**<br>- Audio warning ("Foot fault!")<br>- Red visual warning flash overlay<br>- Copy-ready social media caption and newsletter post panel on Coordinator dashboard |
