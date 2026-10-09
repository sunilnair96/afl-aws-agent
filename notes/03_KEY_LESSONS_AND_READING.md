# 03 — Key lessons, self-test questions and reading list

## The ten lessons

1. **Authentication is not authorization.** Cognito says who you are; Cedar says what you may do. Being signed in and in the right group still got A *nothing* until a permit existed.
2. **Default deny is your friend.** No matching permit ⇒ denied, for users and for brand-new tools. Add a `permit` deliberately; do not hunt for something to block.
3. **A token is a snapshot.** Group changes appear only in tokens issued afterwards, and removing someone doesn't take effect until their old token expires (~1 hour here). Access is not revoked instantly.
4. **A gateway tool cannot see the user.** The Lambda got only the arguments and gateway metadata. Anything identity-like in the arguments is whatever the caller supplied, including the model. Bind identity somewhere that has seen the validated token.
5. **The model must never be the source of identity or permissions.** The `user_id` it sends is untrusted input. Enforce rules in policy or code that the model can't edit.
6. **Forward the user's token; don't fetch a second one.** When user and identity provider are the same, a Runtime agent that forwards the `Authorization` header is simpler than the harness's outbound OAuth step. It needs a JWT authorizer and the header allow-list.
7. **A denial is not an HTTP error.** The gateway returned HTTP 200 with a JSON-RPC error in the body. Check the body. Don't trust the error number alone: `-32002` also appeared on a 403 auth failure.
8. **One project is one stack, and a failed resource rolls everything back.** Plan for the blast radius. Things created outside `agentcore.json` (Lambdas, roles, Cognito groups) are unmanaged and can drift.
9. **Least privilege per component.** `afl-whoami-role` can write logs and nothing else. Trust policy (who may assume) and permissions policy (what it may do) are different things.
10. **Test like an engineer.** Isolate, predict, run the negative case, change one variable, collect evidence from a second source, keep the test as a script.

11. **Policy can make an untrusted argument trustworthy.** `principal.getTag("sub") == context.input.user_id` forces the claimed id to equal the verified caller, so the Lambda can rely on it. The model and the agent still have to *send* the right id, and the agent code (not the model) should add it.
12. **Permits are additive: tightening means removing, not adding.** A strict permit next to a broad one changes nothing. Removing the broad one is the change that matters. Test both stages.
13. **Verify which identity a test really used.** A reused login session made "user B" be user A. Print a safe fingerprint (first 8 characters of `sub`) in every test.
14. **A model's refusal is not a control.** In the attack run the model sometimes declined by itself and sometimes tried; only the policy's refusal was guaranteed. Test with prompts the model *does* follow, and credit the layer that actually stopped it.
15. **Layer the defence: code first, policy second.** The code keeps the model from choosing `user_id`; the Gateway policy refuses a wrong one anyway.
16. **Listed is not available.** A model with ACTIVE profiles in the catalog can still be denied to your account. Only a call proves access. Likewise verify who *owns* a cost (stack resources, grants, tags) instead of inferring from timing.
17. **A browser cannot keep a secret.** Use a public client with PKCE for page-based login; keep the secret-holding client for server-side code.
18. **Every layer has its own allow-list and its own error.** A genuine token still fails if its client isn't allowed (`-32001`), and a Cognito client created from the CLI may have no login page until a branding style exists. Build tools that tell these failures apart.

19. **A preview is not a promise.** `--diff` can't see name clashes or the service's own validation. Test in the lowest environment first, and expect the real deploy to teach you more.
20. **A rolled-back deploy can look like success.** The old version keeps working, so tests pass. Always check the final stack status (and the stack events for the reason), not just whether the app still answers.
21. **Environments are separated by account (or region), by configuration and by data location, not by renaming things.** Same code, different config; Lambda code reads its data location from configuration. Check where a shared resource hides (hand-made Lambdas, one Cognito pool, account-level providers) before calling something "prod".
22. **Put a positive control in every check.** A script that prints `False` when it read the wrong property looks identical to a real negative result.
23. **Promote by moving a name, not by redeploying code.** A runtime update creates an immutable version; a named endpoint stays pinned to its version until you move it. Promotion and rollback are a one-number change (about a minute, 20 s) with no new build. Keep `DEFAULT` for development.
24. **A rollback does not reach sessions already running.** Sessions that started on the old version keep running it; only new sessions get the moved version. Plan for that (shorter session timeouts, or a way to end bad sessions), and expect a cold start of roughly 6-9 s on the first request of every new session.

## Self-test questions (answer without looking)

1. Why was User A denied everything when A was in `full-history` but the only policy named B's user id?
2. What happens to B's access if you remove B from `recent-only` at 10:00 and B's token was issued at 09:30?
3. The Lambda `afl-whoami` returned `{"user_id": "someone-else"}`. Why can't it tell whether that is true?
4. Why does `FullHistoryAnySeason` use `like "*full-history*"`, and what is the weakness?
5. What is the difference between HTTP 401, HTTP 403 `insufficient_scope` and `-32002 Tool Execution Denied`?
6. Why did two deploys fail with the OpenTelemetry error, and where must dependencies be declared?
7. What does `policyEngineConfiguration.mode: ENFORCE` control, versus a policy's `enforcementMode`?
8. Why is "the Lambda didn't run" proven by CloudWatch and not by the gateway reply?
9. In what order should you add a new tool and its Cedar permit, and why?
10. Name two ways to give a tool a trustworthy user id, and one drawback of each.

*(Answers are in `01_CONCEPTS.md`, `02_HOW_WE_TESTED.md` and `poc/TRACE_LOG.md`. Questions 9–10 are previews of the next stage.)*

## Hands-on exercises (low cost, each under 30 minutes)

- Put B in both groups with a **fresh** token, then run the matrix before and after the fresh token. Write down the revocation delay in your own words.
- Decode both tokens (A and B) and compare every claim side by side.
- Change a policy to `LOG_ONLY`, redeploy, and observe what each user gets. Then put it back.
- Add a deliberately wrong tool name in a Cedar policy and read the validation error.
- Read `afl-whoami`'s CloudWatch log lines after a call and find the request id from the reply.

## Reading list

**Opened and used this session** (read these first):

| Topic | Link | Why |
|---|---|---|
| Passing headers (incl. `Authorization`) to a Runtime agent | https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-header-allowlist.html | The reason S1 worked |
| Gateway interceptors (overview, security) | https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-interceptors.html | Next stage; note the `passRequestHeaders` warning |
| Interceptor payload formats | https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-interceptors-types.html | Exact input and output JSON |
| Interceptor configuration | https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-interceptors-configuration.html | Shows it is set with `aws … update-gateway`, not the CLI |
| Interceptor examples | https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-interceptors-examples.html | Pass-through sample |
| Interceptor permissions | https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-interceptors-permissions.html | Gateway role needs `lambda:InvokeFunction` |
| AWS deploy plugin (README) | https://github.com/awslabs/agent-plugins | What the plugin does before you install it |

**Standard references (landing pages, not opened in this session; search inside them):**

| Topic | Link |
|---|---|
| AgentCore developer guide (Gateway, Policy, Runtime, Identity) | https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/ |
| Cedar policy language | https://docs.cedarpolicy.com/ |
| JSON-RPC 2.0 specification (error codes) | https://www.jsonrpc.org/specification |
| Model Context Protocol | https://modelcontextprotocol.io/ |
| JSON Web Token (RFC 7519) | https://www.rfc-editor.org/rfc/rfc7519 |
| OAuth 2.0 (RFC 6749), PKCE (RFC 7636) | https://www.rfc-editor.org/rfc/rfc6749 , https://www.rfc-editor.org/rfc/rfc7636 |
| Amazon Cognito developer guide (groups, tokens, hosted UI) | https://docs.aws.amazon.com/cognito/latest/developerguide/ |
| AWS Lambda developer guide (event, context, roles) | https://docs.aws.amazon.com/lambda/latest/dg/ |
| IAM user guide (roles, trust policies) | https://docs.aws.amazon.com/IAM/latest/UserGuide/ |
| AWS CloudFormation user guide (stacks, rollback) | https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/ |

**Suggested order, about 3 hours total:** header allow-list page (10 min) → Cedar "policy language" basics (30 min) → Cognito tokens and groups (30 min) → interceptor pages (30 min) → JSON-RPC errors section (10 min) → MCP "tools" section (20 min) → Lambda event and context (20 min) → IAM roles and trust policies (30 min).

## Your own files worth re-reading

- `AFLStartingTeamAgent/agentcore/agentcore.json` (the whole system in one file)
- `AFLStartingTeamAgent/app/s1a_echo/main.py` (forwarding the token, ~110 lines)
- `AFLStartingTeamAgent/tests/poc_matrix.py` and `whoami_check.py`
- `AFLStartingTeamAgent/lambda/whoami/` (Lambda, tool schema, trust policy, Cedar file)
- `07_AWS/poc/TRACE_LOG.md` (every observation, with dates)

## Where we stopped, and what comes next

- **Done:** token forwarding on Runtime, group-based Cedar, the group-flip demo, `whoami` showing the Lambda cannot identify callers.
- **Next (your choice):** (a) **P3b-Cedar:** require `user_id` to equal the token's `sub` in policy, the preview for questions 9–10; (b) **P3c:** an interceptor via `update-gateway`, watching whether `agentcore deploy` removes it; (c) **S1c:** put a model in the agent and try prompt injection.
