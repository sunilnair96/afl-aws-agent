# 07 — Harness experiment plan: "same function, built with a harness"

*Written 2026-10-09 before the work (not run). Purpose: measure whether a harness saves time, and where it stops being enough, to finish the "harness vs Runtime" paper outline in `04_HARNESS_VS_RUNTIME.md`. Credit remaining reported by the owner: $158.66 (the billing view can lag).*

## What AWS documentation says (search summaries of the pages below; not read in full)

- A harness can require a **JWT inbound** (callers log in to it).
- For a **Gateway tool**, the harness's **outbound** auth options listed are **AWS IAM** (default), **None** and **OAuth**. **No option to forward the end user's token was found.**
- `JWT_PASSTHROUGH` is a credential-provider type on **gateway targets** (HTTP and Runtime targets only), not on a harness's tool.
- Sources: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-tools.html · https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-security.html · https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-building-adding-targets-authorization.html · https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-target-http-passthrough.html
- What we saw ourselves: the harness's OAuth outbound step failed with `ResourceOauth2ReturnUrl` (snapshot in `notes/harness_snapshot/`).

## Why the experiment uses an IAM-authorised gateway (clarified 2026-10-10)

There are two separate links. **User → harness (inbound)** supports `CUSTOM_JWT` (Cognito) **or** `AWS_IAM` (`agentcore add harness --authorizer-type`), so end users do **not** need IAM roles. **Harness → Gateway (outbound)** offers `awsIam`, `none`, or `oauth` (`CLIENT_CREDENTIALS` or `USER_FEDERATION`); `awsIam` means the harness's **own service role** signs the request, which needs an `AWS_IAM` gateway. It is the simplest service credential for a quick comparison. With any of the first three options the Gateway cannot tell user A from user B (only `USER_FEDERATION` is per user, and it failed for us with `ResourceOauth2ReturnUrl`).
Optional later variant: the same harness calling the existing Cognito (JWT) gateway with `oauth` + `CLIENT_CREDENTIALS` (needs a resource server, an app client with a secret, a credential provider): shows Cognito works at the Gateway as a **service** identity, still not the user.

## Project layout decision (2026-10-10)

The harness lives in its **own project**, `AFLHarness/` (sibling of `AFLStartingTeamAgent/`), with its own `agentcore.json`, stack `AgentCore-AFLHarness-default`, and an IAM gateway; the Lambda, data and `tools.json` are shared. Reason: adding a gateway to the Strands project auto-wired every runtime (new IAM statement, new env vars, new runtime versions), and a failed update rolls back a whole stack.

## The function to rebuild

"Given a player and a season, answer with the statistics, calling the existing Lambda `afl-get-player-performance` through a Gateway." Runtime version already exists (`s1c_agent`).

## AWS's own comparison (page "AgentCore harness vs. Runtime", read in full 2026-10-10)

Source: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/harness-vs-runtime.html

- Runtime = serverless hosting; *the orchestration loop is yours*. Harness = managed loop **powered by Strands Agents**, declared as configuration; "switching a model or adding a tool is a config change, not a redeploy". **The harness is a managed abstraction that runs inside Runtime** (CloudTrail records harness operations under `AWS::BedrockAgentCore::Runtime`).
- **Harness, no code needed:** model selection and mid-session provider switching, shell/file tools, Agent Skills, observability, short- and long-term memory (incl. per-user scoping by actor ID), Gateway, Browser, Code Interpreter, remote MCP tools, context-window truncation, execution limits, streaming, outbound auth/token vault, inbound IAM and OAuth, VPC, session isolation, versioning and endpoints, environment variables, filesystems.
- **Runtime, you write code for:** model selection, memory, Gateway/Browser/Code Interpreter calls, observability, streaming, truncation, limits, outbound auth. Runtime gives infrastructure for free: isolation, inbound auth, VPC, versioning, filesystems, env vars.
- **Harness cannot:** choose the agent framework; bidirectional streaming; non-agent-loop patterns (graph/workflow). **Inline/client-side tools** still need your code. **Lifecycle hooks: Mixed.**
- A search summary of AWS docs (not the page itself) also reported: GA July 2026; harness can be exported to Strands code and run on Runtime; no separate harness charge (pay for the underlying capabilities); inbound caller authorization and prompt-injection defence are the customer's responsibility.

## Capability grid (to fill during the experiment: Strands Runtime = what we built; Harness = observed)

| Capability | Strands Runtime (ours) | Harness (observed) |
|---|---|---|
| Agent loop | our code (`Agent`) | managed |
| Model choice / switch | `MODEL_ID` in code, redeploy | config |
| Gateway tool | our `@tool` wrappers | config tool |
| Per-user identity to the Gateway | **yes** (forwards the header; injects `user_id`) | **? (no option found)** |
| Per-call logging of arguments | **yes** (`gateway_calls`) | ? |
| Packaging/dependencies | `pyproject.toml`, OpenTelemetry, drift | none |
| Memory | none built | config |
| Limits, truncation, streaming | not built | config |
| Versions and endpoints | yes (verified) | ? |
| Code written | `main.py` ~150 lines + `pyproject.toml` | **none** (`harness.json`, 35 lines, plus a one-sentence prompt) |
| Same answer for the same question | yes | **yes** (25 games, 108.88, 147, 78, 19.49) |
| **Does the Gateway apply a per-user rule?** (user B asks for 2023) | **denied** by Cedar | **answered in full**: `--user-id userB` is just a label; the Gateway sees only the harness's role (observed) |
| **What the tool Lambda sees about the caller** | gateway metadata only (no user) | the **same** gateway metadata only; same for userA and userB; no IAM identity either (observed) |
| New tool added to the gateway | n/a | picked up by a new session, no harness redeploy (observed) |
| Latency (first call / warm / new session) | not measured like-for-like | 41.4 s (cold) / 5.9 s / 5.7 s |
| Tool-call visibility for the caller | `gateway_calls` (tool, arguments, outcome) | final text only in the CLI. **Checked 2026-10-10 in CloudWatch GenAI Observability (`clientToolHarness`):** per-trace spans (`invoke_agent`, loop cycles, `execute_tool <name>`, `chat`), model id, input/output tokens, cost, and the prompt/response JSON. So prediction 3 (less detail than `gateway_calls`) was **not** borne out for model and tool spans; whether the tool *arguments and result* appear in the span JSON was not opened. |
| Lifecycle hooks | n/a (our own code) | `before/after_tool_call`, `before/after_invocation`; Lambda can allow/deny, SNS/EventBridge notify; payload has **no caller identity**, no modifying of tool input (AWS docs; **not run**) |
| Deploys for the agent | several | 2 (gateway first, then the harness), 0 failed |
| Role breadth (generated) | broad | broad (`InvokeGateway` on `gateway/*`, whole token vault, any model) |
| **Option C: harness + client-side tools, user's token to the JWT Gateway** | n/a | **Same A/B result as Runtime** (A 2023 answered, B 2023 denied by Cedar, B 2025 allowed, `whoami` = own sub, injection ignored), observed 2026-10-10. The token and the `user_id` binding live in the **client app**, not the harness. |
| Code written for Option C | n/a | `harness.json` (about 60 lines, two inline tools) + `harness_client.py` (about 140 lines); no packaging or agent code |
| Inbound auth (user → agent) | Cognito JWT, `allowedClients` (verified) | **OAuth and IAM supported** (AWS grid); our first harness used `CUSTOM_JWT`; custom-claim checks **?** |
| Outbound auth to the Gateway | we forward the user's token | CLI options: `awsIam`, `none`, `oauth` (`CLIENT_CREDENTIALS` / `USER_FEDERATION`); schema for runtime connections also lists `TOKEN_EXCHANGE`: **?** (Cognito support unknown) |

## Method

1. **Second Gateway with IAM inbound authorisation** (so a harness can call it with its role). Same Lambda target and tool schema. Do not touch `AFLGatewayJWT`.
2. **A harness** (config only: model, short system prompt, one Gateway tool with `outboundAuth: awsIam`), built from `harness_snapshot` minus the OAuth parts. Memory: **off**.
3. Invoke it the way users would (a CLI call first).
4. Record, for **both** harness and Runtime, **counts and capabilities, not wall-clock time** (the owner takes breaks, so time is not comparable): number of commands, deploys, files and lines written, errors met and how they were diagnosed, and what the console/logs let you see; then fill the capability grid in the section above.
6. **Variant to try:** harness **lifecycle hooks** ("validate invocations and tool calls"; the AWS grid marks them *Mixed*): can a hook see who the end user is, and could it check or inject a `user_id` the way our agent code does? **[unknown]**
5. Rerun the A/B authorisation question on the harness.

## Predictions (written before running)

| # | Prediction |
|---|---|
| 1 | The harness needs **fewer files, lines and moving parts** than the Runtime did (no packaging, no dependency file, no agent code), though it needs **two deploys** (the Gateway ARN must exist before the harness can name it) |
| 2 | The Cedar rules **cannot** tell user A from B: the Gateway sees the harness's IAM role, not the person, so "B may not see 2023" cannot be enforced there |
| 3 | The harness logs show less detail about each tool call than our own agent's `gateway_calls` log |
| 4 | Idle cost of a harness with memory off is near zero |

## Success / outputs

- A table of measured differences (counts of files/lines/commands/deploys, visibility, access control) and the filled capability grid.
- One decision sentence the evidence supports, for the paper and a post.
- Everything removed afterwards (`agentcore remove harness`, `remove gateway`, then deploy) so no idle cost remains.

## Risks and cautions

- A second gateway means a second policy engine/target set in the same stack: **names must not collide** with the existing ones (lesson from the environments exercise). Use distinct names.
- Check `agentcore deploy --diff` before every deploy and list the files changed before running anything (agreed working rule).
- Do not weaken `AFLGatewayJWT`. The IAM gateway is experimental and should have no data beyond what the existing tool returns.
