# Data Dictionary - DinkMatch

This document provides the **Data Dictionary** for the **DinkMatch** system databases, specifying field types, constraints, and operational descriptions for every logical storage entity, including the *Matchmaking Engine* subsystem, *External Camera* device integration, *Line & Foot Fault Detection*, and *Club/Profile management*.

---

## 1. Entity: CLUB (Session Metadata)
Tracks club details and operational checkpoints.

| Attribute Name | Data Type | Key / Constraint | Description / Business Rules |
| :--- | :--- | :--- | :--- |
| `uuid` | String | Primary Key (PK), Not Null | Unique identifier mapped to the Likha ERP cloud database schema. |
| `name` | String | Not Null | Display name of the racket facility (e.g. "Manila Pickleball Club"). |
| `settings` | JSON | Nullable | Session configurations: `alpha`, `beta`, `gamma`, `baseThreshold`, and `mode`. |
| `completedMatchesResetAt` | Integer | Default: 0 | Epoch millisecond timestamp of the last session history wipe boundary. |
| `lastModified` | Integer | Not Null | Epoch timestamp of the last administrative state change. Used for hard reset sync. |
| `settingsUpdatedAt` | Integer | Default: 0 | Timestamp of the last settings change. Resolves settings conflicts using LWW. |

---

## 2. Entity: PLAYER (Player Profile)
Caches player identifiers, skill ratings, credentials, and historical counters.

| Attribute Name | Data Type | Key / Constraint | Description / Business Rules |
| :--- | :--- | :--- | :--- |
| `username` | String | Primary Key (PK), Not Null | Unique alphanumeric identifier for player login and display. |
| `userId` | String | Foreign Key (FK), Nullable | Cloud user authentication account ID synced from Likha ERP. |
| `firstName` | String | Nullable | Player's first name. |
| `lastName` | String | Nullable | Player's last name. |
| `duprId` | String | Nullable | External DUPR rating account link (e.g. "US-12345"). |
| `rating` | Float | Default: 3.0 | Skill level rating. Initialized to seed rating; clamped at a floor of 100. |
| `ratingStatus` | String | Default: "No Rating" | Status state: `"No Rating"` (for unrated calibration boost) or `"Calibrated"`. |
| `currentRank` | String | Default: "Beginner" | Dynamic skill tier category: `"Beginner"`, `"Intermediate"`, `"Advanced"`, `"Expert"`, `"Pro"`. |
| `rankUpdatedAt` | Integer | Default: 0 | Timestamp of the last rank promotion or tier change. |
| `matchesPlayed` | Integer | Default: 0 | Total number of matches completed in the active session scope. |
| `wins` | Integer | Default: 0 | Aggregate match wins. |
| `losses` | Integer | Default: 0 | Aggregate match losses. |
| `createdAt` | Integer | Not Null | Epoch millisecond timestamp of player profile registration. |
| `updatedAt` | Integer | Not Null | Epoch timestamp of the last profile update. Used for whole-object LWW. |
| `deletedAt` | Integer | Nullable | Timestamp flag indicating soft deletion. Used as sync tombstone. |
| `statsUpdatedAt` | Integer | Default: 0 | Timestamp of the last stats update. Overrides overall LWW to protect stats. |
| `ratingUpdatedAt` | Integer | Default: 0 | Timestamp of the last Elo update. Overrides overall LWW to protect rating. |
| `history` | JSON | Nullable | Opponent and teammate counters to calculate match novelty constraints. |

---

## 3. Entity: QUEUE_ENTRY (Lobby Waitlist)
Maintains lobby check-in details.

| Attribute Name | Data Type | Key / Constraint | Description / Business Rules |
| :--- | :--- | :--- | :--- |
| `queueId` | String | Primary Key (PK), Not Null | Unique identifier for the queue check-in event. |
| `username` | String | Foreign Key (FK), Not Null | References `PLAYER.username`. |
| `queueType` | String | Not Null | Values: `"GENERAL"`, `"WINNERS"`, or `"LOSERS"`. Maps player sub-queues. |
| `enteredAt` | Integer | Not Null | Sort priority value. Can be set to `0` (jump-to-front fairness sentinel). |
| `queuedAt` | Integer | Not Null | Actual timestamp of queue insertion. Used to calculate wait times. |
| `createdAt` | Integer | Not Null | Creation timestamp. |
| `updatedAt` | Integer | Not Null | Last change timestamp. |
| `deletedAt` | Integer | Nullable | Sync tombstone timestamp indicating player left queue (checked-out/matched). |

---

## 4. Entity: COURT (Physical State)
Tracks active court assignments and boundary calibration details.

| Attribute Name | Data Type | Key / Constraint | Description / Business Rules |
| :--- | :--- | :--- | :--- |
| `courtId` | String | Primary Key (PK), Not Null | Unique identifier (e.g. "court_1", "court_2"). |
| `name` | String | Not Null | Display name (e.g. "Court 1 - Challenger"). |
| `status` | String | Default: "idle" | Current occupancy status. Values: `"idle"` or `"active"`. |
| `activeMatchId` | String | Foreign Key (FK), Nullable | References `ACTIVE_MATCH.matchId`. Null if idle. |
| `occupancyUpdatedAt`| Integer | Default: 0 | Epoch timestamp of the last status change. |

---

## 5. Entity: ACTIVE_MATCH (Drafted Schedule)
Details matches generated by the solver that are waiting to be played or currently in progress.

| Attribute Name | Data Type | Key / Constraint | Description / Business Rules |
| :--- | :--- | :--- | :--- |
| `matchId` | String | Primary Key (PK), Not Null | Unique local match identifier. |
| `queueSource` | String | Not Null | Category indicator (e.g. `"GENERAL"`, `"WINNERS"`, `"LOSERS"`, `"MANUAL"`). |
| `teamA` | JSON | Not Null | JSON array of strings containing usernames for Team A (1 or 2 players). |
| `teamB` | JSON | Not Null | JSON array of strings containing usernames for Team B (1 or 2 players). |
| `expectedDifference`| Float | Default: 0.0 | Expected average rating difference between Team A and Team B. |
| `status` | String | Default: "waiting" | Execution state: `"waiting"` (lobby call) or `"playing"` (on court). |
| `startedAt` | Integer | Nullable | Epoch timestamp of match commencement on court. |
| `createdAt` | Integer | Not Null | Creation timestamp. |
| `updatedAt` | Integer | Not Null | Last change timestamp. Used for match conflict resolution. |
| `deletedAt` | Integer | Nullable | Sync tombstone indicating match was completed or cancelled. |
| `generatedBy` | String | Nullable | Solver identifier (e.g. `"auto_matchmaking"`, `"manual_swap"`). |
| `matchmakingMode` | String | Nullable | Active operating mode when generated (e.g. `"fair_balance"`). |

---

## 6. Entity: COMPLETED_MATCH (Historical Ledger)
Chronicles match outcomes and score margins.

| Attribute Name | Data Type | Key / Constraint | Description / Business Rules |
| :--- | :--- | :--- | :--- |
| `matchId` | String | Primary Key (PK), Not Null | References original `ACTIVE_MATCH.matchId`. |
| `matchType` | String | Not Null | Values: `"singles"` or `"doubles"`. |
| `teamA` | JSON | Not Null | JSON array of player snapshot objects containing usernames, DUPR IDs, and ratings. |
| `teamB` | JSON | Not Null | JSON array of player snapshot objects containing usernames, DUPR IDs, and ratings. |
| `teamAScore` | Integer | Not Null | Final score recorded for Team A. |
| `teamBScore` | Integer | Not Null | Final score recorded for Team B. |
| `startedAt` | Integer | Nullable | Match start timestamp. |
| `completedAt` | Integer | Not Null | Epoch timestamp of score submission. |
| `updatedAt` | Integer | Not Null | Last update timestamp. Used for sync conflict resolution. |
| `club` | String | Foreign Key (FK), Not Null | References `CLUB.uuid`. |

---

## 7. Entity: PEER_FEEDBACK (Sportsmanship Log)
Collects post-match peer feedback reviews.

| Attribute Name | Data Type | Key / Constraint | Description / Business Rules |
| :--- | :--- | :--- | :--- |
| `feedbackId` | String | Primary Key (PK), Not Null | Unique identifier. |
| `matchId` | String | Foreign Key (FK), Not Null | References `COMPLETED_MATCH.matchId`. |
| `giverUsername` | String | Foreign Key (FK), Not Null | References `PLAYER.username`. |
| `receiverUsername` | String | Foreign Key (FK), Not Null | References `PLAYER.username`. |
| `badgeType` | String | Not Null | Badge review category (e.g., `"Great Hustle"`, `"Toxicity"`). |
| `createdAt` | Integer | Not Null | Timestamp of submission. |

---

## 8. Entity: FAULT_EVENT (Computer Vision Violations)
Logs line-calls, foot faults, and boundaries infractions detected by the Matchmaking Engine video capture subsystem.

| Attribute Name | Data Type | Key / Constraint | Description / Business Rules |
| :--- | :--- | :--- | :--- |
| `faultId` | String | Primary Key (PK), Not Null | Unique identifier generated upon detection. |
| `matchId` | String | Foreign Key (FK), Not Null | References active or completed `matchId`. |
| `offendingPlayer` | String | Foreign Key (FK), Nullable | References `PLAYER.username`. Set to `"none"` if ball bounce fault. |
| `faultType` | String | Not Null | Values: `"kitchen_volley_fault"`, `"serve_foot_fault"`, or `"out_of_bounds_call"`. |
| `confidence` | Float | Not Null | Algorithmic confidence level (0.0 to 1.0) of CV detection decision. |
| `timestamp` | Integer | Not Null | System epoch millisecond of event detection. |
| `clipSnapshot` | String | Nullable | String reference to video clip segment (path/URL) or base64 frame image. |

---

## 9. Entity: SESSION_RECAP (AI Generated Analytics)
Logs generative session recaps, MVP selections, and copy-ready social media caption text.

| Attribute Name | Data Type | Key / Constraint | Description / Business Rules |
| :--- | :--- | :--- | :--- |
| `recapId` | String | Primary Key (PK), Not Null | Unique identifier generated upon recap generation. |
| `sessionDate` | Integer | Not Null | Epoch timestamp representing the date/time of the session. |
| `club` | String | Foreign Key (FK), Not Null | References `CLUB.uuid`. |
| `totalMatches` | Integer | Not Null | Count of completed matches in the session. |
| `mvpUsername` | String | Foreign Key (FK), Not Null | References `PLAYER.username` for the selected MVP. |
| `runnersUp` | JSON | Not Null | JSON array of usernames designating runner-up players. |
| `recapMarkdown` | String | Not Null | Natural language newsletter markdown summary of the match session. |
| `socialMediaCaption` | String | Not Null | Copy-ready caption text (e.g. for Facebook/Viber) with emojis and hashtags. |
| `createdAt` | Integer | Not Null | Creation timestamp. |
