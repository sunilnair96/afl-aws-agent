# 09 — Phase 1 plan: a stable, evaluated projection model

*Draft v2, 2026-10-10, revised after the owner's clarification of dates and approach. Companion to `08_APP_REQUIREMENTS.md`. Items marked **[verify]** are not yet checked; **[assumed]** are my defaults.*

## 1. Goal

Finalise the **prediction model, its code and its config** for the 2027 season, and evaluate it against the actual 2026 results. The output is one **consolidated projected score per player**:

- **Established players:** previous statistics plus config parameters.
- **New players or players with too little history:** the draft profiles and draft information (and how previous draftees performed).

These are projections, not promises. The aim is a **fairly accurate season average for established players** (not round-by-round scores), and a defensible, streamlined projection for everyone else. We are justifying and tidying the projection, not chasing exactness.

Not in Phase 1: sign-in, squads, Build Squad, trade planner, the web app.

## 2. Dates (from the owner)

| Date | Event |
|---|---|
| Now (October 2026) | Finalise the model and make any code changes, **locally first** |
| **30 November 2026** | AFL player list finalised. AFL Fantasy and SuperCoach then take time to ingest it and release their player lists |
| **5 January 2027** | Real pre-season starts: training reports and rumours begin appearing |
| Early January 2027 | Fantasy and SuperCoach player lists and prices released |
| **Last week of February 2027** | The one practice match. **The projected score is final by then** (earlier working date: 20 February). After it, only injuries change a projection |
| March 2027 | Season starts, with a Round 0 |

Phase 1's model and config work should finish before the 30 November list, so that December to February is spent running the real lists and applying pre-season news, not changing the method.

## 3. Evaluation protocol (as stated by the owner)

- **Train on seasons 2023 to 2025, validate against the actual 2026 stats.** This is what `validate()` in the local `season_projection.py` already does: the calibration trains on the as-of years 2022, 2023 and 2024 (targets 2023, 2024, 2025), then projects 2026 from 2025 and compares with what happened. Current numbers: AF MAE **7.95**, Spearman 0.81, top-20 hit 13; SC MAE 8.51, Spearman 0.80, top-20 hit 13 (last season's average: 9.78 and 10.69).
- **Test established players and new players separately.** The current validation reports one blended group of about 500 players; it needs a breakdown (below). It cannot test new players at all, because they have no previous average.
- **Judged on season average,** not round by round.

### What the evaluation will report

| Group | Test | Metrics |
|---|---|---|
| Established (a prior season with enough games, 2 or more seasons of history) | Projected 2026 average against actual | MAE, **bias** (mean error), Spearman, top-20 hit; also by age band, by position and by projected level (top tier, middle, bottom) |
| Low-data (returning from injury, few games last season, mature-age recruits) | The same | The same, with sample size shown |
| New (draftees, no AFL history) | Draft-profile projection against their actual 2026 averages (Sharp, Farrow, Lindsay, Retschko, J. Smith and others) | MAE, rank order; small sample stated |
| Named cases and trajectory pairs | As in `08_APP_REQUIREMENTS.md` 4c and 4d | Reported; the pairs check relative order |

A gap already visible in the existing notes: the model shrinks toward the average (in 2026 the top tenth was predicted 95.2 against an actual 100.2), and **all six trajectory examples are projected to decline**. The breakdown by projected level will show whether that shrinkage is a real accuracy problem.

### A caution on using 2026 repeatedly

Each time the model is adjusted after looking at 2026 results, 2026 becomes a little less independent. Suggestion, not a requirement: keep a log of each change and the 2026 result it produced, and re-check any change that helped on 2026 against an earlier target year (the same method with 2025 as the target, trained on 2022 to 2024). The final 2027 model then trains on 2026 as well, with no independent test left; that is normal.

### 3a. WP1 baseline (observed 2026-10-10; as-of 2025 projected against actual 2026; one hold-out year)

Run by the new `season_projection_eval.py` in the local app. It reproduces `validate()` exactly (AF MAE 7.95, SC 8.51; 9 unit tests pass). Established = 10+ games in 2025 and 5+ in 2024 (assumed definition).

| AF | n | MAE | Bias (projected minus actual) | Naive "last avg" MAE |
|---|---|---|---|---|
| All with a last average | 500 | 7.95 | -1.33 | 9.78 |
| **Established** | 339 | **7.97** | -1.59 | **8.61** |
| Low data | 161 | 7.89 | -0.78 | 12.23 |
| No games in 2025 | 13 | 6.51 | +0.90 | n/a |

What stands out (AF; SC shows the same shapes unless stated):
1. **For established players the model beats "last season's average" by only 0.64** (7.97 vs 8.61). Almost all of the overall gain comes from low-data players (7.89 vs 12.23). Since the owner's priority is accuracy for established players, this is the main room to improve.
2. **Young and prime players are under-projected:** bias -4.87 at age 21 or under, -3.49 at 22 to 24, -2.60 at 25 to 26; slightly over at 27 to 29 (+1.40). Named examples: Daicos projected -5.1 against an actual +8.6; Sheezel -4.4 against +7.1; Ashcroft +0.05 against +11.7; Gulden -0.6 against +10.2.
3. **Rucks are over-projected, but this is explained, not a model flaw (owner, 2026-10-10: a 2026 rule change reduced ruck scoring).** Rucks (n=18): AF MAE 13.99, bias +9.91, worse than naive (13.16); SC MAE 15.50, bias +8.16. The data agrees that 2026 was a break: established rucks' average change was +2.0 (2022 to 2023), -0.9 (2023 to 2024) and **-8.7 (2025 to 2026)** in AF, against +1.4 for non-rucks in the last of those (SC: -7.0 against -0.4). A model trained on earlier seasons cannot know about a rule change.
4. **By level:** the top 10% is projected slightly high (AF +1.48, SC +4.13), the bottom half low (AF -3.15). Consistent with shrinkage toward the middle.
5. **Older players:** Steele Sidebottom's drop was larger than projected (projected 76.5, actual 63.1); Neale and Parker, both projected to decline, improved (errors -12.4 and -14.0).
6. **Trajectory pairs in this setup:** model order correct for 2 of 3 (AF) and 3 of 3 (SC). Earlier, on the 2027 projection (as-of 2026), Sheezel vs Parker failed; here it passes. **A single pair is noisy**; do not tune to one pair.

**Consequence of the ruck break for the 2027 model (a decision, not yet made):** the 2025-to-2026 transition is a training example for the 2027 calibration. Left in as a normal ruck transition, it teaches the model that rucks decline by about 9, which would under-project rucks in 2027 (their 2026 averages already reflect the new rule). Options: (a) leave it out of the training set for rucks; (b) add a season or rule-change indicator; (c) correct with a config rule. Only one post-change season exists, so none can be validated against actual data; it is a judgement to record. The evaluation should also report the aggregate **with and without rucks**, so the 2026 break does not hide or distort the other results.

Caveats: one hold-out year; small groups (rucks 18, age 21 or under 26); the established definition is an assumption.

### 3b. What goes with improving on last season's average (observed 2026-10-10; exploratory, not yet code in the app)

Method: established player-seasons across the three available transitions (as-of 2022 to 2024 into 2023 to 2025 targets, plus 2025 into 2026 for the rank chart): 1,034 AF rows, "Improve" = actual next average minus last average. These are associations, not causes; the test used only the attributes already in the cached table plus club change and 2026 start prices.

| Attribute | Association with improving | Stable across years? |
|---|---|---|
| Age in the projected season | **Younger improves:** mean change +4.2 (21 or under), +1.6 (22-24), -0.7 (25-26), -2.1 (27-29), **-4.1 (30+)**; share improving by 5+ points: 45%, 40%, 32%, 23%, **20%** | Yes (rho -0.18, -0.22, -0.29) |
| Level (last average) | **Higher regresses:** mean change -7.7 for 100+ against +0.8 below 80 | Yes (rho -0.24, -0.24, -0.37) |
| Gap to own best season | **Further below own best improves:** +2.7 if 8+ points below, -2.3 otherwise | Yes (rho +0.19, +0.20, +0.24) |
| Change last year | **A big jump last year tends to reverse** | Yes (rho -0.22) |
| Games missed last year | Small plus (+0.12) | Yes |
| Last-6 vs previous-6 form | Weak (+0.06) | Mixed |
| Role-trend measures (disposal, clearance, hitout, tackle trends) | **About zero** (rho 0.01 to 0.04): they do not capture "role change" | n/a |
| Club change (n=58) | +1.7 against -1.0 for the rest; weak | One sample |
| Position change | **Cannot be tested:** no position change appears anywhere in the data, which suggests positions are copied across seasons [verify] | n/a |

Together these explain only about 18% of the variation in improvement (simple linear fit). **Most improvement is not explained by the attributes we have.**

**Other results**
- **Premium returning from a short season** (best average 100+, under 14 games last year; n=14): on average they came back **2.9 below last year's average and 23.7 below their own best**. Gulden (+10) and Docherty (+36 from one game) rose; Oliver, Coniglio, Treloar, Caldwell and others fell. Do not assume a bounce back; small sample.
- **Discounted players** (start price implies an average 8+ below last year's): they did **not** improve on last average (-2.9, n=25) but returned **+9.6 points over their price** (slightly-below-fair priced +11.1, n=36; fairly priced +1.7). So discounts matter for value (points over price), not for improvement. Prices are derived estimates, one season only.
- **Points over price by age (all players):** +9.6 (21 or under), +5.4 (22-24), +3.4 (25-26), -0.9 (27-29), -2.5 (30+).
- **Rank movement among the 48 listed players** (rank by 2025 average against rank by actual 2026): Spearman 0.45 (projected rank against actual: 0.48), so the projection barely improves on last year's order at the top. Movement correlates with age (-0.40) and with last average (-0.51: the higher ranked fell more). Under-24s rose 7.7 places on average (n=11), players 30 and over fell 4.9 (n=15). Biggest risers: Ashcroft +28, L. Jackson +24, Heeney +22, Neale +22, Rankine +22. Biggest fallers: English -34, R. Marshall -30, Caldwell -30, McCluggage -29, Dunkley -22; four of the seven biggest fallers are rucks (the 2026 rule change).
- **Consistency:** the spread of a player's 2025 game scores does **not** predict how far his next season average moves (rho 0.03; mean surprise 8.6, 8.7 and 8.5 for the steadier, middle and most variable thirds). Game-to-game consistency is not what makes a season average predictable. Availability (games played) has not been tested.

### 3c. Owner's design direction (2026-10-10): keep the projection rule, add a common shortlist with attributes

**Decision:** there is not much value in changing the projection rule itself. Instead, maintain a **common shortlist** of players with attributes that the projection program uses to **adjust the base projection**, plus **score bands** per position. The base model only needs checking that it uses age correctly.

**Age check (observed, established players, as-of 2025 into 2026):** the model uses age in the right direction but too weakly. Mean projected change against mean actual change, by age in 2026: 21 or under +1.9 against +6.8; 22-24 +0.4 against +3.9; 25-26 -0.1 against +2.6; 27-29 -0.8 against -2.2; 30+ -3.4 against -2.4. So the model under-projects the rise of players 26 and under by 3 to 5 points and over-projects 27 to 29 by about 1.4. One season only; 2026 may have been a strong year for the young.

**Score bands (counts of players with 10+ games, AF; position by the priority Ruck, Midfield, Defender, Forward):**

| Position | Band | Actual 2025 | Actual 2026 | Projected 2027 |
|---|---|---|---|---|
| Mid | 115-120 | 1 | 3 | 0 |
| Mid | 105-115 | 6 | 5 | 5 |
| Mid | 100-105 | 8 | 8 | 4 |
| Mid | 90-100 | 21 | 24 | 20 |
| Mid | 80-90 | 29 | 26 | 36 |
| Mid | 70-80 | 29 | 45 | 55 |
| Ruck | 110+ | 2 | 1 | 0 |
| Ruck | 100-110 | 4 | 2 | 1 |
| Ruck | 90-100 | 5 | 2 | 5 |
| Ruck | 70-90 | 7 | 10 | 10 |
| Def | 100+ | 4 | 2 | 0 |
| Def | 90-100 | 7 | 9 | 7 |
| Def | 80-90 | 13 | 16 | 16 |
| Def | 70-80 | 22 | 26 | 28 |
| Fwd | 100+ | 0 | 0 | 0 |
| Fwd | 90-100 | 1 | 0 | 0 |
| Fwd | 80-90 | 5 | 3 | 0 |
| Fwd | 70-80 | 9 | 15 | 12 |

Reading: the owner's "3 mids at 115-120" matches the actual 2026 count. The 2027 projection puts **no** player in the top bands (0 mids at 115+, 0 rucks at 110+, 0 defenders at 100+): the top of the distribution is compressed. The owner's "10 mids at 105-115" does not match: the count is 5 here (6 in 2025). **[to reconcile]** Forward bands were not finished.

**Proposed shortlist attributes (draft, developer-maintained, shared by all users).** Identity fields come from the data (player, 2027 team, position, age); the maintained ones are:

| Attribute | Purpose |
|---|---|
| `Archetype` | Gun-Reliable (young but already peaked: Sheezel, Daicos, B. Smith, Ashcroft), Gun (established or rising: L. Jackson, Rankine), Improver (young with signs: draft profile or scoring above average), Value (discounted), Draftee (cash cow), Returning (from injury), Veteran-Top (30+ only if at the top), Avoid (with reason, such as Key Position) |
| `Band` | Position and score band (for example Mid 105-115) |
| `Adjust_Type`, `Adjust_Value` | How to adjust the base projection: fixed value, add or subtract points, or a factor |
| `Adjust_Reason` | Role change, club change, shared duties ended, rule change (rucks 2026), expert view |
| `Adjust_Source`, `Adjust_Date` | Where the adjustment came from (news, expert, own judgement) and when |
| `Role_Now`, `Role_Change` | For example "number one ruck at the new club"; position or role change flag with a note |
| `Games_Last_Year`, `Injury_Status` | Availability; injury status from the AFL injury list (Indefinite = 0 for future rounds; Test = highlight only) |
| `Confidence` | How much to trust the projection for this player (high, medium, low) |
| `Core_Target` | Marked as a player the target team should contain (for example L. Jackson). Locks themselves stay per user |
| `Notes`, `Updated_On`, `Updated_By` | Audit |

Worked examples: **Rowan Marshall** to Geelong: `Adjust_Type` add points, `Adjust_Value` +20 on his 2026 average (83.3 to about 103), reason "number one ruck at the new club, shared duties ended, 2026 ruck rule change". **Luke Jackson:** Gun, `Core_Target`. **Rankine:** Gun, role change (position), source expert views. The existing local `shortlist_exceptions_2027.csv` (columns Competition, Player, Team, Action, Reason, Date_Added; empty today) handles include and exclude only; the attributes above would be a second file or an extension.

### 3d. Shortlist attributes: build slices (local app first)

| Slice | What | Status |
|---|---|---|
| 1. Backend core | `config/shortlist_attributes.yaml` (ladder, roles, tag list, rules), `src/common/shortlist_attributes.py` (band to number, validation, apply to the saved shortlist, rookies from the player master), `tests/test_shortlist_attributes.py` | **Done 2026-10-10.** 25 new tests pass; the whole backend suite passes (471). No existing file changed. |
| 2. API | Endpoints to read and edit the attribute rows per competition, a choices endpoint (bands, roles, tags), and `/shortlist` returning the merged rows | **Done 2026-10-10.** Not blocked by the season lock (next season's view). |
| 3. Screen | Role, Band and Tags columns on the Shortlist page with dropdown editing, a tag filter, an "Add a player by band" form, and the final score with its source | **Built 2026-10-10; build and lint clean. Not yet checked in a browser.** |
| 4. Evidence columns | Games at or above two thresholds over the last 3 seasons (AF 100/120; SC 110/130 assumed); draft profile for players with no AFL games | **Done 2026-10-10** (SC thresholds to confirm) |
| 5. Next five rounds | Projected score and opponent for the next 5 rounds | Later: needs the 2027 fixture and the player list |

Files to change in slices 2 to 4 (listed now, **nothing touched yet**):
- Slice 2: `src/api/main.py` (new routes), `src/api/schemas.py` (new models), `src/common/player_compare.py` (`load_shortlist` merges the attributes), `tests/test_api.py` or a new API test file.
- Slice 3: `frontend/src/api.ts` (types and calls), `frontend/src/pages/ShortlistPage.tsx` (columns, editing, filter), and its styles if needed.
- Slice 4: a new `src/common/` helper for the 100 and 120 counts, `src/common/draft_dossier.py` (read only), the same API and page files.

## 4. Local first, then AWS

**Step 1 (now): test and improve locally,** using the data and code already in the local app. **Step 2 (later): see how SageMaker, or any other AWS modelling feature, does the same job and compare it against the local model** on the same evaluation.

The account is credit-based (about $158 of $160 left on 2026-10-10), so any AWS service draws on credits; check each service's terms on the Billing "Free Tier" page before relying on it **[verify]**. The data is tiny (a season is about 160 KiB as Parquet) and the local base projection takes about 4.5 minutes, so the AWS comparison is mostly a question of fit and learning value, not scale.

| AWS option | Notes |
|---|---|
| Lambda container image (15-minute limit) | Probably enough for one projection run; the full evaluation may exceed it **[verify]** |
| SageMaker Processing or Training job | Runs the same container for longer; also the natural place to learn SageMaker's tooling **[verify]** price |
| SageMaker built-in algorithms or AutoML-style features | A true comparison against the local model; availability and cost **[verify]** |
| Fargate task | Simple container; more setup |
| Glue | Data preparation only. Not recommended for the model |

## 5. Work packages (each ends at a gate, not a date)

**Step 1, local**

| WP | What | Gate (done when) |
|---|---|---|
| **1. Evaluation harness** | Extend the existing validation to the breakdown in section 3, as one command that reads the cached season tables and writes nothing. Log each run (code version, config hash, input hash) | Reproduces AF MAE 7.95 and SC 8.51; prints the group, age, position and level breakdowns |
| **2. Error analysis** | Study where the established-player projection is wrong: shrinkage at the top, age curve (growth to about 24, plateau 24 to 26, decline from 27), Neale-type exceptions, club moves, injury returns, recent-form weighting | A written list of weaknesses ranked by effect on season-average accuracy |
| **3. Config and rules** | Rule schema in config (fixed, factor with a base, range, drop-lowest-N, cap, floor), special-rule engine, thresholds from config | Rules change a projection with no code change; tests pass; rule audit log |
| **4. Established-player improvements** | Try the improvements from WP2, one at a time | Each is judged on the evaluation: accepted only if it helps a named weakness without worsening the aggregate bar [tolerance to agree; my default 0.3 MAE] |
| **5. New-player model** | Inputs: `DraftProfiles` (dossier and its stats file), previous draftees' results by pick band and position, and the structured contents of the `DrafteeAnalysis` PDFs (2021, 2022, 2023, 2024 analyses and the 2026 cash-cows guide) | A stated method in config; error against the 2026 draftees' actuals, with the small sample stated |
| **6. Consolidated output** | One projection file: established projections plus new-player projections plus special rules, with a source tag per row (`model`, `draft_profile`, `rule`) | One file per season, both AF and SC; every row says where its number came from |
| **7. 2027 special cases** | Returning-injury rule, the Marshalls and other cases, as config rules | Cases produce the intended numbers, with reasons recorded |

**Step 2, AWS**

| WP | What | Gate |
|---|---|---|
| **8. Parity on AWS** | Run the same code on AWS from S3 data, with pinned dependencies and seeds | The AWS output reproduces the local output twice in a row |
| **9. AWS modelling comparison** | SageMaker or another AWS modelling feature on the same evaluation | A table of results against the local model; a recorded decision on which becomes production |
| **10. Dress rehearsal and freeze** | End-to-end on the flagged 2027 test data; later the real lists; freeze before the practice match | Frozen file with timestamp and hash |

## 6. Recommendations

1. **Start with WP1.** One reproducible evaluation, with group breakdowns, before any model change. Every later change is judged against it.
2. **Fix the process, not individual players.** Named cases explain the model; tune rules and features, not the output for one player. Individual players are what the special-rules config is for.
3. **Pin everything** when moving to AWS (dependencies, base image, seeds, input hashes). An unpinned dependency changed a package earlier in this project.
4. **Draftees are a small sample.** Report ranges and avoid tuning to individual players. The `DrafteeAnalysis` PDFs are third-party material: extract once into a local table with the source named, and keep it out of git.
5. **Keep the first step cheap.** The evaluation reads the cached season tables and runs in seconds; the 4.5-minute rebuild is only needed when features change.

## 7. Open questions

1. **Definition of "established":** a prior season with at least how many games, and how many seasons of history? [assumed: 10 games in the prior season and 2 or more seasons]
2. **Acceptable regression** on the aggregate bar when a change helps a named case. [assumed: MAE no more than 0.3 worse]
3. **The draftee PDFs:** are the tables of rankings and projected scores the useful part, and did the analysts record what happened next year?
4. **When the AFL draft is held** (clubs and picks fill the dossier).
5. **After the freeze:** do the 3 to 4 round look-ahead rows against the fixture (FR23) still get produced from the frozen scores? [assumed: yes; results do not re-run the model]
6. **SageMaker:** do you want to learn it as part of WP9, or only compare it if it is quick to set up?
