# Structured English - DinkMatch

This document outlines the core logic of the **DinkMatch** scheduling and rating system using **Structured English**, including the *Matchmaking Engine* subsystem, *External Camera* device integration, *Line & Foot Fault Detection*, and *Club/Profile management*.

---

## 1. Process 1.0: Player Check-In, Registration & Club Membership

```
READ player input action (Register, Login, Edit_Profile, Join_Club, Check_In)

CASE action OF

    Register:
        READ input credentials (Username, PIN/Password, Display_Name, Base_Rating)
        IF Username already exists in local Profiles Database THEN
            DISPLAY error "Username is already taken"
            TERMINATE registration
        ENDIF
        CREATE new Profile:
            Set Username, Password_Hash, Display_Name
            Set Rating to default seed rating based on Base_Rating
            Set ratingStatus to "No Rating" (calibration state)
            Set matchesPlayed, wins, losses to 0
        SAVE profile to local database
        DISPLAY confirmation "Registration successful!"

    Login:
        READ input credentials (Username, PIN/Password)
        IF credentials match local Player Profiles Database THEN
            AUTHENTICATE player and initialize session
        ELSE
            DISPLAY error "Invalid credentials"
        ENDIF

    Edit_Profile:
        READ profile modifications (Display_Name, Avatar, DUPR_ID)
        UPDATE player profile fields in local database
        SET profile.updatedAt to current timestamp
        FLAG profile record as "unsynced" for Likha ERP sync middleware

    Join_Club:
        READ input Club_UUID
        VERIFY Club_UUID with local cache or Likha ERP client
        IF club exists THEN
            LINK player profile to Club_UUID as member
            FLAG membership relation as "unsynced"
            DISPLAY confirmation "Successfully joined club!"
        ELSE
            DISPLAY error "Club not found"
        ENDIF

    Check_In:
        IF player session is not authenticated THEN
            DISPLAY error "Authentication required"
            TERMINATE check-in
        ENDIF
        IF player is already active in Queue OR player is currently on an Active Court THEN
            DISPLAY error "Player is already active in session"
            TERMINATE check-in
        ENDIF
        CREATE Queue Entry:
            Set Player_ID to current player ID
            Set Arrival_Timestamp to current system time
            Set Queue_Type to Sport_Category (e.g. "Pickleball")
        INSERT Queue Entry into local Active Queue Database
        DISPLAY confirmation "Check-in successful! You are queued."

ENDCASE
```

---

## 2. Process 2.0: Matchmaking Loop & Optimization Scheduler

```
LOOP periodically (every 10 seconds)
    
    COUNT number of vacant courts in Court Status Database
    IF number of vacant courts is equal to 0 THEN
        EXIT loop iteration (wait for a court to become vacant)
    ENDIF

    RETRIEVE all checked-in players from Active Queue Database
    FILTER players matching the current queue category (e.g. "Pickleball")
    SORT filtered players by Arrival_Timestamp in ascending order (longest waiting first)

    IF total number of waiting players is less than 4 THEN
        EXIT loop iteration (insufficient players to form a doubles match)
    ENDIF

    DETERMINE sliding search window size N (default to 8, or 12 if queue is dense)
    ISOLATE the first N players from the sorted list (this is the candidate pool)

    FOR each player i in the candidate pool:
        READ player i's Wait_Time (Current_Time minus Arrival_Timestamp)
        CALCULATE personal matchmaking threshold:
            Threshold_i = Base_Threshold + (Gamma * Natural_Logarithm(1 + Wait_Time))
    ENDFOR

    GENERATE all unique 4-player combinations from the candidate pool (size C(N, 4))
    INITIALIZE Minimum_Cost to infinity
    INITIALIZE Best_Match to null

    FOR each 4-player combination M:
        DIVIDE the 4 players into Team A (2 players) and Team B (2 players)
        
        CALCULATE rating variance among all 4 players (measures skill parity)
        CALCULATE total wait time (sum of wait times for all 4 players)
        
        CALCULATE Joint_Cost:
            Joint_Cost = (Alpha * Rating_Variance) - (Beta * Total_Wait_Time)

        IF current Operating Mode is "Peak Parity" THEN
            DETERMINE the max rating difference between any two players in M
            IF max rating difference exceeds the average of their Thresholds THEN
                DISCARD combination M (violates skill parity limit)
                ITERATE to next combination
            ENDIF
        ENDIF

        IF Joint_Cost is less than Minimum_Cost THEN
            Set Minimum_Cost to Joint_Cost
            Set Best_Match to combination M
        ENDIF
    ENDFOR

    IF Best_Match is not null THEN
        REMOVE the 4 selected players from Active Queue Database
        ASSIGN Best_Match to the vacant court in Court Status Database
        SET court state to "Active" and START match timer
        TRIGGER Web Speech Audio Announcer to call player names for the assigned court
    ENDIF

ENDLOOP
```

---

## 3. Process 4.0: PM-Elo Calibration & Rank Promotion

```
READ match completed event (Match_ID, Winning_Team, Losing_Team, Score_Winners, Score_Losers)

RETRIEVE player profile records from Profiles Cache Database (D2) for winners and losers
CALCULATE average rating for Winners and Losers

CALCULATE expected win probability using logistic distribution:
    Expected_Winners = 1.0 / (1.0 + 10^((Average_Losers - Average_Winners) / 400.0))
    Expected_Losers = 1.0 - Expected_Winners

CALCULATE actual shares based on score margins:
    Actual_Winners = Score_Winners / (Score_Winners + Score_Losers)
    Actual_Losers = Score_Losers / (Score_Winners + Score_Losers)

CALCULATE base rating delta:
    Base_Delta_Winners = K_Base * (Actual_Winners - Expected_Winners)
    Base_Delta_Losers = K_Base * (Actual_Losers - Expected_Losers)

FOR each Winner:
    SET Player_Delta = Base_Delta_Winners
    IF Winner ratingStatus is "No Rating" OR matchesPlayed < 5 THEN
        Player_Delta = Player_Delta * 1.5 (calibration acceleration)
    ENDIF
    IF skill gap between Winner and Teammate > anti-carry threshold THEN
        APPLY anti-carry discount penalty to strong player's gain
    ENDIF
    SET Old_Rating = Winner.rating
    SET New_Rating = Clamped(Winner.rating + Player_Delta, Floor: 100)
    
    // Validate Rank Promotion
    IF Category(New_Rating) is NOT EQUAL TO Category(Old_Rating) THEN
        SET Winner.currentRank = Category(New_Rating)
        SET Winner.rankUpdatedAt = current timestamp
        IF Category(New_Rating) is higher than Category(Old_Rating) THEN
            TRIGGER Rank Promotion Alert (visual card and TTS voice announcement)
        ENDIF
    ENDIF
    
    UPDATE Winner's rating, matchesPlayed, wins, and timestamps
    SAVE updated Profile to Profiles Cache Database (D2)
ENDFOR

FOR each Loser:
    SET Player_Delta = Base_Delta_Losers
    IF Loser ratingStatus is "No Rating" OR matchesPlayed < 5 THEN
        Player_Delta = Player_Delta * 1.5
    ENDIF
    IF skill gap between Teammate and Loser > anti-carry threshold THEN
        APPLY partner capping protection to weak player's loss
    ENDIF
    SET Old_Rating = Loser.rating
    SET New_Rating = Clamped(Loser.rating + Player_Delta, Floor: 100)
    
    // Validate Rank Promotion / Demotion
    IF Category(New_Rating) is NOT EQUAL TO Category(Old_Rating) THEN
        SET Loser.currentRank = Category(New_Rating)
        SET Loser.rankUpdatedAt = current timestamp
    ENDIF
    
    UPDATE Loser's rating, matchesPlayed, losses, and timestamps
    SAVE updated Profile to Profiles Cache Database (D2)
ENDFOR

ENFORCE team change ratio caps (max 70/30 distribution between partners)
FLAG updated profile records as "unsynced" for Likha ERP sync middleware
```

---

## 4. Process 7.0: Video Capture & Fault Detection

```
READ active camera feed configuration (External_Camera_Device_ID)
INITIALIZE stream connection to External Camera

IF connection fails THEN
    LOG error "External Camera connection failed"
    TERMINATE process
ENDIF

LOOP continuously (every video frame captured)

    RETRIEVE raw image frame from External Camera stream
    SAVE frame image buffer to volatile Video Frames Cache Database

    RETRIEVE court configuration coordinates from Court Status Database
    (Includes boundary line segments for baseline and non-volley kitchen lines)

    FOR each active court monitored by the camera view:
        IF court status is "Active" THEN
            
            RUN computer vision edge detection to verify lines location
            TRACK coordinates of player foot bounding polygons
            
            IF a player hits a volley (ball collision detected near kitchen net) THEN
                IF player's foot touches or crosses the non-volley kitchen line THEN
                    TRIGGER fault warning flag
                    Set Fault_Type = "Kitchen Volley Foot Fault"
                ENDIF
            ENDIF

            IF a player executes a serve (ball serve action detected) THEN
                IF player's foot touches or crosses the baseline or serve boundary line THEN
                    TRIGGER fault warning flag
                    Set Fault_Type = "Serve Foot Fault"
                ENDIF
            ENDIF

            IF ball bounce coordinates exceed outer court boundary lines THEN
                TRIGGER fault warning flag
                Set Fault_Type = "Out-of-Bounds Call"
            ENDIF

            IF fault warning flag is triggered THEN
                CREATE Fault Event Log:
                    Set Match_ID, Timestamp, Player_ID, Fault_Type, and Confidence
                WRITE Fault Event Log to local Match History logs
                
                DISPATCH visual warning overlay (flash red bounding box on screen)
                TRIGGER browser Audio Announcer to voice broadcast the fault type
            ENDIF

        ENDIF
    ENDFOR

ENDLOOP
```

---

## 5. Process 8.0: Generate AI Session Recap & MVP Analytics

```
READ coordinator request to end session and generate recap
RETRIEVE all matches played during the active session from Match History Logs Database (D4)

IF no matches are found for the session THEN
    DISPLAY warning "No matches played in this session."
    TERMINATE recap generation
ENDIF

READ player profile records from Profiles Cache Database (D2) to cross-reference ratings and names
INITIALIZE statistics arrays (Player_Win_Ratios, Player_Points_Scored, Player_Rating_Deltas)

FOR each match M in the session:
    FOR each player P in match M:
        RECORD player P's score margin, win/loss status, and Elo rating change
        ACCUMULATE stats into respective arrays
    ENDFOR
ENDFOR

SORT players by total wins and rating deltas to identify session leaders
IDENTIFY the top player as the Session MVP
IDENTIFY runner-up high-performers for specialized roles (e.g., "Most Improved", "Highest Point Scorer")

CONSTRUCT prompt context:
    Include club details, court numbers, list of matches, player rating changes, and designated MVPs

CALL LLM generative agent (local model or cloud endpoint):
    Pass prompt context
    Request a professional natural language newsletter summary
    Request a copy-ready, highly engaging social media caption (with emojis and hashtags)

IF LLM call is successful THEN
    SAVE generated Recap Markdown and MVP stats table to Match History Logs Database (D4)
    DISPLAY AI Session Recap on the Player Dashboard
    PROVIDE Copy-Ready Social Media Caption panel to the Coordinator dashboard
    DISPLAY confirmation "AI Recap and Social Caption successfully generated!"
ELSE
    DISPLAY error "AI generation failed. Using fallback deterministic template."
    GENERATE template-based summary of scores and MVP list
    SAVE fallback recap to D4
ENDIF
```
