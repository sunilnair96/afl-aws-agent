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
| FR3 | Preferences per user: age cap (A: 32, B: 33), avoid list, premium locks. Applied by the **tool code**, never left to the model. |
| FR4 | Projections are **deterministic**: the app reads stored per-round predictions; it never re-runs a model. |
| FR5 | Trade advice: return a small set of options with the numbers behind them; the user decides. |
| FR6 | The user can state "player X is injured / in doubt". [assumed: applies only to that user, for the current round, saved with the date] |
| FR7 | Expert views (from the email knowledge base) are shown **with source and date**, beside the numbers, and never change them. Example: "<source> suggests <player> is likely to be tagged this week." |
| FR8 | Draft plan: rank by position scaled to league size (value over replacement) [assumed], pick plan for the user's slot, a simple "what if I pick a ruck first" comparison [assumed]. |
| FR9 | Rules (caps, squad shape, trade limits) come from config, never hard-coded; they change by season. |
| FR10 | Every tool call is logged (tool, arguments, allowed or denied) for debugging and cost counts. |
| FR11 | **Build Squad:** run the optimiser against the user's locks and avoid list. Two steps, as in the local app: a fast, rerunnable *build* that never saves, then an explicit *save*. Never auto-save. |
| FR12 | **Squad versions:** a user keeps several named, dated versions of a squad (pre-season changes as rumours, injuries and role changes appear), can compare two versions, and marks one as the active version. [assumed: unlimited versions, per competition] |
| FR13 | **Season lifecycle** (mirrors the local app's season status: Planned, Open, InSeason, Closed). Pre-season: build and versions. After Round 0, actual Round 0 scores can be loaded and a new build uses them. When the round 1 lineup is announced, named and unnamed players are flagged. **At the start of round 1 the active version is frozen as the baseline and Build mode becomes read-only.** |
| FR14 | **In season:** only Manage Trades is available (propose-only, within the round's trade limit), starting from the frozen baseline and each round's current squad. |

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

One S3 bucket, 567 objects, 7.6 MiB in total (454 of the objects are Athena query results).

| Prefix | Contents |
|---|---|
| `raw/player-match-stats/year=2022..2026/` | One CSV per year (about 1.2-1.3 MiB each) |
| `raw/player-season/year=2026/` | Player master list 2026 (CSV) |
| `processed/player-match-stats/year=2022..2026/` | One Parquet per year (about 160 KiB each; 29 columns incl. `PlayerId`) |
| `processed/player-master/year=2026/` | Player master 2026 (Parquet; 14 columns: id, player, team, price estimates, games, age, origin, position, eligibility flags, height) |
| `processed/player-identity/` | Identity table (Parquet; `playerid`, `player`, `footywireplayerid`, `idsource`), in three copies (`player_identity.parquet`, `all/`, `_internal`) |
| `knowledge-base/competition-rules/` | `COMPETITION_RULES.md` (12 KiB): the source for an existing Bedrock knowledge base |
| `athena-results/` | Query output, no lifecycle rule (454 objects, 92 under `Unsaved/`) |

**Glue:** database `afl_analytics`, **3 crawlers** (identity, master, match stats; no schedule), **0 Glue jobs**. The "pipeline" is the Python code in `src/afl_analytics/pipelines`, run by hand, which uploads to S3; the crawlers only catalogue it. There are 6 tables, several overlapping: `glue_all` and `glue_player_identity_parquet` (identity, same 4 columns); `glue_player_master` (partitioned by year); `glue_player_match_stats` (partitioned, 29 columns, has `playerid`, no `footywireplayerid`); `player_match_stats` (raw, 2026 only); `player_match_stats_parquet` (2026 only); `player_match_stats_partitioned` (30 columns, has both ids).

**Lambda:** only the two proof-of-concept functions (`afl-get-player-performance`, `afl-whoami`; python3.14, 128 MB).

**Not in AWS yet (the gaps this app needs):** fixtures, lineups, prices over time, projections and predictions (round and season), shortlists, Round 0 scores, any per-user data (squads, versions, preferences, leagues), and the expert-email knowledge. The data volume is small (a season of match stats is about 160 KiB as Parquet), so S3 and Athena cost will be negligible; the cost drivers will be Bedrock, AgentCore and Lambda.

**Test-world consequence:** `year=2026` match stats are already in S3 (complete season data). To test as if only 2025 and earlier exist, tools must filter to `year <= 2025` (or the 2026 files must be moved aside). Decision needed (open question 9).

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
| Projections | `models/predict_upcoming_round.py`, `season_projection.py`, `component_model.py`, `gradient_boosting_model.py` | Run locally; upload results |
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

## 13. Open questions

1. ~~What is already in S3 and Glue?~~ **Answered 2026-10-10, see 5a.**
2. Do SuperCoach Draft rules differ in structure from AFL Fantasy Draft? [assumed: entered per league]
3. How do squads get in: CSV upload, built in the app, or both? [assumed: CSV first]
4. Does the solver package into Lambda (library size, tie-breaking)? **[verify]**
5. Do scrapers stay local (footywire access from AWS addresses is untested)? [assumed: local]
6. Expert-content handling and the Gmail login: local intake, summaries uploaded. [assumed]
7. Squad versions: unlimited, or capped? Does a version also keep its locks and avoid list at that moment? [assumed: yes, a version is a full snapshot]
8. How does the actual Round 0 score reach the app: loaded by the developer from the local app after the round? [assumed: yes]
9. Test world: S3 already holds `year=2026` match stats. Hide them by filtering to `year <= 2025` in the tools, or move them aside? [assumed: filter, so nothing is deleted]
10. Six overlapping Glue tables: which are canonical for the new app? [assumed: `glue_player_match_stats`-style partitioned table with `playerid`, plus `glue_player_master`; the rest left alone]
11. Athena results have no lifecycle rule (454 objects). Add an expiry rule? [assumed: yes, small and cheap]
