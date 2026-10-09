# 04 — Runbook: Phase 0 (foundations, unblock, spikes)

Shell: Windows. Region: `ap-southeast-2`. Do steps in order; tick as you go. Nothing here deploys the new app yet.

## 1. Account hygiene (≈1–2 h)

1.1 **Root user:** enable MFA, remove any access keys, use it only for billing/account tasks. (You created it — confirm it has no keys.)
1.2 **Budgets:** Billing → Budgets → create a monthly cost budget with alerts at A$20 (actual) and A$50 (forecast) to your email. Turn on *Cost Anomaly Detection*.
1.3 **`<DEV_IAM_USER>`:** fine for now, but note it is long-lived. Prefer short-lived credentials (`aws login`/Identity Center). Plan to replace blanket admin with a scoped policy at P7.
1.4 **Secrets cleanup in `afl-analytics`:** `AwsCommands.txt` is **tracked in git** (and the remote is GitHub) and contains an access token plus account, pool and client IDs. The token has expired, but:
```bash
cd /c/005awsprojects/afl-analytics
git rm --cached AwsCommands.txt
echo AwsCommands.txt >> .gitignore
git commit -m "Stop tracking command scratchpad"
```
   Also confirm the GitHub repo is **private**. Consider `AFLStartingTeamAgent/agentcore/.cli/deployed-state.json` (account IDs) the same way. Keep scratch commands in `07_AWS/` *without* tokens.
1.5 `cognito_test.py` uses a **client secret** from an env var — fine for a CLI test, but the web app needs a second, **public** app client (no secret, PKCE). Create it in P3.

## 2. Install the AWS deploy plugin (your pasted commands)

Yes — install it, but treat it as a **P1/P2 helper (IaC + cost estimates), not something to hand the keys to now.**
What it is (per its README): a `deploy` skill plus three MCP servers (AWS knowledge, AWS pricing, IaC best practices); it recommends architecture, estimates cost, generates CDK/CloudFormation, and asks for confirmation before deploying. Needs Claude Code ≥ 2.1.29 and a configured AWS CLI.

`/plugin` is an interactive terminal command, so it can't be run from this desktop-app session. In a terminal `claude` session in this folder:
```
/plugin marketplace add awslabs/agent-plugins
/plugin install deploy-on-aws@agent-plugins-for-aws
/reload-plugins
```
Guardrails for using it: always review generated IaC and the pricing estimate before approving; run with a profile that can only act in this account; never let it create IAM users/keys. Use it first for **cost estimate of this architecture** (ask it to price `02_ARCHITECTURE.md` at ~10 users).

## 3. Record the current baseline

Run and paste the outputs into a dated note in `00_Planning/EXECUTION_PLAN.md` (no tokens):
```bash
aws sts get-caller-identity
aws --version
agentcore --version
cd /c/005awsprojects/afl-analytics/AFLStartingTeamAgent && agentcore status
```

## 4. Spikes (each ≤ half a day; each answers a [verify] in the architecture)

**S1 — Token passthrough Runtime → Gateway (decides D1).**
Goal: a minimal Strands agent on AgentCore Runtime with a Cognito JWT inbound authorizer and `requestHeaderAllowlist: ["Authorization"]`, which calls `AFLGatewayJWT` with the **same** bearer token and gets `get_player_performance` back.
Success: user B asking for 2023 is denied by Cedar *through the agent*; user B for 2025 works. No OAuth return-URL step involved.
Timebox attempt B (harness): re-test after updating the CLI; check whether `outboundAuth` can be omitted for same-IdP forwarding; if the return-URL error persists, add your bug report to the AgentCore CLI repo and move on with Runtime.

**S2 — Can Cedar see the user's group/claims?**
Try a policy using principal tags/claims (e.g. group `recent-only`) instead of a hard-coded `sub`. Success: one policy covers any user in the group. Fallback: interceptor maps group→`min_season` and Cedar checks `context.input.season >= context.input.min_season`-style rule on the injected value (or generate per-user policies from DynamoDB at provisioning).

**S3 — Interceptor overwrites arguments.**
A request Lambda interceptor that decodes the validated token, looks up the entitlement, and replaces `user_id` and `min_season` in the tool arguments. Success: an agent prompt-injected to pass another user's `user_id` still receives only its own team (use a dummy `get_my_team`).

**S4 — Solver Lambda footprint.**
Container-image Lambda with pandas + pyarrow + PuLP: measure image size, cold start, and a full Fantasy build at 2/3 GB memory. Success: p95 < ~20 s warm; otherwise design the async job pattern.

## 5. Close the decisions

Review `01_PLAN.md` §5 (D1–D8) and §8 questions; reply with changes. After S1–S3 I update D1/D6 and the architecture to v0.2.

## 6. Exit criteria for Phase 0

- [ ] Root MFA on, budgets live, secrets untracked
- [ ] Plugin installed (optional) and used for a cost estimate
- [ ] S1–S4 results written up (a paragraph each) in `00_Planning`
- [ ] Decisions D1–D8 closed; architecture bumped to v0.2
- [ ] `07_AWS` repo initialised (if you want it) and first commit made
