# 01 — Concepts (what each piece is, and where it lives in *your* project)

*Written 2026-10-08 after the POC. Read this first, then `02_HOW_WE_TESTED.md`, then `03_KEY_LESSONS_AND_READING.md`.*

## The one picture to hold in your head

```
 user ──sign in──▶ Cognito ──JWT──▶ Runtime agent ──same JWT──▶ Gateway ──▶ Cedar policy ──▶ Lambda ──▶ data
 (A or B)          (who are you?)   (code you wrote)           (checks    (may this user   (does the
                                                               the badge)  call this tool    work, never sees
                                                                           with these        the badge)
                                                                           arguments?)
```

Everything we did was about one question: **at each arrow, what identity information exists, and who is allowed to decide?**

## Glossary, in the order you meet things

| Concept | Plain meaning | In your project |
|---|---|---|
| **Cognito user pool** | A directory of users plus a login page | Pool `<USER_POOL_ID>`; users A `<USER_A_SUB>`, B `<USER_B_SUB>` |
| **App client** | An application allowed to ask the pool to log people in | Client `<CONF_CLIENT_ID>` (has a secret, so it is a "confidential" client) |
| **Group** | A named label attached to users | `full-history`, `recent-only`. Membership appears in the token as `cognito:groups` |
| **OAuth 2.0 authorization-code flow** | Login happens on Cognito's page; your app gets a one-time **code**, then swaps it (plus its secret) for tokens | What `cognito_test.py` does: listens on `localhost:8080/callback`, exchanges the code |
| **JWT (access token)** | A signed text badge: `header.payload.signature`. Readable by anyone, forgeable by no one | Claims seen: `sub`, `client_id`, `token_use`, `scope`, `exp`, `cognito:groups` (after groups) |
| **`sub`** | The user's permanent unique id | Equals the Cognito Username in your pool |
| **Token lifetime** | A token is a **snapshot** valid ~1 hour (`exp − iat` = 3599 s) | Group changes show up only in tokens issued afterwards |
| **MCP** | A protocol for agents to discover and call tools over HTTP, using JSON-RPC messages | The gateway's `/mcp` endpoint: `tools/list`, `tools/call` |
| **JSON-RPC error codes** | Numbers in the range −32000…−32099 are left to each server to define | `-32002` is used for *both* a policy denial and an auth failure here |
| **AgentCore Gateway** | A managed front door that turns Lambdas into MCP tools and checks the caller | `AFLGatewayJWT` (inbound auth `CUSTOM_JWT`) |
| **Gateway target** | One backend (here a Lambda) plus its tool schema | `PlayerPerformanceTargetJWT`, `WhoamiTarget` |
| **Tool name** | `<target name>___<tool name>` (three underscores) | `PlayerPerformanceTargetJWT___get_player_performance` |
| **Tool schema (`tools.json`)** | The contract the agent/model sees: name, description, input fields | `tools.json`, `lambda/whoami/tools.json` |
| **Cedar** | A policy language: `permit(principal, action, resource) when {…}` | Policies in `agentcore.json` under `policyEngines` |
| **Policy engine** | A container of Cedar policies attached to a gateway | `AFLAuthorizationPolicy` (3 policies) |
| **Default deny** | No matching `permit` ⇒ denied. A `forbid` beats any permit | Why a brand-new tool was blocked until we added a permit |
| **Principal tags** | The token's claims, readable in Cedar via `principal.hasTag("x")` / `getTag("x")` | `cognito:groups` confirmed; `sub` not yet tested |
| **Policy `enforcementMode`** | `ACTIVE` (decides) or `LOG_ONLY` (logs only, never permits) | All ours are `ACTIVE` |
| **Gateway `mode`** | `ENFORCE` or log-only for the whole gateway | `ENFORCE` |
| **Lambda** | A function AWS runs on demand | `afl-get-player-performance`, `afl-whoami` |
| **Lambda `event` / `context`** | Input data / invocation info. The gateway puts tool arguments in `event`, metadata in `context.client_context.custom` | Seen with `whoami`: no user identity at all |
| **IAM role: trust vs permissions** | *Trust policy* = who may become this role. *Permissions* = what it may do | `afl-whoami-role`: trusted by Lambda; allowed to write logs only |
| **Runtime (AgentCore)** | Hosting for your own agent code, behind its own login check | `s1a_echo` |
| **Strands Agents** | An open-source Python SDK that runs the agent loop (prompt → model → tool call → answer). The Runtime is framework-neutral | `s1c_agent` uses it (`Agent`, `tool`, `BedrockModel`); `s1a_echo` uses no framework |
| **Header allow-list** | Which request headers the Runtime hands to your code | `Authorization` (needs a JWT inbound authorizer) |
| **Harness** | AgentCore's configuration-driven managed agent | `teamHarness`; its outbound OAuth step failed (return-URL error) |
| **Interceptor** | A Lambda that runs before/after each gateway call and may rewrite or block it | Documented; **not** available through the `agentcore` CLI 0.31.1 |
| **CloudFormation stack / CDK** | AgentCore CLI turns `agentcore.json` into one CloudFormation stack and deploys it | Stack `AgentCore-AFLStartingTeamAgent-default`. A failed resource **rolls the whole update back** |
| **Idle vs usage cost** | Some resources bill by use, others by existing | A customer-managed KMS key costs money while it exists: ours is the **CDK bootstrap key** (`CDKToolkit` stack), needed for deploys. Knowledge bases can also accrue idle cost depending on how they store vectors |

## Where the access rules live

```
AFLStartingTeamAgent/agentcore/agentcore.json
  agentCoreGateways[AFLGatewayJWT]
      authorizerType CUSTOM_JWT + allowedClients      ← "is the badge genuine?"
      policyEngineConfiguration {AFLAuthorizationPolicy, ENFORCE}
      targets [PlayerPerformanceTargetJWT, WhoamiTarget]
  policyEngines[AFLAuthorizationPolicy].policies
      FullHistoryAnySeason     group full-history → any season
      RecentOnlyFrom2025       group recent-only  → season >= 2025
      WhoamiForGroupMembers    either group       → whoami tool
  runtimes[s1a_echo]           CUSTOM_JWT + requestHeaderAllowlist [Authorization]
```

## Who decides what (the separation the architecture diagram teaches)

| Layer | Decides | Never decides |
|---|---|---|
| Cognito | *Who you are* (and your groups) | What you may do |
| Gateway authorizer | Is the token genuine, unexpired, from an allowed client | Business rules |
| Cedar | May this user call this tool with these arguments | Anything about the data itself |
| Lambda | The work | Who the caller is (it cannot know, see lesson 4) |
| IAM | What AWS resources the Lambda/Gateway roles can touch | Which end user is calling |
| The model | What to say and which tool to try | Any access decision |

## Two things people commonly assume that were **not** true here

1. *"The agent just passes the user along."* A managed harness did **not**: its outbound call is a separate OAuth step. A Runtime agent can forward the token only because you allow-list the header and write the forwarding.
2. *"The tool knows who called it."* A gateway Lambda receives only the tool arguments and gateway metadata. Identity must be bound some other way (policy equality check, interceptor, or the agent code injecting it).
