# 02 — How we tested (the method, the experiments, and what I got wrong)

*It looked easy because each step was small and answered one question. The method below is what made it small. The last section is the part that looked easy only in the retelling.*

## The method in ten rules

1. **Go bottom-up, one layer at a time.** Token → gateway → policy → Lambda → *then* the agent. When something fails you know which layer, because the layers below already work.
2. **Isolate the hop that fails.** The harness failed, but curl-to-gateway had always worked. That told us the problem was only the agent → gateway hop, and we tested exactly that hop with a no-model agent (S1a/S1b), at almost no cost.
3. **Predict before you run, and write the prediction down.** `poc_matrix.py` has a `predicted` column. A wrong prediction (e.g. `tools/list` hiding `whoami`) teaches more than a right one.
4. **Test the positive and the negative path.** "Allowed" alone proves little. Every experiment had an ALLOW row and a DENY row.
5. **Change one variable at a time.** The group-flip demo changed only B's group membership and token. Nothing else moved, so the cause was certain.
6. **Prove it with evidence from a different place.** "The Lambda didn't run" came from CloudWatch (one invocation only), not from the gateway's say-so. Token contents came from decoding the token, not from trusting a screenshot.
7. **Tell failures apart.** HTTP 401/403 = the token or request was rejected (authentication/scope). HTTP 200 + JSON-RPC `-32002` "Tool Execution Denied" = a policy decision. HTTP 200 + `isError:false` = the tool ran. Your script had to learn this distinction.
8. **Read the log's reason line, not the whole log.** `Reason: OpenTelemetry instrumentation executable not found` was one line in a long deploy log.
9. **Keep the test as a script.** `tests/poc_matrix.py` and `tests/whoami_check.py` can be re-run after every policy change. The tokens come from environment variables, so nothing secret is stored.
10. **Write every observation down** (`poc/TRACE_LOG.md`), including the ones that contradict what you said earlier.

## The experiments

| # | Question | What we ran | Predicted | Observed | What it proved |
|---|---|---|---|---|---|
| 1 | What does the token contain? | `cognito_test.py`, decode payload | groups present? | `sub`, `client_id`, `scope`; **no** groups, **no** email | The policy can only see what is in the token |
| 2 | Does the gateway accept the token and enforce the rule? | curl `tools/list`, `tools/call` 2025 and 2023 | 2025 allow, 2023 deny | as predicted; denial = HTTP 200 + error body | Cedar works; denial is in the body |
| 3 | Did the Lambda run on the denied call? | `aws logs tail` | one invocation | one invocation | Denial happens *before* the Lambda |
| 4 | Does the token reach agent code on Runtime? | `s1a_echo` handler returns decoded claims | header present? | yes, plus `baggage` and `workloadaccesstoken` | The allow-list works (S1a) |
| 5 | Can the agent forward it to the gateway? | `s1a_echo` calls the gateway | allow/deny same as curl | identical to curl | The blocked hop is solved on Runtime (S1b) |
| 6 | Do groups reach the token? | add users to groups, sign in again | claim appears | `["full-history"]`, `["recent-only"]` | Membership → claim, after a *new* sign-in |
| 7 | Baseline with the old per-user policy | `poc_matrix.py` | A denied everywhere | as predicted | Default deny; the "intent gap" |
| 8 | Can Cedar read groups? | add group policy, deploy, matrix | A allowed both seasons | as predicted | `principal.getTag("cognito:groups")` works (S2) |
| 9 | Does access follow the group? | add B to `full-history`, new token, matrix | B/2023 flips to ALLOW | flipped; then flipped back | Group controls access, not code |
| 10 | What does the Lambda know about the caller? | `whoami` before/after permit | args only | args + gateway ids; **no identity** | Tools cannot identify callers |
| 11 | Is a new tool open by default? | call `whoami` before any permit | denied | denied, and **hidden** from `tools/list` | Default deny; the tool list is policy-filtered |
| 12 | Does a stricter permit help while a looser one exists? | add `WhoamiBoundToCaller` next to the old permit, run `whoami_binding_check.py old-permit-active` | all ALLOW (permits are additive) | all 8 ALLOW; A sent B's real sub and the Lambda received it | A forgery works until the broad permit is removed |
| 13 | Can policy bind `user_id` to the verified `sub`? | remove the old permit, redeploy, run `bound-only` | only "own sub" allowed | exactly that, for both users, 0 gaps | `principal.getTag("sub")` is readable; the Lambda can trust `user_id` |
| 14 | Can a model be talked into impersonating a user? | `s1c_agent` (Strands, Haiku 4.5), 7 prompts as A/B (`s1c_attack_check.py`) | 0 invariant violations | 0 violations; case 6: model chose the unsafe tool with A's id and Cedar **DENIED**; case 7: the model refused by itself (policy not exercised) | Two layers (code injects `user_id`, policy checks it). The model's own refusal is not a control |
| 15 | Does a real browser login work end to end? | `webtest` page, public client + PKCE, users A and B | login, then allowed | two login traps, then HTTP 401 `-32001` (client not allow-listed) as predicted, then correct allow/deny for both users | No token copy/paste; see `05_WEB_LOGIN_AND_PKCE.md` |
| 16 | Can a second target (`prod`) in the same account be deployed? | `--diff`, a policy-neutralising change in dev first, then a real `--target prod` deploy | preview shows 15 resources; at least one name clash | preview clean; both policy rewrites rejected by the service and rolled back; prod creation failed (policy-engine name exists) and rolled back; cleanup verified "No differences" on dev | Environments need separate accounts; policies must name their gateway ARN; `--diff` is not a promise. See `06_ENVIRONMENTS_AND_DEPLOYMENT.md` |

## What I got wrong (so you don't have to)

| Claim or step | What was wrong | How we found out |
|---|---|---|
| Both test tokens were "A" and "B" | `TOKEN_B` was User A's token; Cognito's login page reused A's session | The script printed `A sub starts <USER_A_SUB>, B sub starts <USER_A_SUB>`. It now refuses identical subs |
| Ran the check with stage `old-permit-active` after removing the old permit | The label sets the prediction; it doesn't change the system | "PREDICTION WRONG" on 6 rows whose actual results matched `bound-only` |
| "Edit `requirements.txt` to add a dependency" | The CLI installs from `pyproject.toml` | Same error after the edit; read the CLI's packaging source |
| `--tool-schema-file ..\lambda\whoami\tools.json` | Path is relative to where you run the command | "Tool schema file not found" |
| "`tools/list` isn't filtered by Cedar" | It **is** filtered. I over-generalised from one observation | Prediction failed in experiment 11 |
| "Token expired" for A's HTTP 403 | An expired token would normally give 401; the cause of the 403 was never established | Not decoded before replacing; recorded as unexplained |
| "Both users see both tools" | No, only tools they may call | Experiment 11 |
| Assumed the harness user-federation flow was the only way | A Runtime agent can forward the header | Spike S1 |
| Post 2 was published | It was never published | You told me; docs corrected |
| Multi-line `git commit -m @'…'@` in PowerShell | Treated as file names | "pathspec did not match"; now commit with `-F <file>` |
| `.gitignore` appended with `echo` | Entries joined on one line (no trailing newline); the token file stopped being ignored | `git check-ignore` printed nothing |
| First `-32002` check | The same code is used for non-policy failures | Experiment: HTTP 403 `insufficient_scope` also had `-32002` |
| "The harness's managed memory created the KMS key and the knowledge-base cost" | The key belongs to the **CDK bootstrap stack**; the knowledge-base line more likely comes from two other knowledge bases in the account | Stack resource listing, key grants and tags; I had inferred from timing |
| "Haiku 5.5 is available (listed)" | Listed and ACTIVE profiles, but `AccessDenied: not available for this account` | A real call |
| Created a Cognito client from the CLI and expected a login page | Managed login needs a branding style per client | "Login pages unavailable" |

## Habits to copy

- Before every command: **what, why, expected output, how to undo**.
- After every command: compare the output with the expectation. Surprises are the lessons.
- Don't claim what you haven't observed. Write "inferred" or "unknown" in your notes.
- Never put a token, secret, or client secret in chat, git, a screenshot or a post.
