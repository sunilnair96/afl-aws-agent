# RESUME HERE — AFL Fantasy AWS app: status and next steps

*Written 2026-10-11 so the project can restart cleanly in about a month (November 2026). Read this first; the other documents are linked at the end. No secrets or account identifiers are in this file (placeholders only).*

## 1. The project in five lines

- **Goal:** learn AWS properly (Bedrock AgentCore, Lambda, Cognito, Cedar, now SageMaker) by building an AFL Fantasy / SuperCoach assistant on a credit-based account (about $158 of $160 left on 2026-10-10; credits last about 6 months from late September).
- **Two apps:** the **local app** (`06AFLAgent/app`, React at `localhost:5173`, backend at `127.0.0.1:8000`) is the *final* app for the 2027 season and uses the Anthropic key. The **AWS app** is a new, separate app for learning and cost measurement; it reuses data and code from the local one.
- **Users:** User A and User B use the tool; the developer runs it. Each user's squads, preferences, locks and shortlist are private to them.
- **Two products:** (1) team builder with squad versions, locked at Round 1, then weekly trade advice; (2) draft planner (Redraft only).
- **Order of work:** make the **projection model and its config final and well evaluated first** (locally), then try SageMaker, then build the AWS app around it.

## 2. Key dates

| Date | What |
|---|---|
| **30 Nov 2026** | AFL player list finalised (AFL Fantasy and SuperCoach then take time to publish theirs) |
| **5 Jan 2027** | Real pre-season news begins |
| Early Jan 2027 | Fantasy and SuperCoach player lists and prices released |
| **Last week of Feb 2027** | The one practice match: **projected scores must be final by then** (working date 20 Feb); afterwards only injuries change them |
| March 2027 | Season starts, with a **Round 0** (teams unknown until the fixture is published) |

## 3. Where everything is (and what is pushed)

| Place | Branch | Last commit | State |
|---|---|---|---|
| `06AFLAgent/app` (repo `afl-agent-app`) | **`preseason-phase1`** (NOT merged to main) | `407d84c` shortlist screen | Pushed. 499 backend tests passed at the last full run. Untracked and **not mine**: `src/common/ingestion/build_player_master_list.py` (it merges the Fantasy and SuperCoach rosters into the player master; see the SageMaker notes) |
| `06AFLAgent/00_Planning` (repo `afl-agent-planning`) | `master` | `785ce2a` | Pushed. Untracked and not mine: `SUPERCOACH_2026_REPORT_CARD_NOTES.md` |
| `06AFLAgent/07_AWS` (repo `afl-aws-agent`, private; created clean, no old history) | `main` | see `git log` | These notes and plans |
| `C:\005awsprojects\afl-analytics` (private GitHub repo) | `main` | `4e62745` | Pushed: `AFLStartingTeamAgent/` (Strands agent + gateway + Cedar), `AFLHarness/` (harness experiments, client app). **Unrelated uncommitted bedrock work** in `src/afl_analytics/bedrock/` is not part of this project's commits |
| Old 07_AWS git history | — | — | Kept locally, **not for sharing**, in `06AFLAgent/07_AWS_git_backup_old_history` (it contains real identifiers) |
| Your own notes moved out of 07_AWS | — | — | `06AFLAgent/07_AWS_Untracked` |

**Not in git (copy these somewhere safe now and then):**
- `02_Data/masterData/shortlist_attributes_2027_AF.csv` (**9 rows** on 2026-10-11) and `..._SC.csv` (**11 rows**): your own 2027 shortlist (Role, Band, Tags). An earlier copy is in `02_Data/masterData/_backup_20261010/`.
- Everything in `02_Data/` (match data, price rosters, season tables, the evaluation exports `projection_eval_players.csv` and `rank_shift_chart.html`).

## 4. What was built (local app, branch `preseason-phase1`)

1. **Evaluation harness** (`src/common/models/season_projection_eval.py`, `config/eval_projection.yaml`): trains the calibration on the earlier transitions and tests on 2025 → actual 2026, with breakdowns (group, age band, position, projected level), named cases, trajectory pairs, and a per-player AF/SC export. Reproduces the original validation exactly (AF MAE **7.95**, SC **8.51**). Run: `uv run python -m src.common.models.season_projection_eval` (prints; writes only with `--log` or `--export-players`).
2. **Rank-shift chart** (`rank_shift_chart.py`) → `02_Data/processed/rank_shift_chart.html`.
3. **Shortlist attributes** (`config/shortlist_attributes.yaml`, `src/common/shortlist_attributes.py`, `player_evidence.py`, API `/shortlist-attributes/*`, screen `ShortlistPage.tsx`):
   - **One band ladder** for both games: 30-40 … 90-100, then steps of 5: 100-105 … 120-125, 125+. A band's number is its **lowest edge**.
   - **Role** (DEF/MID/RUC/FWD), **Band** (overrides the model's projected score), **Tags** (information only; no logic may read them; guarded by a test). Per competition.
   - **Own-list mode** (`list_mode: manual`): your shortlist is exactly the players with an attribute row; the rules' output is a pool of **candidates**; Remove, Add to my list, bulk **Select** (no limit) and **Cmp** (max 4); scrolling table with pinned columns; evidence columns (games at or above two score thresholds over 3 seasons: AF 100/120, SC 110/130 assumed); draft profile for players with no AFL games.
   - `Player_Type` (Emerging / Established / Returning / Other) is rule-derived and **unchanged** (the shortlist rules use it).
4. **Not built yet:** the 2027 test-data generator and `cleanup_testdata`; the special-rules engine; the new-player (draftee) model; opponent look-ahead; anything on AWS for the new app.

## 5. What was learned (use these before changing the model)

- For **established** players the model beats "last season's average" by only about 0.6 points (AF MAE 7.97 vs 8.61); nearly all its gain is on low-data players (7.89 vs 12.23).
- It **under-projects young and prime players** (bias −4.9 at 21 or under, −2.6 at 25-26) and compresses the top (no mid projected at 115+, no ruck at 110+).
- **Rucks dropped about 10 points in 2026** (a rule change, per the owner), so 2025→2026 is a structural break. Left in training it would teach the 2027 model that rucks decline; options are in `09_PHASE1_PLAN.md` 3a.
- What goes with improving: **youth**, being **below your own best**, and not having jumped last year; high averages regress; role-trend features add nothing; game-score consistency does not predict next season's average. Premiums returning from injury came back **below** last year's average on average (n=14).
- **Price vs return:** priced at 80+ nobody beat their price by 15 in 2026; 16% of all players did (32% of players 21 or under). Draftees with 8+ games all beat their price (11 of 14 by 15+); pick number barely matters, **getting games** does.
- **Gotchas:** the data uses "Zachary Merrett", not "Zach Merrett" (names must match or be aliased); always match players on (name, team) with `normalize_team()`; position never changes between seasons in the data (can't test role change); the dossier prospect "Tom Brown" collides with a real player.

## 6. Decisions to keep (full list: `08_APP_REQUIREMENTS.md` 13a)

Redraft only; AWS app = Runtime + Gateway + Cedar (a harness cannot forward the user's token); one projection file per season, no versions, frozen by the practice match, injuries only after that; special rules live in config; users only choose locks, shortlist and exclusions; the shortlist is the owner's own list; tags never drive logic; test data (2027) is a separate module that overlays real IDs and is cleaned up by `cleanup_testdata`; real data always wins.

## 7. How to restart the local app

Two terminals (the backend does not start the frontend):

```
cd C:\003ClaudeProjects\06AFLAgent\app
uv run uvicorn src.api.main:app --reload
```
```
cd C:\003ClaudeProjects\06AFLAgent\app\frontend
npm run dev
```
Open `http://localhost:5173/` → **2027 Shortlist**. Tests: `uv run pytest -q` (about 3 minutes). Frontend checks: `npm run build` and `npm run lint` in `app/frontend`. Check out the right branch first: `git checkout preseason-phase1` in `app/`.

## 8. Next steps, in order

1. **After 30 Nov (player list final):** refresh the rosters and prices; rebuild `player_master_list_2027.csv` (the 2027 master); rerun `shortlist --save`; **check the Team on each of your attribute rows** (they carry the 2026 team; moved players such as Rowan Marshall and Butters will not match until updated).
2. **Keep building your shortlist** (AF and SC). Screen ideas not built: a "Reference" label for the Change column on Returning players; pre-fill Role for single-position players; a search box; a squad-shape counter (6 DEF / 8 MID / 2 RUC / 6 FWD).
3. **Phase 1, projection model** (`09_PHASE1_PLAN.md` 5): error analysis (WP2), config and special rules (WP3: fixed value, factor, drop-lowest-N, with a base), established-player improvements (WP4), the **draftee model** (WP5: dossier, previous draftees by pick band, the `DrafteeAnalysis` PDFs), the ruck-break decision, the consolidated output file.
4. **2027 test data generator** and `cleanup_testdata` (design in `08` section 5b: separate prefix, test IDs from 900,000,001, overlay on real IDs, `data_source` flag, 2026 fixture as stand-in until the real one exists).
5. **SageMaker parity run** (`notes/08_SAGEMAKER_NOTES.md`). Docker is available on this machine. The 2027 fixture does not exist, so the parity run uses the **2026 data and fixture**. Plan: put the inputs in S3, build a container with the locked library versions (Python 3.11, pandas 3.0.5, scikit-learn 1.7.2), run once, compare 782 rows each for AF and SC; then try XGBoost and tuning as comparison experiments. Check the billing Free Tier page and Service Quotas first.
6. **AWS app build** only after the model is stable: per-user state (DynamoDB), tools (Lambda), Gateway + Cedar, the 5174 web page, a cost report.
7. **Housekeeping:** decide the Athena result expiry rule (they are temporary query outputs, 125 KiB so far); look at Cost Explorer by service in early November; the `AFLHarness` stack is kept deployed for reference (idle, no charge known).

## 9. Open questions (answer when you resume)

SuperCoach evidence thresholds (110/130 assumed); how the 80% "returning" rule is defined (it was an example); acceptable tolerance for a model change (0.3 MAE assumed); club for draftees ranked 21-50; Role/Band semantics for "value" (points over price vs scarcity); whether to merge `preseason-phase1` into `main`; which of the two unrelated untracked files to keep.

## 10. Documents

`08_APP_REQUIREMENTS.md` (requirements, decisions, open questions) · `09_PHASE1_PLAN.md` (evaluation, findings, work packages, build slices) · `notes/08_SAGEMAKER_NOTES.md` · `notes/` 01 to 07 (concepts, how we tested, lessons, harness vs Runtime, PKCE, environments) · `diagrams/architecture_app_v0.1.pdf` (sponsor view) · `LnPosts/` (posts; `HarnessVsRuntimePost_draft.txt` and `LessonsPost_draft.txt` are unposted drafts) · `poc/TRACE_LOG.md` (every observation) · and in the app repo's journal `00_Planning/EXECUTION_PLAN.md` (dated entries).

## 11. Working rules (from the owner)

You run the commands; I give a card (what, why, expect, undo) with predictions first; list the files I will touch before changing anything; counts and capability grids, **no clock-time comparisons**; no tokens or secrets in chat, git or posts; scan before committing; commit and push only when asked; posts are generic, disclose that you are learning with Claude, and claim only observed facts; own up to mistakes in the notes.
