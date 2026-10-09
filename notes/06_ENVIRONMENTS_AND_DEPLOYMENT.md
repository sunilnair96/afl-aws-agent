# 06 — Environments and deployment (DEV → PRD): what we tried, what we found, what a real pipeline needs

*Written 2026-10-09 after the "second target" exercise. Everything marked **observed** was seen in your own runs; **[unverified]** items were not tested; **[unknown]** items we could not find out. Several of my own claims during the exercise turned out wrong, and they are listed in section 7.*

## 1. The idea

Teams deploy the same code through a series of environments, for example **DEV → SIT → UAT → PRD**. Each has its own users, data and settings, so a mistake in an early one can't touch real users. The code is identical in every environment; **only the configuration differs**. The usual best practice is **one AWS account per environment**.

## 2. What the AgentCore CLI gives you (read from your project and the CLI help)

| Piece | What it is | Where |
|---|---|---|
| **Target** | A named (account, region) pair | `agentcore/aws-targets.json` (ours had only `default`) |
| **One stack per target** | `agentcore deploy --target <name>` creates/updates a CloudFormation stack named `AgentCore-<project>-<target>` | observed: `AgentCore-AFLStartingTeamAgent-default` |
| **State per target** | Runtime ids, ARNs and so on | `agentcore/.cli/deployed-state.json`, keyed by target |
| **`--diff` / `--dry-run`** | Preview a deployment without making it | `agentcore deploy --help` |
| **`-y`** | Auto-confirm prompts | `agentcore deploy --help` |
| **Target prompt** | With 2+ targets, a plain `agentcore deploy` asks which to deploy ("All Targets" is the first choice) | observed |
| **`agentcore dev`** | Runs the agent on your PC for quick iteration; by default it also **deploys resources first** (`--skip-deploy` disables that) | CLI help, **not run** |
| **`endpoints`** | In the config schema: `{name: {version: N}}`, a named endpoint pinned to a runtime version | `.llm-context/agentcore.ts`; behaviour documented by AWS, see 7b; **not run by us** |

## 3. What we did, in order

1. Added a `prod` target (same account and region) → `agentcore validate` OK.
2. `agentcore deploy --target prod --diff` → 15 resources would be created; nothing was created (the stack did not exist afterwards).
3. Found the **Cedar policies contained the dev gateway's ARN** (my first check of this was faulty; see section 7).
4. Tried to make the policy text neutral **in dev first**: bare `resource` → rejected; `resource is AgentCore::Gateway` → rejected. Both rolled back; dev kept working.
5. Real `agentcore deploy --target prod` → failed on creation: policy-engine name already exists. Rolled back to `ROLLBACK_COMPLETE`.
6. Cleanup: restored files from git, restored `deployed-state.json`, deleted the empty prod stack, `agentcore deploy --diff` on dev: **"No differences"**.

## 4. Findings

| # | Finding | Evidence |
|---|---|---|
| 1 | A second target in the **same account and region** collides on named resources | `CreatePolicyEngine … 409 ConflictException: Policy engine with the same name already exists`. Names are `<project>_<resource>` with no target in them (policy engine `AFLStartingTeamAgent_AFLAuthorizationPolicy`, runtimes `AFLStartingTeamAgent_s1c_agent`, gateway `AFLStartingTeamAgent-AFLGatewayJWT`); only the first conflict was reported |
| 2 | **A tool-specific Cedar policy must name one specific gateway ARN.** The service rejects both a wildcard and a type constraint | 13:57: "a wildcard resource was detected…"; 14:05: "a constrained action scope was encountered, please constrain the resource to a specific AgentCore::Gateway resource when creating tool-specific policies" |
| 3 | So policy text is environment-specific by rule: the ARN holds the account id and the gateway id, which differ per environment | follows from 2 |
| 4 | `envVars`, `connections`, `.env.local` and the OAuth credential providers are single-valued; **no per-target override found** | schema; the provider records were identical in `default` and `prod`; both targets read one `.env.local` |
| 5 | The two OAuth providers are **account-level** and were shared by both targets; nothing now uses them (they were for the harness) | provider list unchanged by the diff; same ARNs in both state entries |
| 6 | Both Lambdas, the Cognito pool and its clients, and the CDK bootstrap bucket/key are outside the stack, so any target in the account shares them | the prod diff's gateway-role statement named the same two Lambda ARNs |
| 7 | **`--diff` is a preview, not a promise.** It cannot see name clashes, and it does not run the service's policy validation | 3 policy changes previewed as fine, then rejected at update; the name clash only appeared on creation |
| 8 | A failed **update** rolls back as a unit; the old version keeps running (tests passed on the old policies) | dev events: `UPDATE_FAILED` → `UPDATE_ROLLBACK_COMPLETE`; the stack stays in that status until the next successful update |
| 9 | A failed **create** leaves a `ROLLBACK_COMPLETE` stack that must be deleted before retrying | prod stack |
| 10 | The CLI's message for that failure was unhelpful ("NoStack: CloudFormationStack object does not hold a stack"); the real reason was in the stack events | `aws cloudformation describe-stack-events … ResourceStatus=='CREATE_FAILED'` |
| 11 | Policy changes apply to **existing tokens immediately**; group changes only to new tokens | tokens issued before a policy deploy kept working with the new rules |
| 12 | Generated runtime roles are broad (`bedrock:InvokeModel*` on any model, several actions on `*`) | the diff's IAM table |

## 5. What a real pipeline looks like (the target design)

```
 developer PC ──▶ git push ──▶ CI: tests ──▶ deploy DEV ──▶ tests ──▶ deploy SIT/UAT ──▶ approval ──▶ deploy PRD ──▶ smoke test
 (agentcore dev)                  (unit +       (own acct)   (attack      (own accts)       (a person)    (own acct)     (+ rollback on failure)
                                   attack set)               matrix)
```

**`deploy_to_env <env>` script: what it should do, as design (not built).** Always the explicit environment, never a default.
1. Print the environment, account and region, and refuse if they aren't the expected ones.
2. Generate that environment's config from a template (pool/client ids, policy ARNs, endpoint values).
3. `agentcore validate`, then `agentcore deploy --target <env> --diff`: show it, and require approval for UAT/PRD.
4. `agentcore deploy --target <env> -y`.
5. **Check the final stack status** (`CREATE_COMPLETE` or `UPDATE_COMPLETE`) and exit non-zero otherwise. Observed: a rolled-back deploy looks harmless because the old version keeps working.
6. Run the smoke test against that environment (tokens for that environment's users).
7. On failure: redeploy the previous commit.

## 6. What must differ per environment in our project (not built)

| Item | Now (hard-coded or shared) | In a real environment |
|---|---|---|
| Cognito pool, clients, users, groups | one pool; test users | one per environment; **separate credentials for DEV and PRD** |
| `allowedClients`, discovery URL (`agentcore.json`) | literal | per environment |
| Cedar policies | literal dev gateway ARN | generated per environment (gateway ARN must exist first → create gateway, then policies: two steps) |
| `GATEWAY_URL` in the agent code, and `webtest/app.js` config | constants | environment variable or config; per environment |
| Lambdas | two shared, hand-made | one set per environment, created by code (CDK), the **same code package** |
| **Data location** | fixed bucket and Athena table in the Lambda | read from an **environment variable**: DEV → dev bucket/table, PRD → prod |
| IAM roles | broad, generated | least privilege per environment, specific model ARNs |
| Budgets, alarms, log retention | one | per environment |

**Data, in one rule:** the Lambda's code is identical everywhere; *where it reads from* comes from configuration. For this project: historical player data is shared and read-only (or dev gets a small sample), and **per-user data** (teams, locks) lives in a table per environment. You don't have to duplicate the historical data.

## 7. What I got wrong during this exercise

| Claim | Reality | How it surfaced |
|---|---|---|
| "The template has no dev ARN" (my first check printed `False`) | It did contain it: I read the wrong property, so I compared against nothing | Printing the raw statement; I then added a **positive control** to my checks |
| `resource is AgentCore::Gateway` fixes it | The service also rejects the type constraint | The 14:05 failure; dev's stack events |
| "The retry succeeded" (I accepted it) | The 14:05 update had failed and rolled back; tests passed only because the old policies were restored | `describe-stacks` showed `UPDATE_ROLLBACK_COMPLETE` with a 14:05 timestamp |
| Predicted a name clash would appear in the preview | It did not; it appeared only on creation | The real deploy |
| My edits left `agentcore.json` in a state that could not deploy | Restored from git (empty `git diff`), verified with a dev diff showing "No differences" | |

## 7b. Versions and named endpoints (from AWS documentation; **not yet run by us**)

Source: *AgentCore Runtime versioning and endpoints* (https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agent-runtime-versioning.html), which I read in full. A secondary summary you pasted (blogs and a video, which I did not open) said the same.

- Each runtime update creates a **new immutable version** (V1, V2, …) with a complete configuration.
- The **`DEFAULT` endpoint always moves to the newest version** automatically.
- A **named endpoint** (the page's example is `PROD`) stays on its version until it is **explicitly updated**. The page's table: after a new version V3, "DEFAULT updates to V3, PROD remains on V2"; "Update PROD to V3" is a separate step that creates no new version.
- Endpoint states: `CREATING`, `CREATE_FAILED`, `READY`, `UPDATING`, `UPDATE_FAILED`. Operations: `UpdateAgentRuntimeEndpoint` (API), `ListAgentRuntimeVersions`, `ListAgentRuntimeEndpoints`. In our config: `endpoints: { "<name>": { "version": N } }` on a runtime (schema in `.llm-context/agentcore.ts`).
- A search summary of AWS's docs also said endpoints can be updated "without downtime", with rollbacks. **That sentence was not on the versioning page I read**, so treat it as **[unverified]**. The page says **nothing** about sessions already running on an older version **[unknown]**.

**What this means for us:** the browser page and the agents call `qualifier=DEFAULT`, so every deploy changes what they run immediately. A cheap pattern *inside one account*: keep `DEFAULT` for development, and have anything that must stay stable call a named endpoint such as `prod`, moved only after tests pass. It separates the **runtime code** only; the gateway, the policies, the Lambdas and the data stay shared.

**Results of that experiment (run 2026-10-09 on dev, `s1c_agent`; details in `poc/TRACE_LOG.md`):**

| Step | What happened (observed) |
|---|---|
| Create `prod` pinned to version 3 | One new resource (`RuntimeEndpoint`); `DEFAULT 3`, `prod 3`; no new version |
| Deploy changed code (version 4) | Version 4 created; **`DEFAULT` moved to 4, `prod` stayed on 3**. Stack `UPDATE_COMPLETE` |
| A session already running during the deploy | **Kept answering with the old code** (it was not moved), no failed request |
| A **new** session after the deploy | `DEFAULT` answered by version 4 (`build-B`); `prod` by version 3. **First request of a new session ~6-9 s (cold start)**, later requests fast |
| Promote `prod` 3 → 4 (edit the version number and deploy; no code upload, no new version) | `prod READY 4`; whole deploy about a minute; a running session on `prod` stayed on version 3; a fresh one got version 4; no failed request |
| Roll back `prod` 4 → 3 | About 20 s end to end (the endpoint itself 7 s); a running session stayed on version 4 for at least the ~18 s observed; fresh sessions got version 3; no failed request |

What this answers: **promotion and rollback by moving a named endpoint work, are quick and create no new version; no request failed in any run; but already-running sessions are not moved, so a rollback does not rescue users who are already mid-session on a bad version**, and the first request of any new session is slow (cold start). **Not measured:** how long an old session lingers (timeouts), more than one run each, and heavier load.

**The experiment as planned (kept for reference):** add `endpoints: { "prod": { "version": <current> } }` to `s1c_agent`, deploy, call it through the page with `qualifier=prod`; change a harmless line in `main.py` and deploy (a new version); show that `DEFAULT` runs the new code and `prod` still runs the old; then move `prod`, and move it back (rollback). Record every status and any failed request.

## 8. Unknown / still to test

- **Zero downtime for users already connected.** Plan: a script that sends a request every second while dev is redeployed with a harmless change (and again while moving the named endpoint), and records failures and latency. It answers "does a deploy affect users?" with data. **[not run]**
- **Do in-flight sessions survive a runtime update?** **[unknown]**
- **Do named endpoints pinned to a version give safe promotion and rollback?** **Run once each on dev (section 7b):** yes for new sessions, no failed requests, about a minute to promote and 20 s to roll back; already-running sessions are not moved. **Still unknown:** how long such sessions linger, behaviour under heavier or concurrent load, and in-flight requests at the exact moment of the move.
- **A real second environment**: either a second AWS account (name uniqueness is per account and region) or a copy of the project under a different project name (a workaround that keeps shared Cognito and Lambdas). **[not tried]**
- How the CLI behaves when `.env.local` differs per account. **[unknown]**

## 9. Handy commands

```
# what exists
aws cloudformation list-stacks --region ap-southeast-2 --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE UPDATE_ROLLBACK_COMPLETE ROLLBACK_COMPLETE --query "StackSummaries[?contains(StackName,'AgentCore')].[StackName,StackStatus]" --output text
# why did a deploy fail?
aws cloudformation describe-stack-events --stack-name <stack> --region ap-southeast-2 --query "StackEvents[?ResourceStatus=='CREATE_FAILED' || ResourceStatus=='UPDATE_FAILED'].[Timestamp,LogicalResourceId,ResourceStatusReason]" --output text
# preview / deploy one target explicitly
agentcore deploy --target <name> --diff
agentcore deploy --target <name>
# remove an empty failed stack
aws cloudformation delete-stack --stack-name <stack> --region ap-southeast-2
```
In the console: **CloudFormation → Stacks → (stack) → Events** (filter `FAILED`), **Resources**, **Template**.

## 10. Self-test questions

1. Why did a `prod` target in the same account and region fail, and what would avoid it?
2. Why can't the same policy text be used in every environment?
3. What does `--diff` tell you, and what can it not tell you?
4. A deploy shows `UPDATE_ROLLBACK_COMPLETE`. Is the application broken? What should a script do?
5. Which parts of this project would be shared between "dev" and "prod" today, and why does that matter?
6. How would you point the same Lambda code at a dev database in DEV and a production database in PRD?
7. A policy change and a group change: which affects tokens that already exist?
8. Why must the target be named explicitly in a deploy script?

## 11. Reading (landing pages, not opened in this session)

| Topic | Link |
|---|---|
| CloudFormation user guide (stack statuses, rollback, events) | https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/ |
| AWS CDK developer guide (stacks, environments, bootstrapping) | https://docs.aws.amazon.com/cdk/v2/guide/ |
| AWS Organizations (multiple accounts) | https://docs.aws.amazon.com/organizations/ |
| AWS whitepaper "Organizing Your AWS Environment Using Multiple Accounts" (search the title) | https://aws.amazon.com/whitepapers/ |
| AgentCore developer guide (Runtime versions and endpoints, Policy) | https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/ |
