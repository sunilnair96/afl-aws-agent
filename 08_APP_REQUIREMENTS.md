# 08 — The AWS app: requirements (version 0.1, draft)

*Written 2026-10-10 from the owner's answers in conversation. This is a draft that will change. Items marked **[assumed]** are my defaults, not yet confirmed. Items marked **[verify]** are things nobody has checked yet. No identifiers are used here (placeholders only).*

## 1. Two apps, one data source

| | Local app (final) | AWS app (this document) |
|---|---|---|
| Where it runs | Owner's machine, `http://localhost:5173/` | AWS free-tier account, front end `http://localhost:5174/` |
| Model access | Anthropic API key on the machine | Amazon Bedrock through the account's IAM (no Anthropic key) |
| Purpose | The real tool for the 2027 season | Learn AgentCore, Lambda, Cognito, Cedar and measure what it costs to run |
| Later | Possibly rebuilt on Claude SDK or LangChain | Cost report after the build |

The AWS app is a **new** app. It consumes data the local app already produced (uploaded to S3; a Glue pipeline exists). It does not call the local app.

Window: the free-tier account lasts 6 months or until the credits are used (about $158 reported on 2026-10-10). The 2027 season starts in **March 2027**.

## 2. Users

| User | Role |
|---|---|
| **User A**, **User B** | Use the tool. Each has their own squads, preferences and leagues. They never see each other's data. |
| **Developer (`<DEV_IAM_USER>`)** | Builds and operates the tool. Runs the data refresh and the email intake. Not an app user. |

## 3. The two products

**P1. Team builder and weekly trade advisor** (AFL Fantasy Classic and SuperCoach, salary cap)
- Build a starting team.
- Each week, advise trades to improve the team using injuries, form, position changes and news.
- Advice only. The app never executes a trade or logs into the game.

**P2. Draft planner** (AFL Fantasy Draft and SuperCoach Draft; **Redraft leagues only**, no salary cap)
- Visited once, before round 0.
- Input: league size, squad structure and rules (differ by league; entered per league). Output: a ranking by position scaled to league size, a draft-order position and a round-by-round pick plan.
- Same strategy for both games, different output format.
- Out of scope: Keeper and Dynasty leagues, rookie draft, future-pick trading.

## 4. Functional requirements

| # | Requirement |
|---|---|
| FR1 | Sign in with Cognito. Every request carries the user's token. |
| FR2 | A user can store squads (one per competition), preferences and leagues. [assumed: squad entered by CSV upload first] |
| FR3 | Preferences per user: age cap (A: 32, B: 33), avoid list, premium locks. Applied by the **tool code**, never left to the model. Each avoid-list entry carries a **reason tag** (for example `Key Position`, `Injury`, `Past Prime`), as the local app already does with free text. The data has no key-position field (only DEF, MID, RUC, FWD), so key-position exclusions are the user's own entries. [assumed: optionally the app can *suggest* candidates from height and position, never exclude automatically] |
| FR4 | Projections are **deterministic**: the app reads stored per-round predictions; it never re-runs a model. |
| FR5 | Trade advice: return a small set of options with the numbers behind them; the user decides. |
| FR6 | The user can state "player X is injured / in doubt". [assumed: applies only to that user, for the current round, saved with the date] |
| FR7 | Expert views (from the email knowledge base) are shown **with source and date**, beside the numbers, and never change them. Example: "<source> suggests <player> is likely to be tagged this week." |
| FR8 | Draft plan: rank by position scaled to league size (value over replacement) [assumed], pick plan for the user's slot, a simple "what if I pick a ruck first" comparison [assumed]. |
| FR9 | Rules (caps, squad shape, trade limits) come from config, never hard-coded; they change by season. |
| FR10 | Every tool call is logged (tool, arguments, allowed or denied) for debugging and cost counts. |
| FR15 | **Special rules are config, not hand edits (changed 2026-10-10).** The projected score is final once built: **prediction model output plus special rules driven by config**, each rule a formula or a fixed value, for example *returning from injury: 80%*, *fixed value 65*, *average after dropping the N lowest games*. A rules file lists player, rule, value and a reason, in the same style as the local app's shortlist exceptions file. Only the developer changes it. Pressing **Project** applies the rules and rewrites the single season projection file in place; every change to the rules file or the file is appended to an audit log. **Rule schema (generic; the 80% example is only one instance):** `player`, `rule_type` (`fixed`, `factor`, `range_midpoint`, `drop_lowest_n`, `cap`, `floor`), `base` for factors (`model`, `last_season_avg`, `best_avg_last_2_seasons`), `value`, `applies_from_round`, `applies_to_round`, `reason`, `added_on`. New rule types are added in code once, then used from config. |
| FR16 | **Users do not touch projections.** Users A and B only choose which players to **lock**, **add to their shortlist** and **exclude**. Age cap, avoid list, locks, squads, versions and league settings stay per user. |
| FR17 | ~~Price discount per special player~~ **Not an app feature (owner, 2026-10-10).** The real player list and prices will be loaded, so the app never calculates discounts. A discount field exists **only in the test-data generator's assumptions file**, to fake a price for test purposes. |
| FR20 | **Projection is the core function** (section 4b): one **projection file per season**, kept in place, with both AFL Fantasy and SuperCoach scores for every player; one projected score per player shared by all users; **final** once frozen (FR25). |
| FR23 | **Opponent handling.** Pre-season there is no fixture, so the season projection is **fixture-independent** (an average over all opponents, as the local `season_projection.py` already does). Once a fixture exists, the app projects each player against the **upcoming fixture for the next 3 or 4 rounds** (config: `rounds_ahead`), as SuperCoach shows. Round projections are rows in the same season file (see 4b), recomputed in place after each round or fixture change. Pre-season squads use the fixture-independent projection; the trade advisor uses the next rounds. |
| FR24 | ~~Overrides scaled by round ratio~~ **Replaced:** the projection is final. Rules (FR15) are applied to the model output the same way for the base and for each round: a fixed value replaces it; a factor multiplies it; a method (for example drop-lowest-N) changes the input average. No per-round scaling logic. |
| FR25 | **Projection lifecycle:** **Draft** (iterate: model, rules, Project, review, repeat) then **Frozen**. **The projected score is final by the one pre-season practice match in the last week of February 2027** (working date 20 February); after that only injuries change it (owner, 2026-10-10). The AFL player list is finalised by 30 November 2026; AFL Fantasy and SuperCoach release their lists in early January; real pre-season news starts about 5 January. A frozen file records its freeze time and hash; changing it needs an explicit unfreeze with a logged reason. Not a copy of the file. |
| FR26 | **New players with no history** (draftees such as Cody Walker): the projection uses the **draft dossier** and **how previous draftees performed** (by pick band and position), as a stated method in config. More draft classes improve it; only one class (the 2025 draft, 2026 season) has been looked at so far. |
| FR27 | **After the freeze (from 20 February), only injuries change a projected score** (see 4e). The model is not re-run after each round's results. [to confirm: the 3 to 4 round look-ahead rows against the fixture are still produced from the frozen scores; see `09_PHASE1_PLAN.md` open question 1] |
| FR28 | **Projection acceptance tests (trajectory pairs):** projections must order known young-rising and old-declining players correctly. Pairs: Daicos vs Neale, Sheezel vs Parker, Jackson vs Gawn. See section 4c for the current result. |
| FR29 | **Evaluation set for the production projection** (section 4d): a fixed, versioned set of tests run on every change to the model or rules. It has three levels: aggregate hold-out metrics, labelled named cases, and trajectory pairs. |
| FR30 | **Known structural breaks are configurable.** A rule change that shifts a group's scoring (2026: rucks scored less than in 2025; owner, 2026-10-10) is declared in config (season, group, treatment: exclude from training, indicator, or correction factor), applied by the model, and reported in the evaluation. The data shows established rucks changing -8.7 AF (-7.0 SC) from 2025 to 2026 against about 0 in earlier years. |
| FR31 | **Shortlist attributes (decided 2026-10-10; built first in the local app).** Each shortlisted player can carry a **Role** (DEF, MID, RUC, FWD: the position he is used in), a **Band** and **Tags**, **per competition** (AF and SC are separate). Choices come from fixed lists in config, never free text. A Band is entered only where the owner wants to override the model's projected average; a blank Band means the model's projection. |
| FR32 | **Band ladder.** One ladder for both games: 30-40, 40-50, 50-60, 60-70, 70-80, 80-90, 90-100, then steps of 5: 100-105, 105-110, 110-115, 115-120, 120-125, 125+. A band's number is its **lowest edge** (50-60 is 50; 125+ is 125). The ladder lives in config. The low bands let rookies whose score cannot be projected be added. A banded player is shortlisted with that number as his projected score (config: `band_implies_inclusion`); a banded player with no projection row (a rookie) is added from the player master. |
| FR33 | **Tags are information only.** A player can have several (for example #Gun, #Great Junior Scores, #Past his Peak, #Target). No scoring, shortlisting or squad logic may read them; a test guards that. They are context for the agent. The existing rule-based **Type** (Emerging, Established, Returning, Other) is left as it is, because the shortlist rules use it (Emerging players need a smaller projected gain). |
| FR34 | **Shortlist screen shows (owner's wish list).** Now buildable: for established stars the number of games scoring over 100 and over 120 (SuperCoach thresholds to confirm), and for new players the draft-profile information. **Later, once the fixture and player list exist:** each player's projected score for the next 5 rounds with the opponents. |
| FR21 | **Shortlist before Build Squad.** For AFL Fantasy and SuperCoach, Build Squad chooses only from the shortlist plus the user's locks (about 50 to 60 players of interest, not the whole list). Build Squad refuses with a clear message if no current shortlist exists. |
| FR22 | **Draft planner projection:** about 300 players, sized by squad size and league size (a bigger league needs a deeper projection). |
| FR11 | **Build Squad:** the user presses Build Squad; the optimiser runs against the user's locks, avoid list and the current season projection, and can be re-run after any edit. Two steps, as in the local app: a fast, rerunnable *build* that never saves, then an explicit *save*. Never auto-save. On save the app asks: **save as a new version / overwrite the active version / discard**. |
| FR18 | **Explanation of how the team was made.** The solver returns a structured explanation and the agent narrates **only from it**: locks placed (and their projected scores); players removed and why (Round 0 teams if configured, below the minimum-average floor, avoid list, age cap); budget used and left; for each slot the pick, price, projected score and the next-best alternative; which rules were binding (cap, positions, club minimum). The agent never invents reasons. |
| FR19 | **Config over code.** Rules live in config files, as in the local app (squad structure and cap, guardrails, shortlist rules, minimum-average floor, club minimum, Round 0 exclusion, magic numbers); config is stored separately from user data, versioned, and each projection baseline and squad version records which config version it used. Carried over from the local app: `fantasy_squad_config.yaml`, `supercoach_squad_config.yaml`, `guardrails.yaml`, `shortlist_2027.yaml` and the shortlist exceptions file. |
| FR12 | **Squad versions:** a user keeps several named, dated versions of a squad (pre-season changes as rumours, injuries and role changes appear), can compare two versions, and marks one as the active version. [assumed: unlimited versions, per competition] |
| FR13 | **Season lifecycle** (mirrors the local app's season status: Planned, Open, InSeason, Closed). Pre-season: build and versions. After Round 0, actual Round 0 scores can be loaded and a new build uses them. When the round 1 lineup is announced, named and unnamed players are flagged. **At the start of round 1 the active version is frozen as the baseline and Build mode becomes read-only.** |
| FR14 | **In season:** only Manage Trades is available (propose-only, within the round's trade limit), starting from the frozen baseline and each round's current squad. |

## 4-scope. What is needed before March 2027, and what comes after

| Before March 2027 (build first) | After the season starts |
|---|---|
| Sign-in, per-user preferences, avoid list, locks | **Trade planner** (weekly) |
| Build Squad, squad versions, lock at round 1 | Weekly injury, form, news handling |
| Draft planner | Round-by-round prediction refresh |
| Questions with expert insights (source and date) | |
| Test data for 2027 | |

## 4b. Core pipeline: projection first (decided 2026-10-10)

**Projection is the core function of the app.** Everything else consumes it.

| Step | Scope | Shared or per user | Notes |
|---|---|---|---|
| 1. Base projection | **All players** (about 600 to 800) | Shared | Heavy; re-run only when the underlying data changes (the local app already caches this step) |
| 2. Overrides and the season file | Special players (FR15) | Shared (developer only) | Cheap; pressing **Project** applies overrides to the cached base projection and rewrites the **single season projection file** in place |
| 3. Shortlist | About 50 to 60 players | Shared (rules in config, plus an exceptions file) | Built from the season projection. **Prerequisite for Build Squad.** |
| 4. Locks and exclusions | Per user | Per user | A user's own locks and avoid list. **A locked player's projected score is the shared value from the season projection; a lock never carries its own number.** |
| 5. Build Squad | Shortlist plus the user's locks | Per user | Solver, then explanation (FR18). Saves the projected score used for each player inside the squad version, plus the projection file's timestamp and content hash and the config version. |
| Draft planner | Projection for about **300 players**, depending on squad size and league size | Per league | Reads the season projection directly; does not use the shortlist |
| DFS (not in scope now) | Players named in a match lineup | | Match-based; would reuse the same projection engine for those players |

**One file per season, no versions of it (owner, 2026-10-10).** The season projection is a single file (long format: player, round, opponent, AF projection, SC projection). Rows with `round = 0` and opponent `ALL` are the fixture-independent base; rows with `round = N` are projections against the actual fixture for the next 3 or 4 rounds, added once a fixture exists. The file is **rewritten in place** (S3 versioning is switched off for this prefix); only the audit log (FR15) keeps history. Before the fixture exists it contains base rows only.

**Staleness rule:** the file carries an `updated_at` timestamp and a content hash. A shortlist records the hash it was built from, and a squad version records the hash plus the projected score used for every player. If the file changes after a shortlist was built, the shortlist is marked stale and Build Squad warns the user. A saved squad version stays explainable because it holds its own scores, even though the older file is gone.

**Where projection runs.** Recommended: in AWS as a batch job triggered from the app, with the cheap "Project" step separate from the heavy base-projection step. The local app's 4.5-minute `--rebuild` and its cache are the model for that split. The compute choice (Lambda container image or a container task, with a Step Functions workflow) is **[verify]**: the model libraries and run time have not been tested on AWS.

## 4c. Projection check: the trajectory pairs (observed 2026-10-10, from the existing 2027 projection files; not re-run)

AF change from 2026 average to projected 2027 (SC in brackets): **Daicos −10.5 (−6.8)** vs **Neale −12.4 (−12.9)**; **Sheezel −7.6 (−11.1)** vs **Parker −7.0 (−9.8)**; **Jackson −4.0 (−6.3)** vs **Gawn −8.6 (−12.9)**.

- **Relative test** (the young player should fall less than the old one): 2 of 3 pairs pass in both AF and SC. **Sheezel vs Parker fails** in both.
- **Absolute test** (the rising player should be projected to rise): **0 of 3 pass.** All six are projected to decline from 2026, the young ones less. This looks like regression to the mean after outlier seasons (the model's season-level calibration is trained on real year-to-year changes); that is my reading, not a confirmed cause.
- **Consequence for the shortlist:** the 2027 shortlist rule requires projected improvement of at least +5 (+2 for emerging players). **Daicos, Sheezel and Jackson are all excluded by it.** Since Build Squad chooses from the shortlist plus locks (FR21), premium players only enter a squad as **locks**. That matches the local design, and it is why locks matter.
- **Returning players** (Tom Green, Nicholas Martin) have no 2026 average; the model projects AF 88.7 and 85.1 (SC 88.6 and 80.5). The 80% rule needs a defined base (see Q14).


## 4d. Projection evaluation set (decided 2026-10-10)

The 2026 actual scores are the answer key. Three levels:

1. **Aggregate hold-out** (exists in the local app, `PRESEASON_2027_PLAN.md`): project 2026 from data up to 2025, compare with actual 2026 averages for 500 players. Current numbers: AF **MAE 7.95**, Spearman 0.81, top-20 hit 13; SC MAE 8.51, Spearman 0.80, top-20 hit 13. The naive "last season's average" scores 9.78 and 10.69. These become the **pass bar**: a change must not make MAE worse by more than a set tolerance [to agree].
2. **Labelled named cases** (reported, **not pass/fail**: "established players the model should predict, not necessarily correctly"). Categories and players supplied by the owner; actual averages are from the local 2026 data (AF, home-and-away):

| Category | Player | 2025 | 2026 | Note |
|---|---|---|---|---|
| New club | Deven Robertson | 15.5 (4 g) | 73.5 (4 g) | Moved to West Coast; scored more, then injured; only 4 games each year |
| Age decline | Scott Pendlebury (38) | 86.5 | 79.2 | |
| Age decline | Steele Sidebottom (35) | 86.0 | 63.0 | Largest drop in the group |
| Rookie, no history, strong draft profile | Jagga Smith | none | 82.0 (23 g) | Missed 2025 injured |
| Exception, lock from pre-season news | Lachlan McAndrew | none | 85.2 (22 g) | |
| Draftees | Dyson Sharp, Patrick Retschko, Josh Lindsay, Jacob Farrow | none | 70.9, 71.1, 65.9, 71.9 | Retschko spelling as matched in the data |
| Mature-age recruits | Marcus Herbert, Joel Fitzgerald | none | 76.2, 75.0 (10 g each) | |
| Established | Bailey Smith 116.3 to 118.2; Daicos 108.1 to 116.7; Sheezel 109.2 to 116.2; Dawson 110.2 to 112.9; Gulden 102.3 to 112.5 (11 g, injured); Gawn 113.3 to 110.0; Ashcroft 95.3 to 107.1; Butters 99.8 to 104.5; Jackson 94.8 to 102.8; Bontempelli 111.3 to 102.5; **Zachary Merrett** 102.8 to 104.1 | | | The data stores him as "Zachary Merrett", not "Zach Merrett". The evaluation set, the draft order and the test-data assumptions must use the data's name, or a name alias (the local alias file has no entry for him) |

3. **Trajectory pairs** (relative order): Daicos vs Neale, Sheezel vs Parker, Jackson vs Gawn (results in 4c).

**Draftee check against 2026:** the 2026 draftees above averaged 65.9 to 71.9 (Sharp pick 13, Farrow pick 10, Lindsay pick 19). Rookie projection (FR26) is tested against these.

## 4e. Injury list handling (decided 2026-10-10)

| AFL injury list status | Projection | Squad list |
|---|---|---|
| **Indefinite** | **0 for all future rounds** until the status changes | Flagged |
| **Test** | **Unchanged** | **Highlighted in the user's squad list** so the user is aware |
| A stated return time (weeks) | 0 until the expected return round [assumed] | Flagged |
| Season | 0 for the rest of the season [assumed] | Flagged |

The developer loads the list after each round (source and terms of use to confirm, Q17).

## 4f. What the data says about age (checked 2026-10-10, local match data, home-and-away, players with 10+ games in both seasons)

- **Age in the training data is the age on that season's roster, not today's age.** `build_player_match_dataset.py` states that Age is "a once-a-season roster snapshot": footywire has no per-match age. A 2022 row carries the 2022 roster age (Sam Walsh: 21 in 2022). For next season, the stub row advances age by the number of years since the player's last season. Open point: the exact roster pull date (age near a birthday could be off by one year) [verify].
- **Mean change in AF average to the next season, by age in the current season:** +5.4 at 18, +3.7 at 21, +3.0 at 22, +0.6 at 23, +1.5 at 24, **-0.8 at 25, 0.0 at 26**, -1.8 at 27, -2.1 at 28, -2.9 at 29, about -2 to -5 from 30. So the data shows improvement to about 24, a **plateau around 24 to 26, and decline from 27**; it does not show a flat peak until 29. The engineer-features docstring says "peak at 29", which looks like a different measure (the age at which the average is highest, not the change). Survivors only (10+ games both years), so declines are probably understated.
- **Sam Walsh (AF average, games):** 103.3 (20) in 2022, 94.6 (15) in 2023, 106.1 (19) in 2024, 93.1 (14) in 2025, 98.4 (23) in 2026. Matches the owner's figures; a good test of injury-affected seasons.
- **Lachie Neale:** 107.0, 97.0, 101.0, 94.6, then **102.2 at age 32** (+7.6): the kind of exception the owner expects the model to mis-handle.


## 4a. Season lifecycle (new, 2026-10-10)

| Stage | What the user can do | Data that changes it |
|---|---|---|
| Pre-season (Open) | Build squad, save versions, compare, edit locks and avoid list | Rumours, injuries, role changes (notes), updated projections |
| After Round 0 | Re-build using Round 0 actual scores | Round 0 scores loaded by the developer |
| Lineup announced | Re-build; unnamed players flagged | Lineup file loaded by the developer |
| Round 1 starts (InSeason) | **Squad locks.** Build is read-only. Only trades. | Frozen baseline saved |
| Each week | Trade advice; user decides | Injury updates, form, position changes, news, lineups |

Reused idea from the local app: flipping the season to InSeason freezes the saved squad into a baseline and makes the build lists read-only.

## 5. Data

| Item | Source | Note |
|---|---|---|
| Player stats up to 2025 | Local app, already in S3 (also 2026) | Verified, see 5a |
| 2026 player list | Local app, in S3 (`player_master_2026`) | Verified. Test world uses the real 2026 list |
| Round predictions, season projections, shortlists | Local app outputs | **New:** `loader/` copies them to S3 |
| Fixtures, lineups, prices, injuries | Local Round Ops ("Get Lineup" etc.) | Run by the developer, then uploaded [assumed: scrapers stay local] |
| Expert knowledge | Developer's Gmail, specific senders | Run locally by the developer; summaries uploaded [assumed]. Personal use; never shown to other users; summaries and links, not full text |
| Per-user state | New | DynamoDB [assumed] |

### 5a. What is actually in AWS (verified 2026-10-10 from S3, Glue and Lambda listings)

One S3 bucket, 567 objects, 7.6 MiB in total (546 of the objects are Athena query results: 270 `.metadata` files plus their result CSVs; 92 are under `Unsaved/`; 124.7 KiB in total).

| Prefix | Contents |
|---|---|
| `raw/player-match-stats/year=2022..2026/` | One CSV per year (about 1.2-1.3 MiB each) |
| `raw/player-season/year=2026/` | Player master list 2026 (CSV) |
| `processed/player-match-stats/year=2022..2026/` | One Parquet per year (about 160 KiB each; 29 columns incl. `PlayerId`) |
| `processed/player-master/year=2026/` | Player master 2026 (Parquet; 14 columns: id, player, team, price estimates, games, age, origin, position, eligibility flags, height) |
| `processed/player-identity/` | Identity table (Parquet; `playerid`, `player`, `footywireplayerid`, `idsource`), in three copies (`player_identity.parquet`, `all/`, `_internal`) |
| `knowledge-base/competition-rules/` | `COMPETITION_RULES.md` (12 KiB): the source for an existing Bedrock knowledge base |
| `athena-results/` | Query output, no lifecycle rule (546 objects, 124.7 KiB, oldest 2026-09-28) |

**Glue:** database `afl_analytics` (your saved Athena queries use only `player_match_stats_partitioned`, `glue_player_master` and `glue_all`), **3 crawlers** (identity, master, match stats; no schedule), **0 Glue jobs**. The "pipeline" is the Python code in `src/afl_analytics/pipelines`, run by hand, which uploads to S3; the crawlers only catalogue it. There are 7 tables, several overlapping: `glue_all` and `glue_player_identity_parquet` (identity, same 4 columns); `glue_player_master` (partitioned by year); `glue_player_match_stats` (partitioned, 29 columns, has `playerid`, no `footywireplayerid`); `player_match_stats` (raw, 2026 only); `player_match_stats_parquet` (2026 only); `player_match_stats_partitioned` (30 columns, has both ids).

**Lambda:** only the two proof-of-concept functions (`afl-get-player-performance`, `afl-whoami`; python3.14, 128 MB).

**Not in AWS yet (the gaps this app needs):** fixtures, lineups, prices over time, projections and predictions (round and season), shortlists, Round 0 scores, any per-user data (squads, versions, preferences, leagues), and the expert-email knowledge. The data volume is small (a season of match stats is about 160 KiB as Parquet), so S3 and Athena cost will be negligible; the cost drivers will be Bedrock, AgentCore and Lambda.

**Test-world consequence (decided 2026-10-10):** the 2026 stats stay visible. The app is simply pre-season 2027: it uses all real data it has (2022 to 2026) and **test data for 2027**, clearly flagged (section 5b).

### 5b. 2027 test data (decided 2026-10-10)

The app runs as if preparing for the 2027 season. Real 2027 data does not exist yet, so a **separate code module** (`AFLFantasyApp/testdata/`, never part of the app itself) builds test data for 2027. When actual data arrives it **replaces** the test data.

| Item | How the test data is built |
|---|---|
| Player list | Every player in the 2026 list continues in 2027 (retirements and delistings ignored). Plus the 2027 draft prospects from the local `draft_prospects_2027.csv` (70 prospects, none drafted yet, 9 with a club link). |
| Team changes | A small hand-kept file (`assumptions_2027.csv`): <player A> to Western Bulldogs, <player B> to Hawthorn, <player C> to Richmond (stated by the owner; more can be added). |
| Value changes | Same file: a stated adjustment to a player's projected average (for example a role or club change). Magnitude to be agreed. |
| Projected averages | Reuse the local app's 2027 projection (`shortlist_2027_AF/SC.csv`, `season_projection.py`), with overrides from the assumptions file. |
| Initial prices | Owner-supplied pricing rules (from public pages, 2026 season values; kept in config because they change by season): **AFL Fantasy** price = previous-year average (less any discount) x **magic number 10,490**, rounded to the nearest $1,000; cap **$18,300,000**. **SuperCoach** magic number **5,410** (fixed to about round 3); cap $10,000,000 (from `COMPETITION_RULES.md`). The "priced-at" average is price / magic number. Test prices use 2026 averages as the previous-year average. |
| Draft prospects | Source: the **raw, read-only** dossier `draft_dossier_2026.csv` (70 rows; `Drafted_Club` and `Draft_Pick` are empty). **Only the first 50 are used.** They have no Footywire ID, so each gets a **test ID** and is marked `data_source = test`. |
| Draftee prices | **By pick number:** the 2027 price of the player at pick N is the 2026 *starting* price of the player drafted at pick N in last season's draft (the owner supplied that draft table from Footywire). Checked 2026-10-10: all of the first 50 picks are in the 2026 player master with an AF start-price estimate (after removing the trailing " A" / " F" marker, which I believe flags academy and father-son picks: **[verify]**). The SuperCoach estimates were not checked. These estimates are derived (current price less total change), not official prices. |
| Draftee clubs | **Owner-supplied first-round draft order (20 picks, 2026 draft; includes traded picks), kept in a local input file.** Dossier rank N takes pick N's club. Players with a known club tie override this: **Cochrane** (rank 1, Port Adelaide academy and father-son, price of pick 1) and **Walker** (rank 2, Carlton father-son, price of pick 2; confirmed by the owner 2026-10-10). Their clubs match the bid, so the bidding clubs' picks 1 and 2 (Essendon, Richmond) produce no test draftee. Ranks 21 to 50 have no pick-to-club list yet: **fallback** is the 9 club-tie players to their clubs, others in reverse 2026 ladder order (computed from `fixture_2026.csv`, home-and-away, 23 games each), flagged synthetic. All club assignments are flagged synthetic. |
| Fixture | **2027 has a Round 0, but its teams are unknown and no 2027 fixture exists yet.** For testing, the **2026 fixture stands in for 2027** (in 2026 the Round 0 teams were Sydney, Carlton, Gold Coast, Geelong, GWS, Hawthorn, Brisbane, Western Bulldogs, St Kilda and Collingwood). It is tagged `data_source = test` like every other test row and is replaced by the real 2027 fixture when published; `cleanup_testdata` covers it. |
| Lineups, injuries, news, Round 0 scores | Synthetic, built later as each capability needs them |

**Isolation (so the real player data is not polluted):**
1. **Separate location.** Test data lives only under a new S3 prefix (for example `app-data/test/`), outside every existing crawler target and every existing Glue table. Nothing is written to `processed/` or `raw/`.
2. **Overlay, not edit.** Existing 2026 players keep their real Footywire ID. Test data is an overlay keyed by that ID (team, position, projected average, price for 2027); the real master and identity tables are never changed.
3. **Own ID range.** Players who do not exist yet (draftees) get test IDs from a reserved range (`900000001` upward). Real IDs are 1 to 318 (internal) and 1,611 to 7,481 (Footywire), so no collision is possible.
4. **Crosswalk on replacement.** When a draftee gets a real Footywire ID, a `test_id -> real_id` crosswalk lets saved squads and versions migrate. Actual data then wins.
5. **Proof of isolation.** A before-and-after check lists the bucket and the Glue tables to show only the new prefix changed.

**Test scenario: draftees the owner will consider (stated 2026-10-10):** dossier ranks 1, 2, 3, 4, 6 and 7: Cochrane (Port Adelaide), Walker (Carlton), Drever, Butler, Mellody and Teixeira (club not yet known). Rank 5 (Harry Van Hattum, a ruck) is left out by the owner: key-position players are normally excluded because they are important in real football but weak for fantasy (the dossier's own note for him: young rucks rarely get senior ruck minutes early). **Cochrane and Walker are locks** (always in the squad, as with premium locks in the local app). Whose locks these are (user A, user B or both) is open.

**Evidence on draftee value (observed 2026, from the owner's pick table and the local 2026 match data; one draft class, so only indicative):**

| Pick (2025 draft) | Games | AF avg | Start price (AF, estimate) | Note |
|---|---|---|---|---|
| 1 | 22 | 72.1 | $350k | Midfielder |
| 3 (Dean) | 22 | 49.9 | $342k | Defender; played every week but scored little, so he was not a fantasy option (the owner's point) |
| 4 (Duff-Tytler) | 16 | 44.2 | $338k | Forward/ruck, 4.6 hitouts a game: mostly not a ruck yet |
| 10 | 21 | 71.9 | $314k | |
| 13 | 15 | 70.9 | $302k | |

- The start price falls in a near-straight line with pick number (about $4k per pick, $350k at pick 1 to $274k at pick 20), but the **return does not**. Picks 10 and 13 out-scored picks 3 and 4. Among the first 20 picks with 8 or more games (14 players), the AF average ranged 33.6 to 72.1, median 58.1.
- Van Hattum's profile (204 cm, 7 Talent League games, 20.4 hitouts, "ruck who rotates forward") resembles Duff-Tytler's first year. This supports the owner's view that rucks take time, but it is one comparison, not proof.
- Consequence: **price by pick is reasonable for test prices, but pick number is a weak predictor of fantasy return.** The app should not rank draftees by pick alone; the owner's judgement (role, position, job security) matters.

**`cleanup_testdata` (required function).** Removes or replaces test data when actual data arrives.
- **Dry run by default:** lists exactly what it would remove or change, with counts; nothing happens without an explicit confirm flag.
- **Modes:** `--all` removes every test row and object; `--replace-with-actual` matches each test draftee to its real Footywire record (by name and club), writes the `test_id -> real_id` crosswalk, moves any saved squad or version to the real ID, then removes the test row. Draftees with no actual match yet are listed and kept.
- **Safety:** it deletes only under the test prefix and only rows tagged `data_source = test`; it refuses to touch anything else; it reports every saved squad version that references a test ID before changing it; it prints what it did.
- **Two stages in practice:** after the AFL draft (club and pick known) update the club and price; when the player list is published (Footywire ID known) replace the test ID.
- **Test:** an automated check that `cleanup_testdata` leaves the real tables unchanged.

**Input file note:** the Footywire draft table the owner supplied is third-party content. It is kept locally for the generator and is **not committed to git**.

**Rules:** every test row carries `data_source = test`; real rows carry `data_source = actual`. Where both exist, `actual` wins and `test` is ignored, never merged. The app shows a "test data" notice whenever an answer used any test row. Results must be based on actual data wherever it exists.

**Test world:** statistics up to 2025, the real 2026 player list, no price movement after round 1. Anything later (injuries, news, lineups, price moves) is **synthetic**. The real 2026 results can be a held-out check of projections later, not app data.

Player identity is always `(name, team)`, never name alone (two different "Max King"s exist).

## 6. Security (what the POC proved, applied here)

- Cognito token → Gateway (JWT) → Cedar → Lambda.
- Every per-user tool takes `user_id` **bound from the token**, not from the model. A prompt cannot change it.
- Cedar protects "A cannot read B's squad or preferences". Preferences (age cap etc.) are **data**, not access control.
- Default deny. Check the response body for denials (HTTP 200 with an error).
- No tokens or secrets in git, chat or posts.

## 7. Architecture decision (recommendation)

**AgentCore Runtime (Strands) + Gateway (JWT) + Cedar + Lambda tools.** Reason: almost every tool is per user; a harness cannot forward the user's token, and with inline tools every tool would run in the client app, which removes the harness's saving.

A **harness** is a reasonable fit for the knowledge side (single user, the developer): a quick agent that searches summaries. The Gmail scan itself is a batch pipeline, not an agent. [assumed]

The draft ranking and the squad/trade solver are **deterministic functions in Lambda**; the model explains their output.

## 8. Reuse from the local app

| Need | Local module | Verdict |
|---|---|---|
| Projections (core) | `models/predict_upcoming_round.py`, `season_projection.py`, `component_model.py`, `gradient_boosting_model.py`, `shortlist.py` | **Port to AWS as the core job** (changed 2026-10-10, was "run locally and upload"). [verify] libraries and run time on AWS |
| Round Ops, scrapers | `round_ops.py`, `ingestion/*` | Run locally; upload results |
| Squad solver, trade planner | `team_builder.py`, `trade_planner.py`, `current_squad.py` | Adapt to per-user; **[verify]** the solver library packages into Lambda and breaks ties the same way each run |
| Age and avoid rules | `guardrails.py`, `guardrails.yaml` | Adapt: per user, not global |
| Rules and caps | `COMPETITION_RULES.md`, squad configs | Config |
| Agent tools | `mcp_server/server.py` | Pattern only (no login there) |
| Draft planner, per-user state, expert search | none | New |

Modules reused are **copied** into the AWS app with a comment naming the source file, not imported across repos. Cost: a fix has to be applied in both places.

## 8a. Architecture diagram

Sponsor-level diagram (data layer checked against the AWS listings on 2026-10-10): [diagrams/architecture_app_v0.1.pdf](diagrams/architecture_app_v0.1.pdf) (source `architecture_app_v0.1.html`). 

## 9. Source layout and ports

`C:\005awsprojects\afl-analytics\AFLFantasyApp\`: `agentcore/` (CLI project), `app/` (agent), `lambda/` (one folder per tool), `webapp/` (React, port **5174**), `loader/` (local → S3), `tests/`. Requirements and notes stay in `07_AWS`.

The Cognito app client for the web page needs `http://localhost:5174/` as an allowed callback URL.

## 10. Test plan (outline)

- Per-user isolation: A and B each read only their own squad and preferences; a forged `user_id` has no effect.
- Preferences: A (cap 32) and B (cap 33) get different candidate lists from the same question.
- Determinism: the same inputs give the same numbers on two runs.
- Attack prompts against the agent.
- Draft plans: different league sizes give different rankings.
- Counts and capability checks, not wall-clock times.

## 11. Cost plan

After the build, measure what it costs to run: billing by **service and usage type**, plus counts (model calls, tokens, tool calls, Lambda invocations, DynamoDB requests, S3 requests). A budget alarm is set before the first user test. Idle cost is checked separately from per-use cost.

## 12. Out of scope (for now)

Keeper and Dynasty leagues; DFS; logging into AFL Fantasy or SuperCoach; executing trades; the local app's rebuild on Claude SDK or LangChain.

## 13. Decisions and open questions (consolidated 2026-10-10)

### 13a. Decided

| # | Decision |
|---|---|
| D1 | Redraft leagues only; two products (team builder + weekly trades, draft planner). Before March 2027: sign-in, preferences, Build Squad with versions and lock, draft planner, expert-insight answers, test data. After the season starts: trade planner. |
| D2 | The 2026 stats stay visible. Real data is used wherever it exists; test data is flagged and replaced by actual data. |
| D3 | Test data comes from separate code, as an overlay on real IDs. Draftees get test IDs from 900000001 upward, tagged `data_source = test`. `cleanup_testdata` removes or replaces them. |
| D4 | Prices: owner-supplied magic numbers (AF 10,490, SC 5,410); draftee price by pick number from last season's draft; first 50 picks. |
| D5 | Draftees: dossier ranks 1 and 2 are Cochrane (pick 1, Port Adelaide) and Walker (pick 2, Carlton); both are locks. Clubs from the supplied 20-pick order. |
| D6 | Fixed facts that change data (team moves) go in the assumptions overlay; **opinions go in the knowledge base, not code** (see 13c). |
| D7 | Key-position exclusions are the user's own avoid-list entries with a reason tag; the app does not infer them. |
| D8 | The projected score is final: **model output plus special rules from config** (fixed value, factor or method), applied by **Project**, which rewrites the single season file in place. Users cannot edit it. Draft, then Frozen (FR25). Supersedes the earlier hand-edit design (FR15, FR16). |
| D9 | Test team moves for 2027 (overlay): Butters to Western Bulldogs, Merrett to Hawthorn, O'Driscoll to Richmond, Rowan Marshall to Geelong, Sam Marshall to St Kilda, **Ben King to Hawthorn**. Ben King is a key-position player and is not considered, so he goes on the avoid list with reason `Key Position` (both test users). |
| D11 | 2027 has a Round 0. Test runs use the 2026 fixture as a stand-in (flagged test). The Round 0 team exclusion is config-driven. |
| D10 | Sam Marshall: 2 games in 2026 (AF 34 and 78, average 56.0), a **21% price discount**, projected at **60 to 70** (manual override). O'Driscoll: 17 games, AF average 64.1; the owner may drop his 5 lowest scores and recalculate (that gives 71.2 over 12 games). |
| D12 | Users A and B choose only locks, shortlist additions and exclusions. |
| D13 | In season, projections are refreshed after each round with the AFL injury list (injured = 0 until expected return). New data source and process: FR27. |
| D14 | New players are projected from the draft dossier and previous draftees' results (FR26). |
| D15 | Band number = lowest edge of the band; one ladder for both games; each player has his own band per game (owner, 2026-10-10). |
| D16 | Type stays rule-derived and unchanged; new **Tags** field is information only (D15 and FR33). |
| D17 | Role, Band and Tags are stored per competition in `02_Data/masterData/shortlist_attributes_{season}_{AF|SC}.csv` (config `file_pattern`), separate from the include and exclude exceptions file. |

### 13b. Open: needs your answer

| # | Question | My default if you don't answer |
|---|---|---|
| Q1 | Is the split in D6 right? A team move must exist in the data (the squad builder checks clubs and fixtures), while commentary such as "value player" or "viable starter" is knowledge-base text that never changes a number. | Yes |
| Q2 | ~~Other team moves~~ **Answered:** Ben King to Hawthorn (key position, not considered). Any more? | None |
| Q3 | ~~O'Driscoll's value~~ **Answered:** special flag with a manual override or a method (D8). | |
| Q4 | Locks for Cochrane and Walker: both users, or only user A? | Both users |
| Q5 | Test projected average for draftees: (a) same-pick player's 2026 return, (b) median by pick band, (c) dossier outlook only. | (b) |
| Q6 | Clubs for draftee ranks 21 to 50: do you have picks 21 onward, or use the reverse-ladder fallback? | Fallback |
| Q7 | Does SuperCoach Draft differ from AFL Fantasy Draft in squad structure or scoring? | League settings entered per league |
| Q8 | Discounts: Sam Marshall has 21%. Is there a general rule (for example by games played in the previous season), or is each discount set by hand per player? | By hand, per special player (FR17) |
| Q9 | Where do owner notes and expert-email summaries live: in the existing Bedrock knowledge base's S3 folder (a new `owner-notes/` and `expert-emails/` folder beside `competition-rules/`), or a separate knowledge base? | Same knowledge base, new folders |
| Q11 | ~~Restrict candidates to the shortlist?~~ **Decided:** yes, Build Squad uses the shortlist plus locks; the shortlist is a prerequisite (FR21). | |
| Q12 | ~~Lock score~~ **Decided:** the projected score is common to all users; locks are per user and take the value from the season projection (section 4b). | |
| Q13 | ~~Round 0 in 2027?~~ **Answered:** yes, teams unknown until the fixture is published. The exclusion of Round 0 teams from the starting squad is a config flag; the team list comes from the fixture's Round 0 matches (test: the 2026 fixture). Until a real fixture exists the app shows a warning that Round 0 teams are unknown. | |
| Q14 | ~~80% rule~~ **Answered:** it was an example. The requirement is a **generic rule schema in config** (FR15). Which rule types do you need first beyond fixed, factor and drop-lowest-N? | The set in FR15 |
| Q15 | ~~Relative or absolute test~~ **Answered:** three-level evaluation set (4d): aggregate hold-out is the pass bar; named cases are reported; pairs check relative order. Tolerance for the aggregate bar: how much worse than MAE 7.95 (AF) is acceptable? | 0.3 |
| Q16 | Users "select the players to lock, shortlist or exclude": is the shortlist **shared and rule-built** with each user able to add or remove players, or **entirely chosen by each user**? | Shared rule-built list plus per-user additions and exclusions |
| Q17 | **AFL injury list** source and terms: is the official AFL injury page the source, and may it be loaded by the developer after each round? (Status handling is decided: 4e.) | Official page, developer loads |
| Q18 | Freeze point: when do you consider the pre-season projection final (before the January player list, after it, after Round 0)? | After the January player list, before building the starting squads; refreshed after Round 0 |
| Q10 | How the app learns each user's squad (see the clarification below). Are versions unlimited, each a full snapshot with its locks and avoid list? | Built in the app; unlimited full snapshots |

### 13b-2. How the local Build Squad works (read 2026-10-10, `team_builder.py`)

1. Place the premium locks (each lock carries its own hand-set projected score).
2. Remove Round 0 teams' players, and non-locked players below the minimum-average floor (60 in 2026 config).
3. Apply the guardrails (avoid list, age cap) to the remaining pool only.
4. Maximise total projected score under the remaining budget and position slots (players' DPP/TPP eligibility treated as a set); the club minimum is checked after the solve.

**It does not use the 2027 shortlist.** The shortlist (`shortlist_2027.yaml` rules plus an exceptions file: proven 70+ scorers likely to improve, under 32, plus "injured guns") is a separate pre-season screening feature with its own pages; it also feeds the round-prediction scripts.

### 13b-1. How squads get in (clarification)

Yes: the squad means each user's squad for AFL Fantasy and for SuperCoach. Three situations:
1. **Pre-season:** the user presses Build Squad; the solver creates a squad inside the app. Nothing needs entering. Each save is a version.
2. **At the lock (round 1):** the user enters the squad into the real game by hand (the app never logs in to the games). If it differs from the app's version, the user edits it in the app so the baseline matches what they actually have.
3. **In season:** the app only advises. After the user makes a trade in the game, the user records it in the app (a "record trade" action), so the squad stays current. CSV upload would only be a convenience for loading a squad from elsewhere (for example the local app's squad file). [assumed: build and edit in the app, record trades; CSV later or never]

### 13b-3. New technical checks created by the core-projection decision [verify]

- Can the base projection (trained model plus season projection) run on AWS: libraries, size, run time? Which compute?
- How does the "Project" button trigger it and report progress (an asynchronous job)?
- Where do the season projection file, audit log, shortlists and config files live (S3 files plus an index in DynamoDB)? S3 versioning is **not enabled** on the bucket (verified 2026-10-10: `get-bucket-versioning` returned nothing, which means it has never been switched on), so overwriting the projection file keeps no old copies. Do not enable bucket versioning without excluding this prefix.

### 13c. Knowledge base (not code)

Content types: competition rules (exists), owner notes, expert-email summaries. Every entry carries **source, date and who wrote it**, and the agent shows it beside the numbers (FR7). Owner notes stated so far:
- Sam Marshall plays for St Kilda in 2027 and is a value player in Fantasy, at least to start.
- Rowan Marshall plays for Geelong in 2027, which makes him a viable starting ruck.
- Luke Jackson is the best young ruckman.
- Key-position players are important in real football but weak for fantasy (a general principle, not a player).

### 13d. Assumed (please confirm in bulk, or tell me which to change)

Scrapers and the Gmail intake stay on your machine, and results are uploaded; Round 0 scores are loaded by the developer; the three Glue tables your queries use are canonical (`player_match_stats_partitioned`, `glue_player_master`, `glue_all`); Athena results expire after 30 days once you have checked that result reuse is off.

### 13e. Technical checks (mine, not yet done)

- Does the squad solver package into Lambda, and does it break ties the same way on each run?
- Can Lambda read the small Parquet files directly (instead of Athena)?
- Is the AF start-price estimate good enough, and what are the SuperCoach ones (not yet checked)?
