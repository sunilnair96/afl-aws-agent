# 03 — Scope: data, reuse, new code (v0.1, 2026-10-08)

## 1. In scope for the first AWS release

| In | Out (later or never) |
|---|---|
| AFL Fantasy + SuperCoach **starting team** build, fine-tune by chat | DFS optimiser (P7+; `dfs_builder.py` is separate by design) |
| Player compare, ceiling, consistency/reliability search by price range | Tipping predictor, draftee dossier (stretch) |
| Per-user team, locks, avoid list, trade planning (propose-only) | Automatic trade execution — **never**; stays propose-only |
| Cognito auth, two entitlement tiers (A: all seasons, B: ≥2025) | Social/leagues, payments, public sign-up |
| Curated data 2022–2026 + 2027 as it lands | Running the scrapers/model training in AWS |

## 2. Data

### Move to S3 (shared, read-only to users) — `curated/`
| Source (local) | Target | Notes |
|---|---|---|
| `02_Data/processed/player_match_2022..2026.csv` (or the existing `player_match_*.parquet` in afl-analytics) | `player_match/year=YYYY/` | **Add `PlayerId`** to every year (known gap) |
| `masterData/player_master_list_2026.csv` (+ 2027 when issued) | `player_master/year=YYYY/` | has AF/SC price estimates, eligibility flags |
| `player_identity.parquet` (+ internal) | `player_identity/` | extend with team+season context to separate same-name players |
| `processed/season_tables_2022_2026.csv`, `shortlist_2027_{AF,SC}*.csv` | `projections/…`, `shortlist/…` | shortlist `_excluded` files keep their reasons |
| `processed/*_price_roster_2026.csv`, `round*_predictions.csv`, `supercoach_round1_2026_predictions.csv` | `prices/…`, `projections/…` | |
| Fixture (`raw/footywire` fixture output) | `fixtures/year=YYYY/` | needed for opponent-aware projections |
| `masterData/supercoach_positions_2026.csv` | `reference/` | SC position eligibility |
| *derived (new)* `player_season_summary` | `player_season_summary/` | built in P1 from player_match; powers ceiling/consistency |
| `config/guardrails.yaml`, squad configs, `player_name_aliases.csv`, `player_name_overrides.csv` | packaged **inside** the tool image (versioned with code) | config, not data |

### Move to DynamoDB (per-user, read/write)
`fantasy_current_squad`, `supercoach_current_squad`, `*_target`, `*_squad_meta`, `premium_locks_*`, `player_avoid_list*`, `*_trade_log_2026` → one item type each, keyed by user. **Your own data becomes user #1** (seed script reads the CSVs once).

### Stays local / not moved
`02_Data/raw/**` scraped HTML/CSV (regenerable), `processed/features_*.csv` and model artefacts (regenerable, large), `raw/DrafteeAnalysis` (16 MB, later), `Working/`, `_backup_*`, `04appDocumentation`, `05Conversations`. `seasons.csv` becomes a small S3 reference file read-only (the active-season concept), not a per-user table.

Volume check: all of `processed/` is ~37 MB and `raw/footywire` ~7 MB — S3/Lambda-in-memory is a non-issue; cost is dominated by Bedrock tokens, not storage.

## 3. Code reuse

| Existing | Use in AWS | Change needed |
|---|---|---|
| `common/team_builder.py` (PuLP) | `build_team` / `adjust_team` Lambda (container) | none to the solver; inject data via an interface |
| `common/guardrails.py` + `guardrails.yaml` | inside team tools | package config |
| `common/player_compare.py` | `compare_players` | add ceiling/consistency fields; accept `min_season` |
| `common/trade_planner.py`, `current_squad.py`, `trade_service.py` | `propose_trades` | swap CSV loaders for repository interface |
| `common/premium_locks.py`, `api/csv_table.py` | `team-service` | replace CSV backend with DynamoDB repository |
| `common/team_names.py`, `player_names.py`, `player_identity.py` | shared lib (Lambda layer / package) | merge with the afl-analytics versions (they duplicate each other) |
| `common/paths.py` | — | **key refactor:** introduce a `DataStore` interface (local FS impl, S3 impl) so the same functions run locally and in Lambda |
| `models/season_projection.py`, `shortlist.py`, features, ingestion, cleaning | stays local; outputs published | add `publish_to_aws` step |
| `mcp_server/server.py` tool docstrings/schemas | blueprint for Gateway `tools.json` | tool names/params become the Gateway contract |
| `api/main.py` (FastAPI) | optional: wrap with Mangum for `team-service` | decide in P4; admin/ops endpoints are **not** exposed |
| `frontend/` (React, SVG charts) | SPA base for P6 | add Cognito login, chat panel; hide admin pages (Seasons, Round Ops, CSV editors) |
| afl-analytics: `s3_utils`, pipelines, identity builder | seed for publish/identity jobs | fix gaps in §1 of the plan; add tests |
| afl-analytics: Lambda `afl-get-player-performance`, `tools.json`, gateway + Cedar config, Cognito setup | P2/P3 starting point | param schema + group-based policy |
| afl-analytics: `harness.json` / `system-prompt.md` | system prompt seed | rules → competition rules (`00_Planning/COMPETITION_RULES.md`) |
| tests with frozen `tests/fixtures/` | run in CI against Lambda handlers | add AWS-event fixtures |

## 4. New code

| Component | Description | Phase |
|---|---|---|
| `publish_to_aws` | schema-validate + sync curated outputs to S3, write a manifest | P1 |
| `player_season_summary` builder | avg, p20 floor, p90 ceiling, coefficient of variation, games-played %, price/value | P1 |
| Tool Lambdas | `get_player_performance` v2, `compare_players`, `find_players`, `get_my_team`, `build_team`, `adjust_team`, `propose_trades`, `save_team` | P2/P4 |
| `DataStore` + repositories | S3 data store; DynamoDB repos for team/locks/avoid/trades | P2/P4 |
| Gateway interceptor Lambda | verify token, resolve entitlement, overwrite `user_id`/`min_season` | P3 |
| Cedar policies (group-based) | replace the per-`sub` policy | P3 |
| `team-service` Lambda + API Gateway | non-LLM CRUD for the UI | P4/P6 |
| Strands agent for Runtime | tool loop, token passthrough, prompt, memory, guardrail config | P5 |
| Eval harness | scripted conversations with expected tool calls/outcomes + the authorization matrix | P3/P5 |
| IaC (CDK Python) | Cognito, S3, DynamoDB, Lambdas, API GW, CloudFront, alarms | P1→P7 |
| User provisioning script | create Cognito user + group + `PROFILE` item | P3 |

## 5. Definition of "consistent and reliable" (needs your sign-off)

Proposed defaults for `find_players` (all derived in `player_season_summary`, over a lookback window the user can change):
- **Consistency:** coefficient of variation (σ/mean) of AF or SC, low is better; plus *floor* = 20th percentile score.
- **Ceiling:** 90th percentile, and mean of best 3 scores.
- **Reliability:** games played ÷ games available, % of games ≥ a threshold (e.g. ≥ 70), no live injury/named-status flag (`named_status.py`).
- Price range filter uses the current AF/SC price (rosters), position filter uses eligibility (DPP/TPP-aware).
