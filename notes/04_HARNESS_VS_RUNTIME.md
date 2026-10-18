# 04 — Harness vs Runtime (comparison today, and the paper still to write)

*Written 2026-10-09. Part 1 is what we know from this project. Part 2 is the outline of a decision paper to finish **after** the harness is rebuilt as a deliberate experiment. Things I did not verify are marked **[unverified]** or **[unknown]**.*

## Part 1 — The comparison, from our own project

**One line:** a *harness* is a **configured** agent that AWS runs for you; a *Runtime* is **hosting** for agent code **you** write.

| | Harness (`teamHarness`) | Runtime (`s1a_echo`, `s1c_agent`) |
|---|---|---|
| What you provide | A config file: model, system prompt, tools, memory, limits (`harness.json`, `system-prompt.md`) | Your own code (`main.py` + `pyproject.toml`) with any framework |
| Who runs the agent loop | AWS | You (through the framework; we used Strands in `s1c_agent`) |
| Adding a tool | `agentcore add tool --harness …` | A function in your code |
| Model | In config (`global.anthropic.claude-sonnet-4-6`) | In code (`MODEL_ID`; we used `au.anthropic.claude-haiku-4-5-20251001-v1:0`) |
| Memory | A setting (`"memory": {"mode": "managed"}`) | Added separately if wanted |
| Inbound login | `authorizerType: CUSTOM_JWT` in config | `CUSTOM_JWT` in `agentcore.json` + `requestHeaderAllowlist` |
| Calling the Gateway as the user | A configured **outbound** OAuth step (`outboundAuth`, `USER_FEDERATION`) - failed for us with `ResourceOauth2ReturnUrl` | Your code forwards the `Authorization` header it received |
| Control over identity logic | What the config offers | Full: we injected `user_id`, logged each call, added an unsafe test tool |
| Effort | Low | Higher (we hit `pyproject.toml` vs `requirements.txt`, and the OpenTelemetry package) |
| Idle cost seen | **None attributable.** I first guessed that harness memory created a KMS key and a knowledge-base charge; that was **wrong** (see "Correction" below) | None identified |

**Why it mattered to us:** the question was how the user's identity gets from the agent to the Gateway. In the harness that hop is a *setting* we could not make work; in the Runtime it is *code* we could write, test and log.

**Analogy:** a meal kit versus renting a kitchen.

**What we do not know**
- ~~[unknown] How the harness is built internally~~ **Resolved 2026-10-10:** AWS's *harness vs. Runtime* page says the harness is a managed abstraction **that runs inside Runtime**, with the loop powered by Strands Agents (see `07_HARNESS_EXPERIMENT_PLAN.md` for the full feature grid).
- **[unknown]** Whether some harness setting makes the outbound OAuth step work. Our bug report says it failed even with `defaultReturnUrl` set.
- **[unverified]** That the outbound step is *the* mechanism for calling a Gateway. This is read from `harness.json` and the error text, not from AWS documentation.

## Part 1b: result of the harness rebuild (2026-10-10, observed)

Same function built three ways. The access-control row is the one that decides.

| | Strands on Runtime | Harness + IAM gateway tool (`quickHarness`) | Harness + inline tools + client app (`clientToolHarness`) |
|---|---|---|---|
| Who runs the tool | our agent code | the harness, with its own IAM role | our client app, with the **user's** token |
| B asks for 2023 | denied by Cedar | **answered in full** | denied by Cedar |
| What the Lambda sees of the caller | `user_id` bound from token | nothing (A and B identical) | `user_id` bound from token |
| Code you write | `main.py` + `pyproject.toml` | none (`harness.json`) | `harness.json` + client app (about the same lines as `main.py`) |
| Packaging and dependency drift | yes | none | none (in the app, a normal dependency) |
| Console playground end to end | n/a | yes | **no**: stops at `tool_use` |

**One-line rule (working, not final):** choose a harness when the tools are standard and no per-user rule must reach the data; when it must, either use a Runtime or keep the harness and put the user's token and the tool execution in your own app. The saving from the harness is the undifferentiated work (loop, memory, limits, packaging), not the lines of code.

**Flaws found in the first draft of this conclusion:** where the app runs does not change the harness benefit (it runs in AWS either way); the lock-in is mostly the Gateway/Cedar/Cognito ties, which exist in both designs; changing the model is also a one-line change on Runtime (the harness saves the redeploy); AWS's own page lists inline tools as "your code" but does not say that this is the route to per-user access control: that is our observation.

## Part 1c: can the harness pass the user's identity to the Gateway? (2026-10-19, from AWS material, not tested by us)

**Short answer.** No. A harness can authenticate its callers at its own door, but towards a JWT Gateway it presents a
**machine token**, not the user's. The Gateway and Cedar therefore see the app, never the person. Per-user rules must be
enforced by the application (or by a Runtime agent that forwards the user's token).

**Evidence, in the order we met it**
1. AWS sample `awslabs/agentcore-samples`, `01-features/01-harness/01-advanced-examples/07-oauth` (README read, scripts not run):
   - inbound: the harness has a `CUSTOM_JWT` authorizer on a Cognito user pool, so an invalid caller token is a 401 before the agent runs;
   - outbound: `outboundAuth.oauth` makes the harness fetch a machine-to-machine token (`client_credentials`) from a **second** Cognito pool on every tool call; the Gateway checks that token's client and scopes, then the Lambda runs;
   - the README does not say the user's token is forwarded and shows no way for a tool or policy to learn which user called.
2. An AWS lab (name not recorded; text pasted by Sunil) says it outright: in the Runtime lab the agent authenticated to the `CUSTOM_JWT` Gateway by
   forwarding the caller's Cognito bearer token (token passthrough); "the zero-code harness has no inbound user token to forward, so it must obtain
   its own Cognito token using a machine-to-machine (client_credentials) OAuth credential provider", configured as the gateway tool's outbound auth.
3. The lab also says that if you skip the outbound auth, `agentcore add tool --type agentcore_gateway` defaults to **IAM (SigV4)**, which a JWT Gateway
   rejects: the harness then fails at invoke time with `Failed to load tool ... 401 Unauthorized` on the gateway `/mcp` endpoint.

**How the machine-token wiring works (the chain, for when it is needed)**
1. A **machine app client** in Cognito: client-credentials grant, with a resource server and a scope.
2. A **credential provider** in the AgentCore token vault (`oauth2credentialprovider/<name>`) that holds that client's ID and secret, so the harness
   configuration itself contains **no secret**. Its ARN is predictable: `arn:aws:bedrock-agentcore:<region>:<account>:token-vault/default/oauth2credentialprovider/<name>`.
3. The harness tool points at the provider and the scope:
   `agentcore add tool --harness <harness> --type agentcore_gateway --name <tool> --gateway-arn <arn> --outbound-auth oauth --provider-arn <provider arn> --scopes <scope> --grant-type CLIENT_CREDENTIALS`
4. The JWT Gateway validates the machine token (`allowedClients` / scopes) and Cedar decides for the **machine principal**.
   Read the client secret only into a variable (never print it, never put it in chat, git or screenshots).

**What this means for our design**
- Our harness uses the IAM Gateway `AFLGatewayIAM` (SigV4), so it never needed a machine client or a credential provider. That is the correct pairing:
  IAM gateway for the harness, JWT + Cedar gateway for the Runtime path.
- **Prediction (not tested):** pointing our harness at our JWT Gateway would be denied everything by Cedar's default deny, because our policies are written for
  a human user (`cognito:groups` tag; `sub` equal to the `user_id` argument), while a machine token has no groups and its `sub` is the client ID. It would need
  a policy for the machine principal, and that policy could not tell users A and B apart.
- So the earlier wording "we did not find any way to forward the user's token" (TRACE_LOG) is now backed by AWS's own statement. It is a statement about the
  harness as it is today; re-check if the harness gains a token-forwarding option.
- The Part 2 hypothesis row "Per-user identity handling, custom checks, token forwarding: lean towards Runtime" still stands, now with this evidence.

**If we ever want to try it** (a learning exercise, not part of the AFL app): in a separate folder with its own project name, add a machine client with a scope
to a Cognito pool, create the credential provider, add the tool as above, and add a Cedar policy for the machine principal. Do not touch the existing pool,
Gateway or policies.

## Correction: where the idle costs actually come from (checked 2026-10-09)

- The customer-managed KMS key `<KMS_KEY_ID>` belongs to the **`CDKToolkit` stack** (`FileAssetsBucketEncryptionKey`): the CDK bootstrap key for the bucket `agentcore deploy` uploads code to. It has no grants or tags and was created on the first deploy (bootstrap). **It is not the harness's**, and removing the harness did not remove it. Deleting it would break deploys. Cost is roughly $1/month **[general KMS pricing, not checked today]**.
- After removing the harness, `list-memories` is empty, so its managed memory is gone.
- The "Managed Knowledge Base" billing line is **more likely from the two knowledge bases in the account** (`quantum-computing-nist`, `afl-fantasy-knowledge-base`, both ACTIVE) than from AgentCore **[inference; their storage type was not checked]**.
- Lesson: timing coincidence is not causation. Confirm with ownership data (stack resources, grants, tags) before attributing a cost.

## Which framework does the Runtime use?

The Runtime itself is framework-neutral. The CLI offered: `Strands`, `LangChain_LangGraph`, `GoogleADK`, `OpenAIAgents`, `VercelAI`, `BedrockManagedAgents`. In this project:
- `s1c_agent` uses **Strands Agents** (`strands-agents` in `pyproject.toml`; `Agent`, `tool`, `BedrockModel` in `main.py`). It runs the loop: sends the prompt to the model, lets the model choose a tool, calls it, returns the answer.
- `s1a_echo` uses **no framework and no model**; it only echoes headers and calls the Gateway.
- The `--framework Strands` flag on `agentcore add agent` is a label the CLI required; the real choice is made by the imports in your code.

## Part 2 — The paper to write: "When to use a harness, when to use a Runtime"

**Plan:** rebuild the harness later as a deliberate experiment (`harness_snapshot/` holds the old config, prompt and bug report), run the same tests as the Runtime, then fill this in with *measured* facts.

**Questions to answer by experiment**

| # | Question | How to find out |
|---|---|---|
| 1 | How long from nothing to a working agent? | Time both from a clean start |
| 2 | What do you get for free in the harness (memory, built-in tools, skills, session handling)? | List config options; try each; compare what Runtime needed in code |
| 3 | Can a harness call a Gateway **as the user**? Under which settings? | Retest the outbound OAuth step; try other `grantType` values; record every error |
| 4 | Can a harness do identity binding like our `user_id` injection? | Try with only config; if not, note what would be missing |
| 5 | Observability: what can you see (traces, logs, per-call arguments)? | Compare CloudWatch/AgentCore traces for the same prompts |
| 6 | Cost, idle and per-call | Cost Explorer by usage type, same workload on both |
| 7 | Limits: tokens, iterations, timeouts, payloads, models | Read docs; trigger limits |
| 8 | How are changes deployed and rolled back? | Change the prompt/model on each; measure time and risk |
| 9 | Testability: can you run the attack test in CI against it? | Reuse `s1c_attack_check.py` |
| 10 | Where is the lock-in and how hard is migrating between them? | Port the same agent both ways |

**Draft decision checklist (to confirm or overturn with evidence)**

| If you need… | Lean towards… (hypothesis) |
|---|---|
| Fast standard assistant, ordinary tools, no custom logic | Harness |
| Per-user identity handling, custom checks, token forwarding | Runtime |
| A specific framework or multi-agent orchestration | Runtime |
| Minimal code to maintain | Harness |
| Fine-grained logging of exactly what the model asked for | Runtime |
| A quick demo or prototype | Harness, then migrate if it hits a wall |

**Format suggestion:** a 2-page PDF: one comparison table, one decision flow, the measured numbers, and the ten experiments' results; plus a LinkedIn post with the one-sentence rule you end up trusting.
