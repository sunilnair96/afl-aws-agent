# 11. AWS workshop, Lab 8: zero-code agents with AgentCore Harness (reading notes)

Read 2026-10-19 (page opened in the browser pane; it is JavaScript-rendered, so a plain fetch only returns a stub).
Source: AWS Workshop Studio, "Getting Started with Amazon Bedrock AgentCore", Lab 8, about 25 minutes:
`https://catalog.us-east-1.prod.workshops.aws/workshops/c770f35f-90a9-4e02-8985-4ef912bddb77/en-US/85-lab8-harness`
These are my notes in my own words, not a copy. **Not run by us**: everything below is what the lab says, not what we observed.
Only Lab 8 was read. Labs 3 to 7 (build the Gateway with Cedar policies) and Lab 9 ("Optimizing Agent Quality") were not.

## 1. What the lab builds

An "Order Research Agent" made only with CLI commands, no application code: a model, a system prompt and a list of tools,
all stored in one `harness.json` (`app/<name>/harness.json`). It then connects to the secured Gateway built in the earlier
labs (tools `check_warranty` and `process_refund`, guarded by Cedar), and explores shell access, per-session files,
changing the model per call, policy enforcement, and a human-approval step. The point it makes at the end: two agents side
by side (one custom Python, one zero-code) sharing one Gateway, one set of tools and one set of governance rules.

## 2. The commands, step by step

| Step | What | Command shape (placeholders in angle brackets) |
|---|---|---|
| 1 | Create the harness | `agentcore add harness --name <n> --model-provider bedrock --model-id <model> --system-prompt "<text>" --tools agentcore_code_interpreter` |
| 2a | Read the Cognito values the lab stored earlier | `aws ssm get-parameter --name /app/customersupport/agentcore/<discovery_url|client_id|pool_id|cognito_auth_scope>`; the client secret from `aws cognito-idp describe-user-pool-client` |
| 2b | Create the outbound credential provider | `agentcore add credential --type oauth --name <provider> --discovery-url <url> --client-id <id> --client-secret <secret> --scopes <scope>` |
| 2c | Add the Gateway as a tool, with OAuth | `agentcore add tool --harness <n> --type agentcore_gateway --name <t> --gateway-arn <arn> --outbound-auth oauth --provider-arn <arn> --scopes <scope> --grant-type CLIENT_CREDENTIALS` |
| 3 | Deploy | `agentcore deploy -y -v` |
| 4 | Invoke | `agentcore invoke --harness <n> --session-id <uuid> --actor-id <label> "<prompt>"` |
| 5 | Run a command inside the harness | `agentcore invoke --exec --harness <n> --session-id <same uuid> "<shell command>"` |
| 7 | Change the model for one call | `agentcore invoke ... --model-id <other model>` |
| 9 | Declare a human-approval tool | `agentcore add tool --harness <n> --type inline_function --name <t> --description "<text>" --input-schema '<json>'` then deploy |

Resulting `harness.json` after step 1: name, model (provider and id), systemPrompt, `tools` (a code-interpreter entry), empty `skills`,
and `memory.mode: disabled`. After step 2c the gateway tool gains `config.agentCoreGateway.outboundAuth.oauth` with
`providerArn`, `scopes` and `grantType: CLIENT_CREDENTIALS`.

## 3. Facts and gotchas the lab states

- **Outbound auth default is IAM (SigV4).** Against a `CUSTOM_JWT` Gateway the harness then fails at invoke time with a 401
  (`Failed to load tool ... 401 Unauthorized` on the gateway `/mcp` endpoint). OAuth outbound must be set up when the tool is added.
- **Why OAuth outbound is needed:** the harness has no inbound user token to forward, so it gets its own machine token
  (`client_credentials`) through the credential provider. (Detail and our reading of it: `04_HARNESS_VS_RUNTIME.md`, Part 1c.)
- The provider ARN can be written before the provider exists:
  `arn:aws:bedrock-agentcore:<region>:<account>:token-vault/default/oauth2credentialprovider/<provider-name>`.
- Reference the Gateway by **ARN**; the `--gateway <name>` form "may not resolve from local state".
- The harness keeps a **real filesystem per session**: files written in one invocation (for example `/tmp/report.md`) are still there
  in the next one with the same session id. A new session id starts empty. The lab shows Python 3.10 in that environment.
- **`--exec` is not a raw shell.** In the preview CLI the command runs deterministically inside the harness microVM, but the output
  comes back through the model as a short conversational summary, so it still costs tokens and may be paraphrased.
- **`--model-id` on invoke overrides the model for that call only**, with no redeploy and the session unchanged (the lab swaps to a cheaper
  Amazon Nova model to compare). Model ids are region-specific (the lab uses `us.` profiles; our region needs `au.` ones).
- **Cedar rules guard the Gateway, not the caller.** The harness inherits the same rules as a Runtime agent with no extra setup. The lab's demo:
  a $500 refund for an unhappy customer is denied by policy. The identity Cedar sees is the harness's machine client.
- **Inline functions (human in the loop):** the harness pauses and hands control back to the caller with `stopReason: "tool_use"`; the caller does
  the work (ask a person, call an API); the caller resumes with **one call that carries both messages**: the assistant's `toolUse` echoed back and the
  user's `toolResult`. The harness deliberately does not save the inline-function turn, so an unanswered pause leaves the session clean.
- **The CLI cannot resume an inline-function call yet** (it treats the answer as a new user message). The lab uses a short Python script with
  boto3: client `bedrock-agentcore`, `invoke_harness(harnessArn=..., runtimeSessionId=..., messages=[...])`, reading the event stream
  (`contentBlockStart/Delta/Stop`, `messageStop` with the stop reason) and printing text as it arrives.
- The lab names real-world approvers: a Slack reaction, a queue (SQS) with a review dashboard, another supervising agent, or a web-UI modal.
- Next pages named: a Bonus page "Advanced Harness Features" (session storage mounts, custom container images, agent skills) and Lab 9
  (optimising agent quality). Neither was read.

## 4. How it compares with what we built

| Topic | Lab | Us |
|---|---|---|
| Model | Claude Sonnet 4.6 (US profile) | Claude Haiku 4.5 (`au.` profile) |
| Tools | Code Interpreter, JWT Gateway, one inline function | IAM Gateway with five Lambda tools (`get_player_performance`, `find_players`, `get_player_game_log`, `run_sql`, `whoami`); the client-side tools harness uses inline functions |
| Memory | disabled | disabled |
| Gateway auth from the harness | machine token via OAuth credential provider | IAM (SigV4): no machine client or credential provider needed |
| Human in the loop | Python script calling `invoke_harness` | `AFLHarness/client/afl_app.py` already does the same pause and resume for our client-side tools |
| Per-user rules | none (the machine identity is all Cedar sees) | enforced by our app on the client-side path; Cedar per-user rules only on the Runtime path |
| Deploy | `agentcore deploy -y -v` | `agentcore deploy --target default --yes -v` |

Our default tool list in `quickHarness` already showed built-in `shell` and `file_operations`, which matches the lab's "full Linux environment".

## 5. What may be useful for us (ideas, not decisions)

1. **Try other models cheaply.** `--model-id` per invocation would let us rerun our test questions (the `run_sql` routing set: expected tool and
   expected answer for about 10 questions) on a cheaper or stronger model without redeploying, and report counts of correct answers and tool
   choices. Needs the right region-specific model ids.
2. **Code Interpreter (our old Option C)** is added with one flag at creation, or with `agentcore add tool --type agentcore_code_interpreter`.
   The lab's report-writing example (write a file, then read it back next turn) shows the per-session filesystem we would rely on.
3. **Human approval before an action.** The trade planner is propose-only by design. If a later version ever submits a trade, an inline-function
   "confirm trade" step (the app shows the proposal, the user approves, the app sends the result back) is the pattern. Not needed today.
4. **`--exec` for debugging**: to see what a harness session created, with the caveat that its output is summarised by the model.
5. **`--actor-id`** is passed on invoke in the lab with no explanation. It looks like a label for the person or actor behind a call; check the docs
   before relying on it (it may matter for per-user memory later).
6. **Reading list:** the "Advanced Harness Features" bonus page (session storage mounts, custom containers, skills) and Lab 9 (agent quality
   and evaluation), which fits our habit of measuring answers against known results.

## 6. Cautions

- The lab passes the Cognito client secret on the command line (`--client-secret "$COGNITO_MACHINE_SECRET"`), so it lands in shell history and
  process listings. Keep it out of chat, git and screenshots; the harness config itself holds no secret because it stores only the provider ARN.
- The lab assumes earlier labs created the SSM parameters, the machine app client and the secured Gateway; nothing here works standalone.
- It is a preview CLI: behaviours such as `--exec` and inline-function resume are described as limitations that may change.
