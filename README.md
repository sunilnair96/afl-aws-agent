# 07_AWS — AFL Agent on AWS (Bedrock AgentCore + Lambda)

*Created 2026-10-08. Status: **initial plan (v0.1) — nothing built yet in this folder.***

Goal: rebuild (and enhance) the 06AFLAgent app as a multi-user AWS service where a Cognito-authenticated
user chats with a Bedrock agent that builds and manages **their own** AFL Fantasy / SuperCoach team, without
ever touching the data directly. Secondary goal: build real expertise in Bedrock AgentCore and Lambda.

| File | What it is |
|---|---|
| [RESUME_HERE.md](RESUME_HERE.md) | **Start here when resuming (written 2026-10-11):** status, where everything is and what is pushed, what was built and learned, how to restart, next steps in order, open questions |
| [01_PLAN.md](01_PLAN.md) | Goals, principles, phases, timeline, decisions, risks, open questions |
| [02_ARCHITECTURE.md](02_ARCHITECTURE.md) | Architecture narrative, request flows, authorization model, diagrams (Mermaid) |
| [architecture_v0.1.svg](architecture_v0.1.svg) | Standalone architecture diagram (open in a browser) |
| [03_SCOPE_DATA_CODE.md](03_SCOPE_DATA_CODE.md) | What data moves to AWS, what code is reused, what is new |
| [04_RUNBOOK_PHASE0.md](04_RUNBOOK_PHASE0.md) | Step-by-step instructions to start: account hygiene, unblock the stuck harness, spikes, plugin |
| [05_POC_PLAN.md](05_POC_PLAN.md) | **Start here:** the small identity/A-vs-B proof-of-concept, with trace points and stages |
| [06_LINKEDIN_SERIES.md](06_LINKEDIN_SERIES.md) | LinkedIn post plan + Post 3 skeleton |
| [08_APP_REQUIREMENTS.md](08_APP_REQUIREMENTS.md) | **The real AWS app (draft v0.1):** two products (team builder + weekly trades, draft planner), users, data, security, Runtime decision, reuse table, source layout (port 5174), cost plan, open questions |
| [09_PHASE1_PLAN.md](09_PHASE1_PLAN.md) | **Phase 1:** stabilise and evaluate the projection model (established players and draftees), run it on AWS, freeze by 20 February 2027; work packages with gates, recommendations, open questions |
| [diagrams/architecture_app_v0.1.pdf](diagrams/architecture_app_v0.1.pdf) | One-page sponsor-style architecture of the AWS app (source HTML alongside) |
| [notes/01_CONCEPTS.md](notes/01_CONCEPTS.md) | **Study notes, read in order:** concepts and where each lives in your project |
| [notes/02_HOW_WE_TESTED.md](notes/02_HOW_WE_TESTED.md) | The testing method, the 11 experiments, and what I got wrong |
| [notes/03_KEY_LESSONS_AND_READING.md](notes/03_KEY_LESSONS_AND_READING.md) | Ten lessons, self-test questions, exercises, reading list with links |
| [notes/08_SAGEMAKER_NOTES.md](notes/08_SAGEMAKER_NOTES.md) | SageMaker notes: what it is, what would run there, what is required, will the result differ (same code and versions: same; built-in algorithms: different by design), options, parity test plan, cost and free-tier facts with sources |
| [notes/07_HARNESS_EXPERIMENT_PLAN.md](notes/07_HARNESS_EXPERIMENT_PLAN.md) | Planned (not run): rebuild the same function with a harness, measure time/effort/visibility/access control; what AWS docs say about harness → Gateway auth |
| [notes/06_ENVIRONMENTS_AND_DEPLOYMENT.md](notes/06_ENVIRONMENTS_AND_DEPLOYMENT.md) | DEV→PRD with AgentCore targets: what we tried, 12 findings, the target pipeline and per-environment config, my mistakes, open tests |
| [notes/05_WEB_LOGIN_AND_PKCE.md](notes/05_WEB_LOGIN_AND_PKCE.md) | The browser test page: public client, PKCE in plain words, the four traps hit, commands to recreate/remove, results |
| [notes/04_HARNESS_VS_RUNTIME.md](notes/04_HARNESS_VS_RUNTIME.md) | Harness vs Runtime comparison, framework used, and the outline of the future "when to use which" paper (config snapshot in `notes/harness_snapshot/`) |
| [poc/CHEATSHEET_agent_to_gateway.md](poc/CHEATSHEET_agent_to_gateway.md) | **Read first after a break:** how the agent→gateway problem was fixed, errors + fixes, commands |
| [poc/POST2_REVIEW.md](poc/POST2_REVIEW.md) | Pre-publication checklist: what in the unpublished Post 2 held up, and what to fix |
| [poc/LinkedInPost_2_DRAFT.txt](poc/LinkedInPost_2_DRAFT.txt) | Rewritten identity post (unpublished) with the agent-hop finding |
| [poc/architecture_post3.html](poc/architecture_post3.html) | Printable A4 diagram: failed path vs working path (open in Chrome → Print → Save as PDF) |
| [poc/TRACE_LOG.md](poc/TRACE_LOG.md) | Observed values at each trace point (no secrets) |

## Ground rules for this folder

- **Repo layout rule (agreed 2026-10-08):** code that deploys or runs against AWS (AgentCore config, Runtime/Lambda code, tests such as `poc_matrix.py`) lives in the GitHub repo `C:\005awsprojects\afl-analytics` (private). This folder keeps notes: plans, trace log, cheat sheet, diagrams, LinkedIn drafts.
- This folder = AWS **notes and plans** only (docs, trace log, diagrams, post drafts). Later phases may still add IaC here; decide at phase P4. The local app in `../app/`
  stays the source of truth for the **2027 build in Jan 2027** until the AWS version passes the parity gate in `01_PLAN.md`.
- No secrets, tokens, or `.env` files here. Ever. (See the cleanup item in `04_RUNBOOK_PHASE0.md`.)
- This folder is its own git repo (initialised 2026-10-08, branch `main`, no commits or remote yet; `.gitignore` blocks tokens, `.env`, parquet/csv).
- Log progress as dated entries in `../00_Planning/EXECUTION_PLAN.md` (pointer added 2026-10-08).

> Identifiers (AWS account, Cognito pool/client/user ids, gateway/harness/runtime ids, bucket and IAM user names) are replaced with `<PLACEHOLDERS>` in these notes. The real values live in the project's own config and are not needed to read the notes.
