# Entity Relationship Diagram (ERD) - DinkMatch

> [!NOTE]
> **Diagram Notation Used:** **IE (Information Engineering) Crow's Foot Notation**
> - **Entities:** Rectangles represent database tables. Primary Keys (PK) and Foreign Keys (FK) are designated alongside field attributes.
> - **Cardinality Relationships:** Connecting lines specify database mapping dependencies:
>   - `||--|{` One-to-Many relationship.
>   - `||--o{` Zero-to-Many relationship.
>   - `||--o|` Zero-to-One relationship.

This document outlines the **Entity Relationship Diagram (ERD)** for the **DinkMatch** system, including the *Matchmaking Engine* subsystem, *External Camera* device integration, *Line & Foot Fault Detection*, and *Club/Profile management*.

---

## 1. Mermaid Entity Relationship Diagram

```mermaid
erDiagram
    CLUB ||--o{ PLAYER : registers
    CLUB ||--o{ COURT : manages
    CLUB ||--o{ QUEUE_ENTRY : houses
    CLUB ||--o{ ACTIVE_MATCH : schedules
    CLUB ||--o{ COMPLETED_MATCH : logs
    CLUB ||--o{ SESSION_RECAP : generates

    PLAYER ||--o{ QUEUE_ENTRY : joins
    PLAYER ||--o{ PEER_FEEDBACK : gives_or_receives
    PLAYER ||--o{ SESSION_RECAP : crowned_mvp

    ACTIVE_MATCH ||--o| COURT : occupies
    ACTIVE_MATCH ||--o{ FAULT_EVENT : records
    COMPLETED_MATCH ||--o{ PEER_FEEDBACK : references
    COMPLETED_MATCH ||--o{ FAULT_EVENT : history

    CLUB {
        string uuid PK
        string name
        json settings
        int completedMatchesResetAt
        int lastModified
        int settingsUpdatedAt
    }

    PLAYER {
        string username PK
        string userId FK "Auth identity"
        string firstName
        string lastName
        string duprId
        float rating
        string ratingStatus "No Rating | Calibrated"
        string currentRank "Beginner | Intermediate | Advanced | Expert | Pro"
        int rankUpdatedAt
        int matchesPlayed
        int wins
        int losses
        int createdAt
        int updatedAt
        int deletedAt
        int statsUpdatedAt
        int ratingUpdatedAt
        json history "playedWith / playedAgainst counts"
    }

    QUEUE_ENTRY {
        string queueId PK "Auto-generated"
        string username FK "References PLAYER"
        string queueType "GENERAL | WINNERS | LOSERS"
        int enteredAt "Sort priority"
        int queuedAt
        int createdAt
        int updatedAt
        int deletedAt "Tombstone for sync"
    }

    COURT {
        string courtId PK
        string name
        string status "idle | active"
        string activeMatchId FK "References ACTIVE_MATCH"
        int occupancyUpdatedAt
    }

    ACTIVE_MATCH {
        string matchId PK
        string queueSource "GENERAL | WINNERS | LOSERS | MANUAL"
        string teamA "JSON array of usernames"
        string teamB "JSON array of usernames"
        float expectedDifference
        string status "waiting | playing"
        int startedAt
        int createdAt
        int updatedAt
        int deletedAt "Tombstone for sync"
        string generatedBy
        string matchmakingMode
    }

    COMPLETED_MATCH {
        string matchId PK
        string matchType "singles | doubles"
        json teamA "Snapshot player array"
        json teamB "Snapshot player array"
        int teamAScore
        int teamBScore
        int startedAt
        int completedAt
        int updatedAt
        string club FK "References CLUB"
    }

    PEER_FEEDBACK {
        string feedbackId PK
        string matchId FK "References COMPLETED_MATCH"
        string giverUsername FK "References PLAYER"
        string receiverUsername FK "References PLAYER"
        string badgeType "Hustle | Synergy | Toxicity etc"
        int createdAt
    }

    FAULT_EVENT {
        string faultId PK
        string matchId FK "References ACTIVE_MATCH / COMPLETED_MATCH"
        string offendingPlayer FK "References PLAYER.username"
        string faultType "kitchen_volley_fault | serve_foot_fault | out_of_bounds"
        float confidence
        int timestamp
        string clipSnapshot "URL / base64 frame reference"
    }

    SESSION_RECAP {
        string recapId PK
        int sessionDate
        string club FK "References CLUB"
        int totalMatches
        string mvpUsername FK "References PLAYER"
        json runnersUp
        string recapMarkdown
        string socialMediaCaption
        int createdAt
    }
```

---

## 2. Cardinality & Relationships Description

1.  **CLUB to PLAYER (1:N)**: A club manages registrations for multiple players. A player belongs to one club session at a time in the kiosk context.
2.  **CLUB to COURT (1:N)**: A club facility manages $C$ physical courts. Each court belongs to one club.
3.  **CLUB to QUEUE_ENTRY (1:N)**: A club hosts multiple player queue entries in its open-play session.
4.  **PLAYER to QUEUE_ENTRY (1:1/1:N)**: A player can only have one active (non-deleted) queue entry at any time. History is stored via soft deletion (deletedAt timestamps).
5.  **ACTIVE_MATCH to COURT (1:1)**: An active match occupies exactly one court, and an active court runs exactly one active match.
6.  **ACTIVE_MATCH/COMPLETED_MATCH to FAULT_EVENT (1:N)**: An active match detects and logs multiple foot/line faults, which persist into the completed match record history.
7.  **COMPLETED_MATCH to PEER_FEEDBACK (1:N)**: A completed match can generate multiple peer feedback badge reviews (e.g. commendations for teammates/opponents).
8.  **PLAYER to PEER_FEEDBACK (1:N)**: A player can submit feedback (as a giver) and receive badges (as a receiver) from other players.
9.  **CLUB to SESSION_RECAP (1:N)**: A club generates multiple session recaps over time (one per open play night/session).
10. **PLAYER to SESSION_RECAP (1:N)**: A player can be selected as the session MVP across multiple session recaps.
