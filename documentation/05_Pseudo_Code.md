# Pseudocode - DinkMatch Algorithms

This document provides the detailed **Pseudocode** for the algorithms running within the **DinkMatch** kiosk system, including the *Matchmaking Engine* subsystem, *External Camera* device integration, *Line & Foot Fault Detection*, and *Club/Profile management*.

---

## 1. Algorithm 1: Point-Margin Elo (PM-Elo) Calibration

```typescript
function calculateShift(
    winners: Player[], 
    losers: Player[], 
    scoreW: number, 
    scoreL: number,
    kBase: number = 32,
    antiCarryThreshold: number = 200
): { updatedWinners: Player[], updatedLosers: Player[] } 
{
    // Step 1: Calculate team averages
    let ratingWinnersSum = 0;
    for winner in winners {
        ratingWinnersSum = ratingWinnersSum + winner.rating;
    }
    let ratingWinnersAvg = ratingWinnersSum / winners.length;

    let ratingLosersSum = 0;
    for loser in losers {
        ratingLosersSum = ratingLosersSum + loser.rating;
    }
    let ratingLosersAvg = ratingLosersSum / losers.length;

    // Step 2: Compute win expectancy using logistic distribution
    let expectedW = 1.0 / (1.0 + Math.pow(10.0, (ratingLosersAvg - ratingWinnersAvg) / 400.0));
    let expectedL = 1.0 - expectedW;

    // Step 3: Compute actual shares from score margin
    let actualW = scoreW / (scoreW + scoreL);
    let actualL = scoreL / (scoreW + scoreL);

    // Step 4: Calculate base rating adjustments
    let baseDeltaW = kBase * (actualW - expectedW);
    let baseDeltaL = kBase * (actualL - expectedL); // Note: baseDeltaL is negative because actualL < expectedL

    let updatedWinners: Player[] = [];
    let updatedLosers: Player[] = [];

    // Step 5: Adjust winner ratings (with anti-carry and calibration checks)
    for winner in winners {
        let playerDelta = baseDeltaW;

        // Accelerated calibration for unrated/new players
        if winner.ratingStatus == "No Rating" or winner.matchesPlayed < 5 {
            playerDelta = playerDelta * 1.5;
        }

        // Apply Teammate Anti-Carry Penalty
        if winners.length == 2 {
            let partner = (winner.username == winners[0].username) ? winners[1] : winners[0];
            let skillGap = winner.rating - partner.rating;
            if skillGap > antiCarryThreshold {
                // Discount the rating gain of the stronger player carrying the weaker one
                let discountFactor = 1.0 / (1.0 + (skillGap - antiCarryThreshold) / 200.0);
                playerDelta = playerDelta * discountFactor;
            }
        }

        let oldRating = winner.rating;
        let newRating = Math.max(100.0, winner.rating + playerDelta); // Clamped at 100 rating floor
        let clonedWinner = clone(winner);
        clonedWinner.rating = newRating;
        clonedWinner.matchesPlayed = winner.matchesPlayed + 1;
        clonedWinner.wins = winner.wins + 1;
        clonedWinner.ratingUpdatedAt = getCurrentTimestamp();
        clonedWinner.statsUpdatedAt = getCurrentTimestamp();

        // Step 5b: Check Rank Promotion / Tier transition
        let promoCheck = checkRankPromotion(clonedWinner, oldRating, newRating);
        if (promoCheck.promoted) {
            triggerRankPromotionAlert(clonedWinner.username, promoCheck.oldRank, promoCheck.newRank);
        }

        updatedWinners.push(clonedWinner);
    }

    // Step 6: Adjust loser ratings (with partner carry and calibration checks)
    for loser in losers {
        let playerDelta = baseDeltaL;

        if loser.ratingStatus == "No Rating" or loser.matchesPlayed < 5 {
            playerDelta = playerDelta * 1.5;
        }

        // Partner Capping: ensure rating loss is shared fairly
        if losers.length == 2 {
            let partner = (loser.username == losers[0].username) ? losers[1] : losers[0];
            let skillGap = partner.rating - loser.rating;
            if skillGap > antiCarryThreshold {
                // The weaker partner is protected from excessive rating drop
                let capFactor = 0.5;
                playerDelta = playerDelta * capFactor;
            }
        }

        let oldRating = loser.rating;
        let newRating = Math.max(100.0, loser.rating + playerDelta);
        let clonedLoser = clone(loser);
        clonedLoser.rating = newRating;
        clonedLoser.matchesPlayed = loser.matchesPlayed + 1;
        clonedLoser.losses = loser.losses + 1;
        clonedLoser.ratingUpdatedAt = getCurrentTimestamp();
        clonedLoser.statsUpdatedAt = getCurrentTimestamp();

        // Step 6b: Check Rank Promotion / Tier transition (demotion protected if applicable)
        let promoCheck = checkRankPromotion(clonedLoser, oldRating, newRating);
        if (promoCheck.promoted) {
            triggerRankPromotionAlert(clonedLoser.username, promoCheck.oldRank, promoCheck.newRank);
        }

        updatedLosers.push(clonedLoser);
    }

    // Partner Ratio Capping Enforcement (e.g. max 70/30 shift ratio between partners)
    enforceTeammateChangeRatio(updatedWinners);
    enforceTeammateChangeRatio(updatedLosers);

    return { updatedWinners, updatedLosers };
}

function getRatingCategory(rating: number): string {
    if (rating < 1400) return "Beginner";
    if (rating < 1700) return "Intermediate";
    if (rating < 1900) return "Advanced";
    if (rating < 2100) return "Expert";
    return "Pro";
}

function checkRankPromotion(
    player: Player, 
    oldRating: number, 
    newRating: number
): { promoted: boolean, oldRank: string, newRank: string } {
    let oldRank = getRatingCategory(oldRating);
    let newRank = getRatingCategory(newRating);
    if (newRank !== oldRank) {
        player.currentRank = newRank;
        player.rankUpdatedAt = getCurrentTimestamp();
        return { promoted: true, oldRank, newRank };
    }
    return { promoted: false, oldRank, newRank };
}

function triggerRankPromotionAlert(username: string, oldRank: string, newRank: string): void {
    // Dispatches a visual alert overlay and triggers speech announcer
    let alertMsg = `Congratulations ${username}! You have been promoted from ${oldRank} to ${newRank}!`;
    dispatchUIEvent("RANK_PROMOTION_ALERT", { username, oldRank, newRank, message: alertMsg });
    speakTTS(alertMsg);
}
```,StartLine:67,TargetContent:
```

---

## 2. Algorithm 2: Multi-Objective Matchmaking Optimization

```typescript
function draftBalancedMatch(
    queue: QueueEntry[],
    activeCourts: Court[],
    players: Record<string, Player>,
    settings: MatchmakingSettings
): ActiveMatch | null 
{
    // Step 1: Check system capacity
    if countIdleCourts(activeCourts) == 0 {
        return null; // Court congestion: wait
    }

    // Step 2: Fetch and sort candidates
    let checkedIn = queue.filter(q => q.deletedAt == null);
    checkedIn.sortByArrivalTimestampAscending();

    if checkedIn.length < 4 {
        return null; // Insufficient players to draft a match
    }

    // Step 3: Extract sliding window candidates (N = 8 or 12)
    let windowSize = Math.min(settings.slidingWindowSize, checkedIn.length);
    let candidates = checkedIn.slice(0, windowSize);

    // Step 4: Compute dynamic personal thresholds with logarithmic wait-time aging
    let personalThresholds = new Map<string, number>();
    let currentTimestamp = getCurrentTimestamp();
    for entry in candidates {
        let waitTimeSeconds = (currentTimestamp - entry.enteredAt) / 1000;
        let threshold = settings.baseThreshold + settings.gamma * Math.log(1.0 + waitTimeSeconds);
        personalThresholds.set(entry.username, threshold);
    }

    // Step 5: Evaluate combinations
    let allCombinations = generateCombinationsOfFour(candidates);
    let bestCost = Infinity;
    let bestFoursome: string[] = [];
    let bestTeamA: string[] = [];
    let bestTeamB: string[] = [];
    let bestExpectedDiff = 0.0;

    for foursome in allCombinations {
        let p1 = players[foursome[0]];
        let p2 = players[foursome[1]];
        let p3 = players[foursome[2]];
        let p4 = players[foursome[3]];

        // Generate the 3 possible team pairings for these 4 players
        let pairings = [
            { tA: [p1, p2], tB: [p3, p4] },
            { tA: [p1, p3], tB: [p2, p4] },
            { tA: [p1, p4], tB: [p2, p3] }
        ];

        for pairing in pairings {
            let ratingTA = (pairing.tA[0].rating + pairing.tA[1].rating) / 2;
            let ratingTB = (pairing.tB[0].rating + pairing.tB[1].rating) / 2;
            
            // Skill variance (Rating discrepancy)
            let meanRating = (p1.rating + p2.rating + p3.rating + p4.rating) / 4;
            let ratingVariance = (Math.pow(p1.rating - meanRating, 2) + 
                                  Math.pow(p2.rating - meanRating, 2) + 
                                  Math.pow(p3.rating - meanRating, 2) + 
                                  Math.pow(p4.rating - meanRating, 2)) / 4;

            // Total wait time sum
            let totalWaitTime = 0.0;
            for player in pairing.tA + pairing.tB {
                let entry = candidates.find(c => c.username == player.username);
                totalWaitTime = totalWaitTime + (currentTimestamp - entry.enteredAt);
            }

            // Calculate Joint Cost
            let cost = (settings.alpha * ratingVariance) - (settings.beta * totalWaitTime);

            // Peak Parity Mode validation checks
            if settings.mode == "Peak Parity" {
                let maxDiff = getMaxRatingDifference(pairing.tA + pairing.tB);
                let thresholdSum = 0;
                for player in pairing.tA + pairing.tB {
                    thresholdSum = thresholdSum + personalThresholds.get(player.username);
                }
                let avgThreshold = thresholdSum / 4;
                if maxDiff > avgThreshold {
                    continue; // Skip pairing: exceeds acceptable skill gap bounds
                }
            }

            if cost < bestCost {
                bestCost = cost;
                bestFoursome = [p1.username, p2.username, p3.username, p4.username];
                bestTeamA = [pairing.tA[0].username, pairing.tA[1].username];
                bestTeamB = [pairing.tB[0].username, pairing.tB[1].username];
                bestExpectedDiff = Math.abs(ratingTA - ratingTB);
            }
        }
    }

    if bestFoursome.length == 4 {
        // Return constructed match details
        return {
            matchId: generateRandomString(7),
            teamA: bestTeamA,
            teamB: bestTeamB,
            expectedDifference: bestExpectedDiff,
            status: "waiting",
            createdAt: currentTimestamp,
            updatedAt: currentTimestamp
        };
    }
    return null;
}
```

---

## 3. Algorithm 3: Local-First Conflict Reconciliation (`mergeAppState`)

```typescript
function mergeAppState(local: AppState, server: AppState): AppState 
{
    // Step 1: Detect hard resets
    if isStateEmpty(local) and local.lastModified > server.lastModified {
        return clone(local);
    }
    if isStateEmpty(server) and server.lastModified > local.lastModified {
        return clone(server);
    }

    // Step 2: Compute effective reset checkpoints
    let playersResetAt = Math.max(local.playersResetAt ?? 0, server.playersResetAt ?? 0);
    let queuesResetAt = Math.max(local.queuesResetAt ?? 0, server.queuesResetAt ?? 0);
    let matchesResetAt = Math.max(local.matchesResetAt ?? 0, server.matchesResetAt ?? 0);

    // Step 3: Filter collections using checkpoint timestamps
    let localPlayers = filterPlayersByCheckpoint(local.players, playersResetAt);
    let serverPlayers = filterPlayersByCheckpoint(server.players, playersResetAt);
    let localQueues = filterQueuesByCheckpoint(local.queues, queuesResetAt);
    let serverQueues = filterQueuesByCheckpoint(server.queues, queuesResetAt);
    let localMatches = filterMatchesByCheckpoint(local.activeMatches, matchesResetAt);
    let serverMatches = filterMatchesByCheckpoint(server.activeMatches, matchesResetAt);

    // Step 4: Merge Players using per-field Last-Write-Wins (LWW)
    let mergedPlayers: Record<string, Player> = clone(serverPlayers);

    for (username, lp) in localPlayers {
        let sp = mergedPlayers[username];
        if sp == null {
            mergedPlayers[username] = lp;
            continue;
        }

        // Whole-object base selection using overall updatedAt timestamp
        let baseWinner = (lp.updatedAt > sp.updatedAt) ? clone(lp) : clone(sp);

        // Stats per-field overlay (use statsUpdatedAt to preserve active match results)
        let other = (lp.updatedAt > sp.updatedAt) ? sp : lp;
        if other.statsUpdatedAt > baseWinner.statsUpdatedAt {
            baseWinner.matchesPlayed = other.matchesPlayed;
            baseWinner.wins = other.wins;
            baseWinner.losses = other.losses;
            baseWinner.history = other.history;
            baseWinner.statsUpdatedAt = other.statsUpdatedAt;
        }

        // Rating per-field overlay (use ratingUpdatedAt to preserve match rating engine updates)
        if other.ratingUpdatedAt > baseWinner.ratingUpdatedAt {
            baseWinner.rating = other.rating;
            baseWinner.ratingUpdatedAt = other.ratingUpdatedAt;
        }

        mergedPlayers[username] = baseWinner;
    }

    // Step 5: Resolve player deletions using tombstones
    for username in mergedPlayers.keys() {
        let lp = localPlayers[username];
        let sp = serverPlayers[username];
        let localDeleted = lp.deletedAt ?? 0;
        let serverDeleted = sp.deletedAt ?? 0;

        if localDeleted > 0 or serverDeleted > 0 {
            let liveAddedAt = Math.max(
                (lp != null and lp.deletedAt == null) ? (lp.addedAt ?? lp.createdAt) : 0,
                (sp != null and sp.deletedAt == null) ? (sp.addedAt ?? sp.createdAt) : 0
            );
            let tombDeletedAt = Math.max(localDeleted, serverDeleted);

            // Tombstone wins only if deletedAt is newer than the latest re-addition timestamp
            if tombDeletedAt > liveAddedAt {
                mergedPlayers[username].deletedAt = tombDeletedAt;
            }
        }
    }

    // Step 6: Merge Queue entries per-player using queuedAt vs deletedAt
    let mergedQueues: QueueEntry[] = [];
    let allQueueEntries = concat(localQueues, serverQueues);
    let entriesByPlayer = groupByPlayer(allQueueEntries);

    for (username, entries) in entriesByPlayer {
        let live = entries.filter(e => e.deletedAt == null);
        let tombstones = entries.filter(e => e.deletedAt != null);

        let maxLiveQueuedAt = getMaxTimestamp(live, "queuedAt");
        let maxTombDeletedAt = getMaxTimestamp(tombstones, "deletedAt");

        let candidates = entries;
        if maxTombDeletedAt > maxLiveQueuedAt {
            candidates = tombstones; // Deletion was performed last: tombstone wins
        } else if maxLiveQueuedAt > maxTombDeletedAt {
            candidates = live;      // Re-queue was performed last: live entry wins
        }

        // Resolve candidate conflict using highest queuedAt (or updatedAt)
        let winner = pickWinnerByTimestamps(candidates);
        mergedQueues.push(winner);
    }

    // Step 7: Merge Matches using LWW and explicit deletion tombstones
    let mergedMatches: ActiveMatch[] = [];
    let matchMap = new Map<string, ActiveMatch>();

    for m in localMatches { matchMap.set(m.matchId, m); }
    for m in serverMatches {
        let existing = matchMap.get(m.matchId);
        if existing == null or m.updatedAt > existing.updatedAt {
            matchMap.set(m.matchId, m);
        }
    }

    // Enforce match deletion tombstones
    for matchId in matchMap.keys() {
        let lm = localMatches.find(m => m.matchId == matchId);
        let sm = serverMatches.find(m => m.matchId == matchId);
        let localDeleted = lm.deletedAt ?? 0;
        let serverDeleted = sm.deletedAt ?? 0;
        
        if localDeleted > 0 or serverDeleted > 0 {
            let maxDeleted = Math.max(localDeleted, serverDeleted);
            matchMap.get(matchId).deletedAt = maxDeleted;
        }
    }

    for (matchId, m) in matchMap {
        mergedMatches.push(m);
    }

    // Step 8: Enforce player queue XOR match constraint
    let playersInActiveMatches = new Set<string>();
    for m in mergedMatches {
        if m.deletedAt == null {
            addAll(playersInActiveMatches, m.teamA);
            addAll(playersInActiveMatches, m.teamB);
        }
    }

    // Remove live queue entries if the player is currently playing on an active court
    let finalQueues = mergedQueues.filter(
        q => q.deletedAt != null or not playersInActiveMatches.has(q.username)
    );

    // Step 9: Assemble final merged state
    let mergedState = clone(server);
    mergedState.players = mergedPlayers;
    mergedState.queues = finalQueues;
    mergedState.activeMatches = mergedMatches;
    mergedState.lastModified = getCurrentTimestamp();

    return mergedState;
}
```

---

## 4. Algorithm 4: Real-time Computer Vision Fault Detection

```typescript
function detectFaults(
    frame: ImageFrame,
    courtLines: CourtLinesCalibration,
    activePlayers: PlayerCoordinates[],
    ballEvents: BallEvent[]
): FaultDetectionEvent | null 
{
    // Step 1: Calibrate homography mapping of the court surface
    let transformedFrame = applyHomographyMatrix(frame, courtLines.homographyMatrix);
    
    // Step 2: Track foot positions of players relative to court lines
    for player in activePlayers {
        let leftFootPolygon = getFootBoundingPolygon(transformedFrame, player.id, "left");
        let rightFootPolygon = getFootBoundingPolygon(transformedFrame, player.id, "right");

        // Rule A: Kitchen (Non-Volley Zone) Volley Fault Check
        // If a volley event occurs, verify that neither foot overlaps the kitchen boundary
        let kitchenLine = courtLines.kitchenLinePolygon;
        for ballEvent in ballEvents {
            if ballEvent.type == "volley" and ballEvent.playerId == player.id {
                let leftFootTouches = leftFootPolygon.intersects(kitchenLine);
                let rightFootTouches = rightFootPolygon.intersects(kitchenLine);
                
                if leftFootTouches or rightFootTouches {
                    return {
                        eventId: generateRandomString(9),
                        matchId: player.activeMatchId,
                        timestamp: getCurrentTimestamp(),
                        faultType: "kitchen_volley_fault",
                        offendingPlayer: player.username,
                        confidence: calculateCvConfidence(leftFootPolygon, rightFootPolygon),
                        clipSnapshot: captureFrameSnapshot(frame)
                    };
                }
            }
        }

        // Rule B: Serve Foot Fault Check
        // If a serve event occurs, the server's feet must not cross the baseline or serve centerline
        for ballEvent in ballEvents {
            if ballEvent.type == "serve" and ballEvent.playerId == player.id {
                let baseline = courtLines.baselinePolygon;
                let centerLine = courtLines.serveCenterLine;
                
                let crossBaseline = leftFootPolygon.intersects(baseline) or rightFootPolygon.intersects(baseline);
                let crossCenterLine = leftFootPolygon.intersects(centerLine) or rightFootPolygon.intersects(centerLine);

                if crossBaseline or crossCenterLine {
                    return {
                        eventId: generateRandomString(9),
                        matchId: player.activeMatchId,
                        timestamp: getCurrentTimestamp(),
                        faultType: "serve_foot_fault",
                        offendingPlayer: player.username,
                        confidence: calculateCvConfidence(leftFootPolygon, rightFootPolygon),
                        clipSnapshot: captureFrameSnapshot(frame)
                    };
                }
            }
        }
    }

    // Rule C: Out-of-bounds line call detection
    for ballEvent in ballEvents {
        if ballEvent.type == "bounce" {
            let courtBoundaries = courtLines.outerBoundariesPolygon;
            let ballImpactPoint = ballEvent.coordinates;
            
            if not courtBoundaries.contains(ballImpactPoint) {
                return {
                    eventId: generateRandomString(9),
                    matchId: ballEvent.activeMatchId,
                    timestamp: getCurrentTimestamp(),
                    faultType: "out_of_bounds_call",
                    offendingPlayer: "none", // Ball fault
                    confidence: calculateBallCvConfidence(ballImpactPoint),
                    clipSnapshot: captureFrameSnapshot(frame)
                };
            }
        }
    }

    return null;
}
```

---

## 5. Algorithm 5: Generative Session Recap & MVP Analytics

```typescript
interface SessionRecap {
    recapId: string;
    sessionDate: number;
    mvp: string;
    runnersUp: string[];
    recapMarkdown: string;
    socialMediaCaption: string;
}

function generateSessionRecap(
    sessionMatches: MatchRecord[],
    players: Record<string, Player>,
    clubName: string
): SessionRecap | null
{
    if (sessionMatches.length == 0) {
        return null;
    }

    // Step 1: Calculate Session Statistics for all participating players
    let stats = new Map<string, { wins: number, games: number, pointsDiff: number, ratingDelta: number }>();

    for (let match of sessionMatches) {
        let playersInMatch = match.winners.concat(match.losers);
        for (let player of playersInMatch) {
            let entry = stats.get(player.username) || { wins: 0, games: 0, pointsDiff: 0, ratingDelta: 0 };
            entry.games += 1;
            if (match.winners.some(w => w.username == player.username)) {
                entry.wins += 1;
                entry.pointsDiff += (match.scoreWinner - match.scoreLoser);
            } else {
                entry.pointsDiff -= (match.scoreWinner - match.scoreLoser);
            }
            // Find rating delta from match log
            let ratingChange = match.ratingDeltas[player.username] || 0.0;
            entry.ratingDelta += ratingChange;
            stats.set(player.username, entry);
        }
    }

    // Step 2: Determine Leaderboard Rankings & MVP designations
    let sortedStats = Array.from(stats.entries()).sort((a, b) => {
        // Rank by win ratio first, then rating delta, then points margin
        let winRatioA = a[1].wins / a[1].games;
        let winRatioB = b[1].wins / b[1].games;
        if (winRatioA !== winRatioB) return winRatioB - winRatioA;
        return b[1].ratingDelta - a[1].ratingDelta;
    });

    let mvpUsername = sortedStats[0][0];
    let runnersUp: string[] = [];
    for (let i = 1; i < Math.min(4, sortedStats.length); i++) {
        runnersUp.push(sortedStats[i][0]);
    }

    // Step 3: Construct prompt payload for the LLM
    let promptPayload = {
        clubName: clubName,
        date: getCurrentDateString(),
        totalMatches: sessionMatches.length,
        mvp: players[mvpUsername].displayName || mvpUsername,
        mvpStats: sortedStats[0][1],
        runnersUpDetails: runnersUp.map(username => ({
            name: players[username].displayName || username,
            stats: stats.get(username)
        })),
        matchesSummary: sessionMatches.map(m => ({
            winnerNames: m.winners.map(w => w.displayName).join(" & "),
            loserNames: m.losers.map(l => l.displayName).join(" & "),
            score: `${m.scoreWinner} - ${m.scoreLoser}`
        }))
    };

    // Step 4: Call external LLM API or local inference engine
    let response = callGenerativeAI({
        model: "gemini-1.5-flash",
        prompt: buildRecapPrompt(promptPayload),
        temperature: 0.7
    });

    if (response == null || response.error != null) {
        // Fallback: Generate template-based static output
        let fallbackText = `# ${clubName} Session Recap\n\n` +
            `Congratulations to our MVP **${promptPayload.mvp}** with a record of ` +
            `${promptPayload.mvpStats.wins}/${promptPayload.mvpStats.games} games and a rating change of ` +
            `+${promptPayload.mvpStats.ratingDelta.toFixed(1)}!\n\n` +
            `Total matches played today: ${promptPayload.totalMatches}.`;
        
        let fallbackCaption = `Congrats to today's MVP ${promptPayload.mvp}! 🏓 Great games all around at ${clubName}! #DinkMatch`;
        
        return {
            recapId: generateRandomString(9),
            sessionDate: getCurrentTimestamp(),
            mvp: mvpUsername,
            runnersUp: runnersUp,
            recapMarkdown: fallbackText,
            socialMediaCaption: fallbackCaption
        };
    }

    return {
        recapId: generateRandomString(9),
        sessionDate: getCurrentTimestamp(),
        mvp: mvpUsername,
        runnersUp: runnersUp,
        recapMarkdown: response.recapMarkdownText,
        socialMediaCaption: response.socialCaptionText
    };
}
```
