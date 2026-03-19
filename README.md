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

## 4. Elo Calculation

### 4.1 Expected Result

```
We = 1 / (1 + 10^(−dr/400))
```

Where `dr` is the Elo rating difference between the two teams (or leagues, for League Elo).

### 4.2 Elo Update

Elo is updated at the **series level**, not the individual game level. The series winner (team with the majority of game wins) is treated as W = 1; the loser as W = 0.

```
P_after = P_before + K_effective × (W − We)
```

Where:
- `W` = series result (1 = series win, 0 = series loss)
- `K_effective` = the final adjusted K-factor after applying tournament tier and temporal decay (see Sections 5 and 6)

### 4.3 Series Reconstruction

Oracle's Elixir CSV data provides individual game rows. Series are reconstructed by grouping games that share the same `(date, league, split, playoffs, sorted team pair)` context key. The `game` column (game number within a series) is used for ordering when present. Games on different calendar dates are never grouped into the same series.

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
| S | Worlds (main event) | Apex annual championship |
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

Oracle's Elixir identifies tournaments via the `league` field. The canonicalization function uses exact dictionary lookup first, then falls back to substring matching to handle variants like `"LoL Worlds 2024"` or `"Mid-Season Invitational"`:

```python
# Exact lookup
LEAGUE_CANON = {
    'worlds': 'Worlds', 'wlds': 'Worlds', 'lol worlds': 'Worlds',
    'msi': 'MSI', 'mid-season invitational': 'MSI',
    'first stand': 'First Stand',
    ...
}

# Substring fallback
if 'world' in s: return 'Worlds'
if 'msi'   in s: return 'MSI'
if 'first stand' in s: return 'First Stand'
```

The `playoffs` column (0 or 1) distinguishes playoff games from group/regular stage games.

### 5.5 Worlds Regional Qualifier Treatment

Oracle's Elixir tags Worlds regional qualifier games (the play-in stage held before the main event begins) under the same `league` value as the main event. These games are **not** Tier S competition — they are regional teams competing for a slot, similar in stakes to a domestic playoff. The model detects qualifier games by comparing the game date against a known main-event start date per year, and reclassifies them as `tier = 'regional'` with a league multiplier drawn from the qualifying team's home region.

---

## 6. Improvement 2 — Temporal Decay

### 6.1 Rationale

A result from 22 months ago should carry less weight than one from last week. This is especially true when the same roster rarely persists across multiple years. Temporal decay applies a continuous discount to historical results proportional to their age, rather than a binary "inside/outside the window" cutoff.

### 6.2 Decay Function

```
decay(t) = e^(−λ × t)
```

Where:
- `t` = time elapsed since the series, measured in **months**
- `λ` = decay rate constant
- `e` = Euler's number (~2.718)

The effective K-factor for any series becomes:

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
| **8 months** | **0.0866** | **Default — recent domestic form dominates stale international results** |
| 12 months | 0.0578 | Conservative — closer to Riot's current behavior |
| 18 months | 0.0385 | Very conservative — sustained historical record weighted heavily |
**Default recommendation: 8-month half-life (λ = 0.0866)**

This means:
- A result from today contributes 100% of its Elo delta
- A result from 8 months ago contributes ~50%
- A result from 12 months ago contributes ~35%
- A result from 24 months ago contributes ~13%

Under this setting, First Stand 2025 (approximately 12 months old at time of a March 2026 ranking) contributes only ~35% of its original Elo delta. A recent LCK Cup 2026 playoff series (~2 months old) retains ~84% of its delta — meaning current domestic form decisively outweighs stale international results, which is the intended behavior.

A 6-month half-life is also valid if even more aggressive recency weighting is desired; at that setting a 12-month-old result contributes only ~25%. A 12-month half-life is available for users who prefer a more conservative, historically-weighted ranking.

### 6.4 Interaction with Tournament Tiers

Decay and tier multipliers stack multiplicatively. All examples below use the default λ = 0.0866 (8-month half-life).

A Worlds 2024 playoff result (K = 48 base) evaluated in March 2026 (~17 months later):

```
K_effective = 48 × e^(−0.0866 × 17) ≈ 48 × 0.228 ≈ 10.9
```

A First Stand 2025 playoff result (K = 24 base) evaluated in March 2026 (~12 months later):

```
K_effective = 24 × e^(−0.0866 × 12) ≈ 24 × 0.353 ≈ 8.5
```

A LCK Cup 2026 playoff result (K = 20 base) played in February 2026 (~1 month ago):

```
K_effective = 20 × e^(−0.0866 × 1) ≈ 20 × 0.917 ≈ 18.3
```

A LCK Cup 2026 regular season result (K = 8 × 1.15 = 9.2 base) played in January 2026 (~2 months ago):

```
K_effective = 9.2 × e^(−0.0866 × 2) ≈ 9.2 × 0.841 ≈ 7.7
```

At this setting, a single recent LCK Cup playoff series (K_eff ≈ 18.3) is worth more than two First Stand playoff series from 12 months ago combined (K_eff ≈ 8.5 each). This directly resolves the HLE-over-BLG symptom: a team's current domestic playoff form now clearly dominates their year-old international results.

---

## 7. Evaluation Windows

| Component | Window |
|---|---|
| Team Elo | 2 years (24 months) |
| League Elo | 3 years (36 months) |

Because decay reduces the influence of old results continuously, the hard window primarily serves as a computational cutoff rather than a meaningful weighting boundary. Results near the edge of the window already contribute very little due to decay.

---

## 8. League Elo Updates from International Play

League Elo is updated when teams from different home regions compete at an international event (Worlds main event, MSI, First Stand). The same tiered K-factor and temporal decay are applied, but at half weight (× 0.5) to avoid league Elo drifting too aggressively from individual series results:

```
K_league = K_base × decay(t) × 0.5
```

Worlds regional qualifier games are excluded from League Elo updates, consistent with their reclassification as regional-tier events.

---

## 9. Initial Elo Values

| Parameter | Value |
|---|---|
| Starting Team Elo | 1500 |
| Starting League Elo | 1500 |
| Minimum games before ranking | 5 |

Teams with fewer than 5 recorded games in the evaluation window are excluded from the ranked output.

---

## 10. Regional K-Factor Scaling

Regional K-factors are scaled by a league multiplier to account for differences in intra-regional competition depth. This prevents a dominant region from being under-represented simply because their internal Elo ceiling is lower due to stronger competition.

| League | Multiplier |
|---|---|
| LCK | 1.15 |
| LPL | 1.15 |
| LEC | 1.10 |
| LCS | 1.00 |
| LCP | 0.95 |
| CBLoL | 0.90 |

These multipliers should be recalibrated annually based on updated international placement data.

---

## 11. Ranking Eligibility

To appear in the final ranked output, a team must satisfy all three conditions:

1. **Minimum games:** At least 5 games recorded in the evaluation window
2. **Home region:** The team must have a resolved home league (LCK, LPL, LEC, LCS, LCP, or CBLoL)
3. **Anchor-year activity:** The team must have appeared in a Tier 1 domestic league during the anchor year (the year of the most recent data point). Teams that disbanded, were relegated, or are otherwise inactive in the current season are excluded.

---

## 12. International Event Placement Estimation

Oracle's Elixir CSV data does not include a placement column. Placement is estimated from each team's series record within a given tournament instance using the following heuristic:

| Condition | Estimated Placement |
|---|---|
| Won the tournament's final series | Champion |
| Lost the tournament's final series | Runner-Up |
| 2+ series wins, did not reach final | 3rd / 4th |
| 1 series win | 5th – 8th |
| 0 series wins, 2+ series losses | 9th – 16th |
| 0 series wins, 1 series loss | Play-In Exit |

The Champion and Runner-Up placements are determined precisely by tracking which team won and lost the last series played in each tournament. All other placements are bracket-depth estimates based on series wins accumulated.

---

## 13. Data Pipeline — Oracle's Elixir CSV

### 13.1 Required Columns

From the Oracle's Elixir downloadable match data CSV, the model uses only team-level rows (where `position == "team"`):

| Column | Usage |
|---|---|
| `gameid` | Unique game identifier — used for series reconstruction |
| `date` | Game date — used to calculate temporal decay and group series |
| `league` | Tournament identification → tier mapping |
| `split` | Splits within a league year |
| `playoffs` | 0 = regular season/group stage, 1 = playoff match |
| `teamname` | Team name |
| `result` | 1 = win, 0 = loss |
| `year` | Year of the match |
| `game` | Game number within a series (optional — used for series ordering when present) |

### 13.2 Processing Steps

1. Load CSV(s), filter to `position == "team"` rows only
2. Canonicalize `league` field using exact lookup + substring fallback
3. Filter out Tier 2 leagues and unknown league codes
4. Filter out teams whose name contains academy/challenger/youth identifiers
5. Parse `date` column; compute `t = (anchor_date - game_date).days / 30.44`
6. Reconstruct series by grouping games on `(date, league, split, playoffs, sorted team pair)`
7. Determine series winner (majority of game wins); flag series with tied game counts as incomplete
8. Process series in chronological order (oldest first) to simulate Elo evolution correctly
9. For each series: compute `We`, compute `K_effective = K_base × e^(−λt)`, update Team Elo and League Elo

### 13.3 Tier 2 Exclusion

The model is scoped to Tier 1 competition only. Leagues are excluded via both exact name matching and substring detection (e.g. "challengers", "academy", "masters"). Team names containing these tokens are also excluded as an additional guard against challenger-league teams appearing under a Tier 1 league code.

---

## 14. Output

The model produces:

- **Team rankings:** Power Score, Team Elo, League Elo, series record (most recent domestic split), total games in window
- **League rankings:** League Elo, average team Elo, top team, team count
- **International event breakdown (per team):** Event name, estimated placement, series W/L, game W/L
- **Domestic split history (per team):** Series W/L and game W/L per split, most recent first
- **Biggest upsets:** Series where the lower-rated team won, sorted by upset magnitude (pre-series win probability of the winner)

---

## 15. Configurable Parameters

| Parameter | Default | Description |
|---|---|---|
| `alpha` | 0.80 | Team Elo weight in Power Score |
| `beta` | 0.20 | League Elo weight in Power Score (= 1 − alpha) |
| `half_life_months` | 8 | Temporal decay half-life |
| `team_window_months` | 24 | Max age of results for Team Elo |
| `league_window_months` | 36 | Max age of results for League Elo |
| `starting_elo` | 1500 | Initial Elo for all teams and leagues |
| `min_games` | 5 | Minimum games to appear in ranked output |

---

## 16. Known Limitations and Future Work

- **Roster change tracking:** The model does not yet penalize teams for significant roster turnover between the date of a result and today. A future iteration should discount results from lineups that share fewer than 3 players with the current roster.
- **Player-level Elo:** As Riot themselves noted, integrating individual player ratings would improve predictive accuracy, particularly for newly assembled rosters.
- **Meta shift weighting:** The model cannot distinguish results across drastically different game patches. A meta-volatility modifier could discount results from epochs where the game played significantly differently.
- **λ calibration:** The 8-month half-life default is a principled starting point, not an empirically optimized value. Backtesting against known tournament outcomes should drive refinement. The slider range of 3–24 months allows experimentation.
- **Worlds main event start dates:** The qualifier/main-event boundary is currently hardcoded per year. This table will need updating each season.