# Structure Chart - DinkMatch

> [!NOTE]
> **Diagram Notation Used:** **Constantine & Yourdon Structure Chart Notation**
> - **Modules:** Rectangular boxes represent software modules (functions/procedures/objects).
> - **Module Call Connection:** Directed lines indicate that a parent module calls or executes a child module.
> - **Data Couple:** Arrows with an empty circle tail represent variables or data structures passed between modules.
> - **Control Flag:** Arrows with a filled circle tail represent control signals or flags (e.g., status flags) passed between modules.

This document presents the **Structure Chart** for the **DinkMatch** system, including the *Matchmaking Engine* subsystem, *External Camera* device integration, *Line & Foot Fault Detection*, and *Club/Profile management*.

---

## 1. Modular Hierarchy Diagram

```mermaid
graph LR
    classDef root fill:#1e293b,stroke:#00d2ff,stroke-width:2px,color:#fff;
    classDef module fill:#104C81,stroke:#fff,stroke-width:1px,color:#fff;
    classDef sub fill:#3b82f6,stroke:#fff,stroke-width:1px,color:#fff;

    Root["0.0 Main Kiosk Coordinator<br>(PlayPage / ClubPage)"]:::root

    %% Level 1 Modules
    M1["1.0 Profile & Check-In Controller"]:::module
    M2["2.0 Matchmaking Scheduler"]:::module
    M3["3.0 Court Manager & Announcer"]:::module
    M4["4.0 Score & Rating Processor"]:::module
    M5["5.0 Cloud Sync Middleware"]:::module
    M6["6.0 DinkMatch AI Subsystem"]:::module

    Root --> M1
    Root --> M2
    Root --> M3
    Root --> M4
    Root --> M5
    Root --> M6

    %% Level 2 Sub-modules for M1
    M1_1["1.1 Registration & Auth Agent"]:::sub
    M1_2["1.2 Club Membership Manager"]:::sub
    M1_3["1.3 Profile Editor"]:::sub
    M1_4["1.4 Check-In / Out Validator"]:::sub
    M1 --> M1_1
    M1 --> M1_2
    M1 --> M1_3
    M1 --> M1_4

    %% Level 2 Sub-modules for M2
    M2_1["2.1 Queue Manager"]:::sub
    M2_2["2.2 Combinatorial Solver"]:::sub
    M2_3["2.3 Mode & Aging Evaluator"]:::sub
    M2 --> M2_1
    M2 --> M2_2
    M2 --> M2_3

    %% Level 2 Sub-modules for M3
    M3_1["3.1 Court Occupancy Monitor"]:::sub
    M3_2["3.2 Browser Audio Announcer (TTS)"]:::sub
    M3 --> M3_1
    M3 --> M3_2

    %% Level 2 Sub-modules for M4
    M4_1["4.1 Score Entry Validator"]:::sub
    M4_2["4.2 PM-Elo Rating Engine"]:::sub
    M4_3["4.3 Peer Feedback Collector"]:::sub
    M4_4["4.4 Rank Promotion Validator"]:::sub
    M4 --> M4_1
    M4 --> M4_2
    M4 --> M4_3
    M4 --> M4_4

    %% Level 2 Sub-modules for M5
    M5_1["5.1 Queue/Match Offline Cache"]:::sub
    M5_2["5.2 Likha SDK Cloud Client"]:::sub
    M5 --> M5_1
    M5 --> M5_2

    %% Level 2 Sub-modules for M6 (DinkMatch AI)
    M6_1["6.1 Camera Stream Interface"]:::sub
    M6_2["6.2 Computer Vision Analyzer"]:::sub
    M6_3["6.3 Session Recap & MVP Generator"]:::sub
    M6_4["6.4 Fault Alert Dispatcher"]:::sub
    M6 --> M6_1
    M6 --> M6_2
    M6 --> M6_3
    M6 --> M6_4
```

---

## 2. Module Descriptions & Couples

The table below explains the data couples (D) and control couples (C) flowing between the modules in the hierarchy:

| Parent Module | Child Module | Data Couples (D) / Control Couples (C) | Description |
| :--- | :--- | :--- | :--- |
| **0.0 Main Kiosk** | **1.0 Profile Controller** | **Out (D):** Credentials, Profile Edits, Club Join Request<br>**In (D):** Hydrated Profile, Club Membership data<br>**In (C):** Registration/Login Status Flag (Success/Fail) | Coordinates registrations, logins, profile edits, club joins, and checks-in/out. |
| **1.0 Profile Controller** | **1.1 Registration & Auth** | **Out (D):** Username, Password/PIN, Initial Rating<br>**In (C):** Auth Token, Session Status Flag | Manages player sign-up, sign-in, and sign-out states. |
| **1.0 Profile Controller** | **1.2 Club Membership Manager** | **Out (D):** Club UUID, Player ID<br>**In (C):** Club Join Status (Approved/Pending) | Processes club search and enrollment logic. |
| **1.0 Profile Controller** | **1.3 Profile Editor** | **Out (D):** Avatar edits, display names, DUPR IDs | Commits profile changes to local cache and sync queues. |
| **0.0 Main Kiosk** | **2.0 Matchmaking Scheduler** | **Out (D):** Active Queue List, Court Settings<br>**In (D):** Matched Quartet, Assigned Court ID<br>**In (C):** Match Success Flag (Boolean) | Directs the automated matchmaking search and court assignment loop. |
| **2.0 Matchmaking Scheduler** | **2.1 Queue Manager** | **Out (D):** Player ID, Target Queue<br>**In (D):** Position in Queue | Controls adding, removing, sorting, and positioning players in the waitlist. |
| **0.0 Main Kiosk** | **3.0 Court Manager & Announcer** | **Out (D):** Match Pairings, Court ID<br>**In (C):** Speech Complete Flag | Alerts players of their court assignments via Text-to-Speech audio and visual displays. |
| **0.0 Main Kiosk** | **4.0 Score & Rating Processor** | **Out (D):** Raw Scores, Player Ratings, Teammate IDs<br>**In (D):** Rating Deltas, Updated Profiles<br>**In (C):** Score Saved Flag (Boolean) | Processes score logging, rating calibration, and feedback collection. |
| **4.0 Score Processor** | **4.2 PM-Elo Rating Engine** | **Out (D):** Game Score Margin, Player Ratings<br>**In (D):** Calibrated rating changes | Evaluates win expectancy, point shares, and carry discount rules to update ratings. |
| **4.0 Score Processor** | **4.4 Rank Promotion Validator** | **Out (D):** New Rating, Old Rating<br>**In (D):** Rank promotion alert payload, new tier string | Compares previous rating with updated rating to detect tier transitions and trigger rank-up audio/visual alerts. |
| **0.0 Main Kiosk** | **5.0 Cloud Sync Middleware** | **Out (D):** Unsynced Local Matches, Stats changes<br>**In (D):** Global profiles updates<br>**In (C):** Online Status Flag (Boolean) | Ensures local-first offline continuity with asynchronous background sync. |
| **0.0 Main Kiosk** | **6.0 DinkMatch AI Subsystem** | **Out (D):** Active Court mapping, line coordinates calibration, completed match logs<br>**In (D):** Foot/Line fault event logs, AI session recap markdown, MVP stats<br>**In (C):** Fault alert trigger signal, Recap generation status | Coordinates real-time camera feeds, computer vision analysis, and generative recap updates. |
| **6.0 DinkMatch AI Subsystem** | **6.1 Camera Stream Interface** | **Out (D):** Device ID, Frame Buffer request<br>**In (D):** Raw image frames stream | Interfaces directly with the physical **External Camera** hardware. |
| **6.0 DinkMatch AI Subsystem** | **6.2 Computer Vision Analyzer** | **Out (D):** Raw image frames, mapped line segments<br>**In (C):** Line cross detection event flag (Foot Fault / Out) | Runs edge-detection and object-tracking algorithms to detect foot placement. |
| **6.0 DinkMatch AI Subsystem** | **6.3 Session Recap & MVP Generator** | **Out (D):** Completed Match logs array for the session<br>**In (D):** AI-generated Recap markdown, MVP rankings | Generates natural language summaries of matching sessions, calculates MVPs, and synthesizes copy-ready social media engagement captions / promotional club posts. |
| **6.0 DinkMatch AI Subsystem** | **6.4 Fault Alert Dispatcher** | **Out (D):** Fault details, court number<br>**In (C):** Sound effect trigger, text overlay update | Renders real-time visual warnings on screens and pushes fault signals to speakers. |
