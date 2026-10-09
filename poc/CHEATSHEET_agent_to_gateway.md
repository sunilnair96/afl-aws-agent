# Cheat sheet — getting an agent to call a Gateway *as the user*

*Written 2026-10-08 after the S1 spike. Everything marked ✔ was run and observed in this account. Anything marked "reading" is my interpretation, not proof.*

## 1. The 2-minute explanation (read this first)

Think of a building with a security desk.

- **JWT (access token)** = your visitor badge. Cognito prints it after you log in. It says *who you are* (`sub`), *which app issued it for* (`client_id`) and *when it expires*. It is signed, not secret-by-encryption: anyone can read it, nobody can forge it.
- **Gateway** = the security desk at the tool door. It checks the badge is real and not expired.
- **Cedar policy** = the rulebook behind the desk: "badge X may use tool Y only when season ≥ 2025". No matching rule = **denied** (default deny).
- **Lambda** = the room behind the door. It never sees the badge unless we pass something in.
- **Agent** = a helper who goes to the desk on your behalf.

The question this spike answered: **when the agent walks to the desk, does it show *your* badge?**

| | How the agent gets a badge for the desk | Result |
|---|---|---|
| **Managed harness** (what I started with) | Configured an *outbound* OAuth step (user federation) that must fetch a badge through a consent flow and needs a **return URL** | ✘ failed: `You must provide a ResourceOauth2ReturnUrl` |
| **Runtime agent with header passthrough** (the fix) | The Runtime checks your badge, hands the `Authorization` header to the agent code, and the code **forwards the same badge** | ✔ worked |

Reading (not proven): the harness is designed to call *third-party* services on a user's behalf, so it obtains a separate outbound token; our case (same user, same identity provider) only needs the original token passed along.

## 2. The fix in 6 steps

1. **Inbound check on the Runtime**: `authorizerType: CUSTOM_JWT` with the Cognito discovery URL and allowed client. A request without a valid token never reaches the code.
2. **Allow the header through**: `requestHeaderAllowlist: ["Authorization"]`. Without this the code never sees the token. (Only works together with a JWT authorizer.)
3. **Read it in code**: `context.request_headers` in the `@app.entrypoint` handler.
4. **Forward it unchanged** to the Gateway: `"Authorization": <same value>` on the MCP request. Do not mint or fetch a new token.
5. **Gateway validates + Cedar decides**: allow → Lambda runs; deny → JSON-RPC error `-32002`, Lambda never runs.
6. **Never print the token.** Return only a few decoded, non-secret claims while testing.

Minimal code that proved it (`app/s1a_echo/main.py`):

```python
@app.entrypoint
def handler(payload, context: RequestContext):
    headers = context.request_headers or {}
    auth = next((v for k, v in headers.items() if k.lower() == "authorization"), None)
    ...
    request = urllib.request.Request(GATEWAY_URL, data=..., headers={"Authorization": auth, ...})
```

## 3. Evidence table (all ✔ observed, user B `<USER_B_SUB>`)

| Check | Observed |
|---|---|
| Access token claims | `sub`, `client_id`, `token_use=access`, `scope`; **no** `cognito:groups`, **no** email/name |
| Token lifetime | `exp − iat` = 3599 s (~1 h) |
| Runtime passes header to code | `Authorization`, `baggage`, `workloadaccesstoken` received; claims identical to the token |
| `tools/list` via agent | HTTP 200, one tool |
| Season 2025 via agent | allowed; same data as curl (25 games, avg 108.88, stdev 19.49) |
| Season 2023 via agent | `error.code -32002`, "No policy applies to the request (denied by default)" |
| Lambda invoked for denied call? | **No**: CloudWatch showed exactly one invocation (the 2025 call) |
| Denial HTTP status | **200** (the refusal is inside the JSON body) |

## 4. Errors met on the way, and the fix

| Symptom | Cause | Fix |
|---|---|---|
| `aws kms list-aliases --query ...` → "expected one argument" | Windows `cmd` mangled quotes/`!` in the query | Drop `--query`, use `--output table` |
| Harness: `You must provide a ResourceOauth2ReturnUrl` | Outbound OAuth user-federation step in the harness needs a return/consent flow | Don't use it for this: use a Runtime agent and forward the header |
| `cognito_test.py`: "Token exchange failed" | Auth code is **single-use**; or stale script still on port 8080; or wrong client secret | Close old windows, check `netstat -ano \| findstr :8080`, set secret in the *same* window, sign in once, print the error body (`e.read()`) |
| Script shows nothing in terminal | Python buffers output when not a plain console | `python -u` (unbuffered) or Ctrl+C to flush |
| `agentcore add agent`: `--framework is required`, then `--model-provider is required` | CLI requires both even for bring-your-own code | `--framework Strands --model-provider Bedrock` |
| Deploy: Runtime `CREATE_FAILED … OpenTelemetry instrumentation executable not found` | The zip had no `aws-opentelemetry-distro` | Add it to **`pyproject.toml`** |
| Edited `requirements.txt`, same failure | The CLI installs from **`pyproject.toml`** (`uv pip install -r pyproject.toml`, searched upward), **not** `requirements.txt` | Give the app its own `pyproject.toml` with `bedrock-agentcore` + `aws-opentelemetry-distro` |
| Deploy log full of `STS AssumeRole AccessDenied` | CDK first tries its bootstrap roles, then falls back to your user. Normal noise | Ignore; search the log for `CREATE_FAILED` / `Reason:` instead |
| Stack shows `UPDATE_ROLLBACK_COMPLETE` | CloudFormation automatically undid the failed create | Nothing to clean; fix and redeploy |
| Gateway returns 200 but no data | Denials are JSON-RPC errors in a 200 response | Check `error` in the body, not the HTTP status |
| 401 after an hour | Access token expired | Sign in again |

## 5. Commands used (what / why)

| Command | What it does |
|---|---|
| `agentcore status` | Lists what is deployed (agents, gateways, policy engines, harness) |
| `agentcore add agent --type byo ... --authorizer-type CUSTOM_JWT --request-header-allowlist Authorization` | Registers a bring-your-own-code Runtime with JWT inbound auth and the header allowlist (config only, nothing deployed) |
| `agentcore validate` | Checks `agentcore.json` is valid |
| `agentcore deploy` | Packages code, runs CloudFormation, creates/updates resources |
| `agentcore invoke --runtime NAME --bearer-token %TOKEN% --prompt "..."` | Calls the deployed runtime with a user's token |
| `aws logs tail /aws/lambda/FUNCTION --since 30m` | Shows recent Lambda logs (proof of invocation or not) |
| `set TOKEN=...` (cmd) | Stores the token in this window only |

## 6. Rules to remember

1. Authentication ≠ authorization. Cognito says *who*; Cedar says *what*.
2. The LLM never supplies identity; code takes it from the verified token.
3. Check denial in the **body**, not the HTTP status.
4. Default deny: no permit = deny.
5. Keep tokens and secrets out of chat, git, screenshots and posts.
6. One failed deploy is information: read `Reason:` in the log, not the whole log.

## 7. Not proven yet (don't claim these)

- Group-based policy (`cognito:groups`) instead of one hard-coded user (P2).
- What the Lambda itself knows about the caller (the `whoami` tool, P3).
- Interceptor overwriting a forged `user_id` (P3).
- Behaviour with a language model in the loop and a prompt-injection attempt (S1c).
- *Why exactly* the harness path fails: I only know the error and its configuration; the explanation above is my reading.
