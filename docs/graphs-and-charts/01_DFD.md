# Data Flow Diagrams (DFD) - DinkMatch

> [!NOTE]
> **Diagram Notation Used:** **Yourdon & Coad Notation**
> - **External Entities:** Rectangles represent users or systems outside the DinkMatch system boundary.
> - **Processes:** Circles/ellipses represent system logic that transforms data inputs.
> - **Data Stores:** Double-parallel horizontal lines indicate persistent or volatile data repositories.
> - **Data Flows:** Directed arrows represent the direction of data elements moving through the system.

This document outlines the Data Flow Diagrams (DFD) for the **DinkMatch** system, including the *Matchmaking Engine* subsystem, *External Camera* device integration, *Line & Foot Fault Detection*, and *Club/Profile management*.

---

## 1. Level 0 Context Diagram
The Level 0 Context Diagram establishes the boundary of the local kiosk application, detailing its interactions with the external entities: the **Player**, the **Session Coordinator**, **Likha ERP** (cloud backend), and the newly integrated **External Camera (Device)**.

```mermaid
graph TD
    classDef entity fill:#1e293b,stroke:#00d2ff,stroke-width:2px,color:#fff;
    classDef process fill:#104C81,stroke:#fff,stroke-width:2px,color:#fff;
    
    Player["Player (Guest / Registered)"]:::entity
    Coordinator["Session Coordinator (Admin)"]:::entity
    LikhaERP["Likha ERP (Cloud Backend)"]:::entity
    Camera["«device» External Camera"]:::entity
    
    System(("DinkMatch Touchscreen Kiosk System<br>(Process 0.0)")):::process
    
    %% Player Flows
    Player -- "Registration, PIN, Check-In, Scores, Peer Feedback" --> System
    System -- "Waitlist Status, Court Calls, Leaderboard, Personal Dashboards, Rank Promotion Alerts" --> Player
    
    %% Coordinator Flows
    Coordinator -- "Club Setup, Court Mapping, Swaps, Matchmaking Settings" --> System
    System -- "Spearman Correlation, Court Stats, Fault Logs" --> Coordinator
    
    %% Likha ERP Flows
    LikhaERP -- "Verified Profiles & Configuration Synced Audit" --> System
    System -- "Asynchronous Match Logs & Profile Updates" --> LikhaERP

    %% Camera Flow
    Camera -- "Real-Time Video Frames" --> System
```

---

## 2. Level 1 DFD: Functional Decomposition
The Level 1 DFD decomposes the system boundary into seven core operational processes and five local data stores (Active Queue, Player Profiles, Court Status, Match History, and the new Video Frames Cache).

```mermaid
graph TD
    classDef entity fill:#1e293b,stroke:#00d2ff,stroke-width:2px,color:#fff;
    classDef process fill:#104C81,stroke:#fff,stroke-width:1px,color:#fff;
    classDef datastore fill:#16a34a,stroke:#fff,stroke-width:1px,color:#fff;

    Player["Player"]:::entity
    Coordinator["Session Coordinator"]:::entity
    LikhaERP["Likha ERP"]:::entity
    Camera["«device» External Camera"]:::entity

    %% Processes
    P1(("1.0 Manage Check-in<br>& Profiles")):::process
    P2(("2.0 Matchmaker<br>Solver")):::process
    P3(("3.0 Execution<br>& Alerts")):::process
    P4(("4.0 PM-Elo<br>Calibration")):::process
    P5(("5.0 Sync<br>Agent")):::process
    P6(("6.0 Validation<br>Report")):::process
    P7(("7.0 Video Capture<br>& Fault Detector")):::process
    P8(("8.0 Generate AI<br>Session Recap")):::process

    %% Data Stores
    D1[("D1: Active Queue (Volatile)")]:::datastore
    D2[("D2: Player Profiles (Local Cache)")]:::datastore
    D3[("D3: Court Status (Local State)")]:::datastore
    D4[("D4: Match History Logs (Persistent)")]:::datastore
    D5[("D5: Video Frames Cache (Volatile)")]:::datastore

    %% Flows for P1
    Player -- "Register, PIN, Club Join, Edit Profile" --> P1
    P1 -- "Read/Write Profile" --> D2
    P1 -- "Insert Checked-In Player" --> D1

    %% Flows for P2
    Coordinator -- "Matchmaking Rules" --> P2
    D1 -- "Get Longest-Waiting Players" --> P2
    D2 -- "Retrieve Player Ratings" --> P2
    P2 -- "Assign Quartet" --> D3
    P2 -- "Remove Matched Players" --> D1

    %% Flows for P3
    D3 -- "Read Occupancy" --> P3
    P3 -- "TTS Court-Call Alerts & Visual Display" --> Player

    %% Flows for P4
    Player -- "Scores & Peer Feedback" --> P4
    P4 -- "Read Seed Ratings" --> D2
    P4 -- "Write Calibrated Ratings & Ranks" --> D2
    P4 -- "Record Completed Match & Deltas" --> D4
    P4 -- "Mark Court Vacant" --> D3
    P4 -- "Rank Promotion Alert (A/V & TTS)" --> Player

    %% Flows for P5
    D4 -- "Retrieve Unsynced Matches" --> P5
    P5 -- "Sync Match Payloads" --> LikhaERP
    LikhaERP -- "Global Profile Updates" --> P5
    P5 -- "Update Local Profiles" --> D2

    %% Flows for P6
    Coordinator -- "Request Validation" --> P6
    D4 -- "Retrieve Rating Deltas" --> P6
    P6 -- "Spearman Correlation vs. Pre-Rankings" --> Coordinator

    %% Flows for P7 (New Video & Fault Detection)
    Camera -- "Raw Video Stream" --> P7
    P7 -- "Cache Frame Buffers" --> D5
    P7 -- "Log Detected Faults" --> D4
    P7 -- "Audio/Visual Fault Alerts" --> Player
    D3 -- "Fetch Active Courts Mapping" --> P7

    %% Flows for P8 (AI Session Recap)
    Coordinator -- "Request Session Recap" --> P8
    D4 -- "Read Completed Matches" --> P8
    P8 -- "Write Session Recap & MVP Stats" --> D4
    P8 -- "Display AI Session Recap & MVP Stats" --> Player
    P8 -- "Provide Copy-Ready Social Captions & Club Recaps" --> Coordinator
```

---

## 3. Level 2 DFD: Process 2.0 (Matchmaking Solver)
This diagram decomposes **Process 2.0 (Matchmaking Solver)** to show how the queue is partitioned, evaluated using a multi-objective cost function, and scheduled.

```mermaid
graph TD
    classDef process fill:#104C81,stroke:#fff,stroke-width:1px,color:#fff;
    classDef datastore fill:#16a34a,stroke:#fff,stroke-width:1px,color:#fff;
    classDef external fill:#1e293b,stroke:#00d2ff,stroke-width:2px,color:#fff;

    D1[("D1: Active Queue")]:::datastore
    D2[("D2: Player Profiles")]:::datastore
    D3[("D3: Court Status")]:::datastore
    AudioAlert["Audio / Visual Alerts"]:::external

    P2_1(("2.1 Fetch Queue<br>Sliding Window")):::process
    P2_2(("2.2 Evaluate<br>Combinatorial Cost")):::process
    P2_3(("2.3 Apply Mode<br>Constraints & Aging")):::process
    P2_4(("2.4 Dispatch Match<br>& Update State")):::process

    %% Flows
    D1 -- "Get Check-Ins" --> P2_1
    P2_1 -- "Longest-Waiting N Players (N=8/12)" --> P2_2
    D2 -- "Retrieve Skill Ratings" --> P2_2
    P2_2 -- "All C(N, 4) Combinations with Costs J(M)" --> P2_3
    P2_3 -- "Adjust Thresholds (Logarithmic Aging)" --> P2_3
    P2_3 -- "Optimal Quartet Selected" --> P2_4
    P2_4 -- "Remove Players from Queue" --> D1
    P2_4 -- "Mark Court Active" --> D3
    P2_4 -- "Trigger Announcements" --> AudioAlert
```

---

## 4. Level 2 DFD: Process 7.0 (Video Capture & Fault Detection)
This new diagram decomposes **Process 7.0 (Video Capture & Fault Detection)**, showing how frame streams are captured, analyzed for foot and line violations, and dispatched.

```mermaid
graph TD
    classDef process fill:#104C81,stroke:#fff,stroke-width:1px,color:#fff;
    classDef datastore fill:#16a34a,stroke:#fff,stroke-width:1px,color:#fff;
    classDef external fill:#1e293b,stroke:#00d2ff,stroke-width:2px,color:#fff;

    Camera["«device» External Camera"]:::external
    Player["Player / Audio Speaker"]:::external
    D3[("D3: Court Status")]:::datastore
    D4[("D4: Match History Logs")]:::datastore
    D5[("D5: Video Frames Cache")]:::datastore

    P7_1(("7.1 Capture Frame<br>& Buffer Stream")):::process
    P7_2(("7.2 CV Court Boundary<br>Calibration")):::process
    P7_3(("7.3 Track Player Feet<br>& Collision Detect")):::process
    P7_4(("7.4 Log Event &<br>Dispatch Warning")):::process

    %% Flows
    Camera -- "Real-Time Raw Video Frames" --> P7_1
    P7_1 -- "Write to Frame Buffer" --> D5
    D3 -- "Read Active Match & Court Layout" --> P7_2
    P7_2 -- "Map Line/Corner Coordinates" --> P7_3
    D5 -- "Read Frames" --> P7_3
    P7_3 -- "Identify Foot Faults / Out-of-Bounds" --> P7_4
    P7_4 -- "Save Fault Clip & Meta" --> D4
    P7_4 -- "Trigger Sound Alert & Overlay Visual" --> Player
```
