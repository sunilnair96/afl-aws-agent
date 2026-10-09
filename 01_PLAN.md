# 01 — AWS Build Plan (v0.1, 2026-10-08)

## 0. Revision 2026-10-08 (after your answers)

- **Pilot = you only** (User A and User B are both you, two Cognito accounts). Purely learning; licensing risk in §7 is therefore low for now but would return if anyone else were given access.
- **Timebox: ~2 months (to ~8 Dec 2026)** inside the free-plan window (≈170 days from today, to ~27 Mar 2027 — verify in the Billing console).
- **New first step: the POC** in `05_POC_PLAN.md` (identity flow + A/B access, no real app). The phases below follow it. Roughly: wk1 hygiene + POC-0, wk2–4 POC (P1–P4), wk5–8 core tools and team building (P1/P2/P4 below, compressed), agent evals; React hosting (P6) and hardening (P7) fall **after** the 2 months and are optional.
- **LinkedIn:** each milestone produces a post (`06_LINKEDIN_SERIES.md`). The Jan 2027 team build still uses the local app.
- **Working rule:** you run every command; each gets a What/Why/Expect/Undo card first.
- Repo: `07_AWS/` is now its own git repo (`main`, no commits yet).

## 1. Where we are

**Local app (`../app`, mature):** data pipeline 2022–2026, projection model, PuLP solvers (Fantasy, SuperCoach, DFS),
guardrails, trade planner, player compare, shortlist, FastAPI + React UI, MCP server (`afl-tools`). Single user,
CSV files in `02_Data/`.

**AWS learning project (`C:\005awsprojects\afl-analytics`, repo `sunilnair96/afl-analytics`):** already working:
- S3 bucket `<S3_BUCKET>` (raw CSV → processed Parquet, partitioned by year), Athena tables.
- Identity: `FootywirePlayerId` + internal `PlayerId` (`player_identity.parquet`: 318 internal + 810 Footywire).
- Lambda `afl-get-player-performance` (ap-southeast-2).
- Cognito user pool + app client; AgentCore Gateway `AFLGatewayJWT` (CUSTOM_JWT) with Cedar policy engine
  (`UserBRecentSeasons`: user B may call the tool only when `season >= 2025`) — **the user A / user B scenario works at the Gateway**.
- AgentCore harness `teamHarness` (Bedrock Claude Sonnet) + managed memory.

**Where you got stuck** (`SecurityArchitecture.png` step 2→3, and `BUG_REPORT_defaultReturnUrl.md`): the harness calling the
Gateway via *outbound OAuth user federation* demands a `ResourceOauth2ReturnUrl` and fails at tool load. That flow is the
3-legged consent pattern for calling *third-party* services on a user's behalf. Our case is simpler — same Cognito, same user —
so the diagram's pattern ("pass the user's JWT in the request header") should be done by **forwarding the inbound token**, not by
a second OAuth dance. See decision D1 and spike S1.

**Known gaps in the learning project** (from its own `PROJECT_REVIEW.md`): match files 2022–2025 have no `PlayerId`;
match pipeline only accepts Footywire IDs; identity table has no team/season context (two "Max King", two "Bailey Williams");
no tests; no IaC for the non-AgentCore pieces. Phase 1 fixes these.

## 2. Target (what "done" means)

1. A registered user (Cognito) signs in to a web app and chats with an agent:
   - "Build me an optimised AFL Fantasy starting team" / "…SuperCoach…" → valid, solver-verified squad (cap, positions, DPP/TPP, club minimum, guardrails).
   - "Swap Player X for someone ≤ $600k who is consistent" → adjusted team, re-validated.
   - "How does Nick Daicos compare with Harry Sheezel? Who has the better ceiling?"
   - "Find me a midfielder priced $450–600k who is consistent and reliable."
2. A user sees **only their own team**. Users never hold credentials that can reach S3/Athena/DynamoDB; only the app path works.
3. Data entitlement tiers enforced outside the LLM: user A = all seasons, user B = 2025 onward (extensible to more tiers).
4. Every number the agent states comes from a tool (no invented stats); every squad is checked by the solver/validator before being called "valid".
5. Observable, cost-capped, deployable from code (IaC), with an evaluation set the agent must pass before each release.

## 3. Design principles

- **LLM reasons; code decides.** Solver, validation, guardrails and authorization are deterministic code/policy, never prompt text.
- **Identity travels with the request** and is enforced outside the LLM (Cognito JWT → Gateway → Cedar + interceptor → Lambda) — exactly your diagram.
- **The LLM never supplies `user_id` or entitlements.** They are derived from the verified token.
- **Propose, then confirm.** Team-changing tools are two-step (build/preview → explicit save), as in the local app.
- **One logic base.** The AWS tools wrap the same `src/common/*` code as the local API/MCP server (a third thin front door, never a fork).
- **Local app stays usable** throughout; AWS is not on the critical path for the Jan 2027 team build until parity is proven.

## 4. Phases

Dates are indicative (today = 2026-10-08; 2027 lists land Nov–Dec; team build Jan 2027).

| Phase | Outcome | Learning focus | Indicative |
|---|---|---|---|
| **P0 Foundations + unblock** | Account hygiene (MFA, budget alarms, least-privilege), secrets cleanup, spikes S1–S3 answered, decisions D1–D6 closed | IAM, Cognito claims, AgentCore auth model | Oct 8–18 |
| **P1 Data foundation** | Curated Parquet for 2022–2026 in S3 with `PlayerId` on every row; Glue/Athena tables; derived `player_season_summary` (avg, floor, ceiling, consistency, availability); publish script | S3 layout, Glue, Athena, partitioning, data contracts | Oct 19–Nov 1 |
| **P2 Read tools (Lambda)** | `get_player_performance` v2, `compare_players`, `find_players` (price range + consistency + reliability) behind the Gateway; unit tests with frozen fixtures | Lambda packaging (container image), layers, cold starts, tool schemas | Nov 2–15 |
| **P3 Identity + authorization** | Cognito groups (`full-history`, `recent-only`), Cedar policies by group, Lambda interceptor injecting `user_id`/`min_season`, A/B/C test matrix automated | Cedar, interceptors, JWT claims, LOG_ONLY→ENFORCE | Nov 16–29 |
| **P4 Team tools + per-user state** | DynamoDB tables (teams, locks, avoid, trade log, entitlements); `build_team`, `get_my_team`, `adjust_team`, `propose_trades`, `save_team` (two-step); solver in Lambda | DynamoDB single-table design, container Lambda w/ PuLP, idempotency | Nov 30–Dec 20 |
| **P5 Agent** | AgentCore Runtime agent (Strands) with token passthrough, system prompt with competition rules, AgentCore Memory, Bedrock Guardrails, **evaluation set** (extends your "test evaluation set" commit) | Bedrock models/inference profiles, prompts, memory, evals | Dec 21–Jan 10 |
| **P6 Web app** | React SPA on S3+CloudFront, Cognito Hosted UI (PKCE), chat panel + "My Team" page via API Gateway | CloudFront/OAC, CORS, streaming | Jan–Feb |
| **P7 Harden + operate** | IaC for everything, CI/CD, dashboards/alarms, per-user rate/cost caps, scheduled data refresh (EventBridge + Step Functions) in season | CDK, observability, FinOps | Feb–Mar |

### Parity gate (before AWS is used for the real 2027 build)
AWS agent reproduces the local app's `/build` output (same squad, ±tie-breaks) for Fantasy and SuperCoach on the 2026 data, passes the
authorization matrix (§6), and the eval set ≥ agreed pass-rate. If not met by early Jan, build 2027 locally and keep AWS going in parallel.

## 5. Decisions

| # | Decision | Recommendation | Why |
|---|---|---|---|
| D1 | Agent host: AgentCore **harness** (config-managed) vs AgentCore **Runtime** (your own agent code) | **Runtime + Strands**, forwarding the inbound `Authorization` header to the Gateway. Timebox one more harness attempt (S1). | Runtime supports passing `Authorization` to agent code via the request-header allowlist (requires a JWT inbound authorizer — which you have). Removes the 3-legged OAuth step entirely; you also learn more by owning the agent loop. |
| D2 | Tool data access: **Athena** vs **pandas/DuckDB on Parquet in Lambda** | Lambda reads Parquet from S3 and reuses existing pandas code. Keep Athena for ad-hoc analysis and for your learning. | Data is tiny (tens of MB). Athena adds seconds of latency + per-scan cost per tool call; the solver/compare code is already pandas. |
| D3 | Packaging | Lambda **container images** for solver tools (PuLP + pandas + pyarrow exceed zip comfort); zip/layer for light tools. | PuLP/CBC binary + pandas ≈ large. |
| D4 | IaC | **AWS CDK (Python)** for app stacks; AgentCore CLI keeps managing its own stack until P7, then consolidate. | Your project is Python; the CLI already generates CDK. |
| D5 | Per-user state | **DynamoDB**, partition key `USER#<cognito sub>`. | Serverless, trivial isolation by key, cheap. |
| D6 | Entitlements | Cognito **groups** for the tier + a `users` item in DynamoDB holding `min_season` etc.; interceptor resolves them from the verified token. Avoid hard-coding a user `sub` in policy (as the current `UserBRecentSeasons` does). | Scales beyond two test users. |
| D7 | Chat transport (P6) | Browser → AgentCore Runtime directly with the bearer token if CORS/streaming allow; else a thin Lambda Function URL streaming proxy. | Decide in P6 after a spike; CLI/test client is enough until then. |
| D8 | Models | Latest Sonnet via inference profile for the agent; a Haiku-class model for cheap sub-tasks later. Confirm exact Bedrock model IDs/availability in ap-southeast-2 in the console. | Cost/latency; I have not verified current Bedrock IDs. |

## 6. Authorization test matrix (automate in P3)

| Caller | Tool call | Expected |
|---|---|---|
| No token / bad signature / wrong client | any | 401 at Gateway |
| User A (`full-history`) | `get_player_performance(season=2023)` | allow |
| User B (`recent-only`) | `season=2025` | allow |
| User B | `season=2023` | **deny** (Cedar) — Lambda not invoked |
| User B | `compare_players` with no season argument | allowed, but result silently clipped to ≥2025 by interceptor-supplied `min_season` (and Lambda re-checks) |
| User A | `get_my_team(user_id=<B's sub>)` (prompt-injected) | interceptor overwrites with A's sub; B's data never returned |
| User A | `save_team` without a prior preview token | rejected |
| Any user | direct S3/Athena/DynamoDB calls with Cognito tokens | no IAM path exists |

## 7. Risks

| Risk | Mitigation |
|---|---|
| Token passthrough from Runtime→Gateway behaves differently than assumed | Spike S1 first; fallback = harness with user federation (fix return URL) or Lambda-side token exchange |
| Cedar can't read Cognito groups as principal attributes | Spike S2; fallback = interceptor maps group→`min_season` and Cedar checks the injected value, or generate per-group policies |
| Solver latency/timeouts through Gateway→Lambda | Container Lambda 2–3 GB, benchmark; async job pattern if >30 s |
| LLM cost per user chat | Max iterations/tokens (already in harness config), per-user daily caps in DynamoDB, Budgets alarm, prompt caching |
| Scope creep (3 competitions + trades + DFS) | Fantasy + SuperCoach starting team first; trades P4; DFS later |
| **Data licensing** — Footywire-scraped and Champion Data–derived stats shown to other users | Keep to a private pilot of known users; review source terms before any public launch |
| Secrets in git | `AwsCommands.txt` is tracked in `afl-analytics` (contains an access token — expired, but also account/pool/client IDs). Untrack, scrub, keep repo private (runbook step 1.4) |
| Bedrock/AgentCore regional or CLI churn (CLI 0.31.x, fast-moving) | Pin versions, keep `agentcore.json` as source of truth, record versions in the runbook |

## 8. Open questions for you (none block P0)

1. Is this a private pilot (you + a few friends) or could it be public? Drives licensing, rate limits, and whether you need email verification/MFA flows.
2. Do you want the app to stay single-region ap-southeast-2 (recommended)?
3. Which competitions first for users: Fantasy only, or Fantasy + SuperCoach together? (Plan assumes both, Fantasy first.)
4. Should I `git init` `07_AWS/` as its own repo (`afl-agent-aws`)?
5. Budget ceiling per month for dev? (Plan assumes a hard alarm at A$50, soft at A$20 — adjust.)
