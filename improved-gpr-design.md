# Improved Global Power Rankings — Design Document

**Author:** Community Model  
**Reference:** Riot Games / AWS Global Power Rankings (Worlds 2024)  
**Data Source:** Oracle's Elixir downloadable match CSV files  

---

## 1. Motivation and Critique of the Existing Model

The Riot/AWS Global Power Rankings (GPR) system introduced at Worlds 2024 is a meaningful step forward for LoL esports analytics. It blends individual team Elo with a league-wide Elo to contextualize results across regions that rarely play each other. Its core formula is:

```
Power Score = 0.8 × Team Elo + 0.2 × League Elo
```

However, two structural weaknesses undermine the model's goal of reflecting *current* team strength:

### 1.1 Flat International K-Factor

All international play is awarded a single elevated K-factor tier regardless of tournament prestige:

| Stage | Regional K | International K |
|---|---|---|
| Play-Ins | 8 | 12 |
| Main Stage | 16 | 20 |
| Playoffs | 20 | 36 |

A Worlds playoff game and a First Stand playoff game both receive K = 36. This is incorrect. Worlds is the apex of the competitive calendar, drawing every major region's best team in a single-elimination bracket. First Stand, while a legitimate international event, is smaller in field size, prestige, and competitive stakes. Treating them identically inflates the Elo signal from lesser international events.

### 1.2 No Temporal Decay Within the Evaluation Window

The model uses hard evaluation windows — 2 years for Team Elo, 3 years for League Elo — but applies no decay within those windows. A series played 23 months ago contributes the same Elo delta as one played last week. In a game defined by frequent roster changes, meta shifts, and split-by-split form variance, this produces rankings that can lag significantly behind current reality.

**The canonical symptom:** A team like HLE, who placed 10th in LCK Cup 2026, can rank above BLG, who won LPL Winter 2026, because HLE's First Stand 2025 victory is still contributing full, undecayed Elo. The historical international result is crowding out the most recent evidence of team quality.

---

## 2. Design Goals

The improved model retains all the features of the original that are well-founded:

- Blended Team Elo + League Elo to contextualize intra-regional results
- K-factor scaling to weight high-stakes matches more heavily
- League Elo updated by international performance to capture regional strength dynamics

It adds two targeted improvements:

1. **Tiered international K-factors** scaled by tournament prestige
2. **Exponential temporal decay** applied to every Elo delta as a function of how old the result is

The acknowledged tradeoff: the model becomes more reactive. Teams on hot streaks rise faster; teams in slumps fall faster. This is intentional — the goal is *current* strength, not a sustained historical ledger.

---

## 3. Core Formula (Unchanged)

```
Power Score = α × Team_Elo + β × League_Elo
```

Default weights (matching Riot's validated split):

- **α = 0.80** (Team Elo weight)
- **β = 0.20** (League Elo weight)

Both weights remain configurable.

---

## 4. Elo Calculation (Unchanged Structure)

### 4.1 Expected Result

```
We = 1 / (1 + 10^(−dr/400))
```

Where `dr` is the Elo rating difference between the two teams (or leagues, for League Elo).

### 4.2 Elo Update

```
P_after = P_before + K_effective × (W − We)
```

Where:
- `W` = match result (1 = win, 0 = loss)
- `K_effective` = the final adjusted K-factor after applying tournament tier and temporal decay (see Sections 5 and 6)

---

## 5. Improvement 1 — Tiered International K-Factors

### 5.1 Rationale

Not all international tournaments are equal. The prestige hierarchy from highest to lowest is clearly:

1. **Worlds** — All major regions represented; full bracket; the undisputed championship
2. **MSI** — Mid-season invitational; major-region only; meaningful but shorter
3. **First Stand** (and equivalent future inter-regional invitationals) — Legitimate international play, but smaller field and lower stakes

The K-factor for international play should reflect this hierarchy rather than collapsing it into a single multiplier.

### 5.2 Tournament Tier Definitions

| Tier | Tournaments | Description |
|---|---|---|
| S | Worlds | Apex annual championship |
| A | MSI | Mid-season major invitational |
| B | First Stand, other multi-region invitationals | Smaller international events |
| Regional | All domestic league play | LCK, LPL, LEC, LCS, etc. |

### 5.3 K-Factor Table

| Stage | Regional | Tier B (e.g. First Stand) | Tier A (MSI) | Tier S (Worlds) |
|---|---|---|---|---|
| Play-Ins / Group Stage | 8 | 12 | 16 | 20 |
| Main Stage / Bracket | 16 | 20 | 28 | 36 |
| Playoffs / Finals | 20 | 24 | 32 | 48 |

**Key changes from Riot's model:**
- Worlds playoffs increases from 36 → 48 (strengthening the apex signal)
- MSI playoffs decreases from 36 → 32 (appropriate de-emphasis relative to Worlds)
- First Stand playoffs decreases from 36 → 24 (meaningful, but not equivalent to MSI)
- Play-in stages are correspondingly tiered

These values are starting parameters and should be tuned as data accumulates.

### 5.4 Assigning Tournament Tier from Oracle's Elixir Data

Oracle's Elixir identifies tournaments via the `league` field. The mapping is:

```python
TOURNAMENT_TIERS = {
    # Tier S
    "WLDs": "S",
    "Worlds": "S",
    # Tier A
    "MSI": "A",
    # Tier B
    "First Stand": "B",
    # All other values → Regional
}
```

The `playoffs` column (0 or 1) distinguishes playoff games from group/regular stage games. The `league` field in combination with `playoffs` and contextual parsing of `split` provides everything needed to assign a tier and stage.

---

## 6. Improvement 2 — Temporal Decay

### 6.1 Rationale

A result from 22 months ago should carry less weight than one from last week. This is especially true when the same roster rarely persists across multiple years. Temporal decay applies a continuous discount to historical results proportional to their age, rather than a binary "inside/outside the window" cutoff.

### 6.2 Decay Function

```
decay(t) = e^(−λ × t)
```

Where:
- `t` = time elapsed since the match, measured in **months**
- `λ` = decay rate constant
- `e` = Euler's number (~2.718)

The effective K-factor for any match becomes:

```
K_effective = K_base × decay(t)
              = K_base × e^(−λ × t)
```

### 6.3 Choosing λ — The Half-Life Parameter

The decay rate is most intuitively expressed as a **half-life**: the age at which a result contributes half as much Elo as it would if it were played today.

```
λ = ln(2) / half_life_months
```

| Half-Life | λ Value | Character |
|---|---|---|
| 6 months | 0.1155 | Very aggressive — one split matters most |
| 12 months | 0.0578 | Moderate — approximately one competitive year |
| 18 months | 0.0385 | Conservative — closer to Riot's current behavior |

**Default recommendation: 12-month half-life (λ = 0.0578)**

This means:
- A result from today contributes 100% of its Elo delta
- A result from 12 months ago contributes ~50%
- A result from 24 months ago contributes ~25%

Under this setting, First Stand 2025 (approximately 12–15 months old at time of a March 2026 ranking) would contribute ~40–47% of its original Elo delta — still meaningful, but no longer capable of overriding a team's entire current-season performance.

### 6.4 Interaction with Tournament Tiers

Decay and tier multipliers stack multiplicatively. A Worlds 2024 playoff result (K = 48 base) evaluated in March 2026 (~17 months later) with λ = 0.0578:

```
K_effective = 48 × e^(−0.0578 × 17) ≈ 48 × 0.373 ≈ 17.9
```

A First Stand 2025 playoff result (K = 24 base) evaluated in March 2026 (~12 months later):

```
K_effective = 24 × e^(−0.0578 × 12) ≈ 24 × 0.500 ≈ 12.0
```

Compare: A LCK Winter 2026 playoff result (K = 20 base) played in January 2026 (~2 months ago):

```
K_effective = 20 × e^(−0.0578 × 2) ≈ 20 × 0.891 ≈ 17.8
```

This illustrates the correct behavior: a very recent domestic playoff result now carries roughly the same weight as a tiered-and-decayed international result from 17 months ago, rather than being vastly overshadowed by it.

---

## 7. Evaluation Windows

Following the original model's approach, with slight adjustment to align with the decay function:

| Component | Window |
|---|---|
| Team Elo | 2 years (24 months) |
| League Elo | 3 years (36 months) |

Because decay reduces the influence of old results continuously, the hard window primarily serves as a computational cutoff rather than a meaningful weighting boundary. Results near the edge of the window already contribute very little due to decay.

---

## 8. League Elo Updates from International Play

League Elo continues to be updated when teams play internationally, using the same tiered K-factors applied to the corresponding league's rating rather than the individual team's. This preserves the original model's insight that inter-regional matchups reveal regional strength.

The league-level decay is applied using the same λ parameter, consistent with Team Elo treatment.

---

## 9. Initial Elo Values

| Parameter | Value |
|---|---|
| Starting Team Elo | 1500 |
| Starting League Elo | 1500 |
| Minimum matches before ranking | 5 |

Teams with fewer than 5 recorded matches in the evaluation window are marked as unranked to avoid noise from extremely small samples.

---

## 10. Regional K-Factor Scaling

Following Riot's approach, regional K-factors are scaled to align with each league's historical international performance. This prevents a dominant region from being under-represented simply because their intra-regional Elo ceiling is lower due to internal competition depth.

The implementation scales regional K by a `league_multiplier` derived from average historical Worlds/MSI placement:

| League | Multiplier (example) |
|---|---|
| LCK | 1.15 |
| LPL | 1.15 |
| LEC | 1.05 |
| LCS | 1.00 |
| PCS, VCS, CBLoL, etc. | 0.90 |

These multipliers should be recalibrated annually based on updated international placement data.

---

## 11. Data Pipeline — Oracle's Elixir CSV

### 11.1 Required Columns

From the Oracle's Elixir downloadable match data CSV, the model uses only team-level rows (where `position == "team"`):

| Column | Usage |
|---|---|
| `gameid` | Unique match identifier |
| `date` | Match date — used to calculate temporal decay |
| `league` | Tournament identification → tier mapping |
| `split` | Splits within a league year |
| `playoffs` | 0 = regular season/group stage, 1 = playoff match |
| `team` | Team name |
| `result` | 1 = win, 0 = loss |
| `year` | Year of the match |

### 11.2 Processing Steps

1. Load CSV(s), filter to `position == "team"` rows only
2. Parse `date` column to a datetime object
3. For each match, compute `t = (today - match_date).days / 30.44` (months elapsed)
4. Discard rows where `t > 24` (outside Team Elo window) for team calculations; `t > 36` for league calculations
5. Map `league` to tournament tier using the tier table
6. Map `playoffs` to stage (0 = group/regular, 1 = playoff)
7. Process matches in chronological order (oldest first) to simulate Elo evolution correctly
8. On each match: compute `We`, compute `K_effective = K_base × e^(−λ × t)`, update Team Elo and League Elo

### 11.3 De-duplication

Oracle's Elixir stores each game as multiple rows (one per player + one per team). After filtering to `position == "team"`, each game still has two rows (one per team). The model processes both, updating each team's Elo independently on the same match.

---

## 12. Output

The model produces a ranked list of all teams with:

- Current Power Score
- Current Team Elo
- Current League Elo
- League affiliation
- Number of matches in window
- Δ rank and Δ Power Score vs. prior snapshot (optional)

League-level rankings are also produced for regional strength comparison.

---

## 13. Configurable Parameters Summary

| Parameter | Default | Description |
|---|---|---|
| `alpha` | 0.80 | Team Elo weight in Power Score |
| `beta` | 0.20 | League Elo weight in Power Score |
| `half_life_months` | 12 | Temporal decay half-life |
| `team_window_months` | 24 | Max age of results for Team Elo |
| `league_window_months` | 36 | Max age of results for League Elo |
| `starting_elo` | 1500 | Initial Elo for all teams and leagues |
| `min_games` | 5 | Minimum games to appear in ranked output |

---

## 14. Known Limitations and Future Work

- **Roster change tracking:** The model does not yet penalize teams for significant roster turnover between the date of a result and today. A future iteration should discount results from lineups that share fewer than 3 players with the current roster.
- **Player-level Elo:** As Riot themselves noted, integrating individual player ratings would improve predictive accuracy, particularly for newly assembled rosters.
- **Meta shift weighting:** The model cannot distinguish results across drastically different game patches. A meta-volatility modifier could discount results from epochs where the game played significantly differently.
- **λ calibration:** The 12-month half-life is a principled starting point, not an empirically optimized value. Backtesting against known tournament outcomes should drive refinement.
