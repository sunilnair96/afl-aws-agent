# Pre-publication review of the (unpublished) Post 2 against what the POC showed

*Update: Post 2 was **never published**; only the files exist. So nothing needs correcting publicly. This is a checklist of what to fix before the identity post goes out. The replacement draft is `LinkedInPost_2_DRAFT.txt`; the matching diagram is `architecture_post3.html`.*

*2026-10-08. Compared: `PostText_revised.txt` (4 Oct), `LinkedInPost_final.txt` and `arch.html` (6 Oct; same timestamp as the PDF, which I could not render here) with the evidence in `TRACE_LOG.md`. I do not know which version you published; both are covered. "Wrong" below means "a reader who tries to reproduce it would hit a gap", not that the main idea fails. The core message (rules outside the LLM, default deny, deny before the Lambda) still stands and is now better evidenced.*

## What holds up (now with evidence)

| Claim in Post 2 | Evidence |
|---|---|
| Unauthorized request denied; Lambda never invoked | ✔ CloudWatch: one invocation only (the allowed call) |
| Default deny: one permit with a condition is enough | ✔ Denial text: "No policy applies to the request (denied by default)" |
| Rule lives outside the LLM, evaluated by the Gateway before the tool | ✔ same decision whether called by curl or via agent code |
| Tool input parameters can be used in the rule | ✔ `season >= 2025` was enforced |

## What was inaccurate or unproven

| # | Where | Issue | What actually happened | Suggested correction |
|---|---|---|---|---|
| 1 | Diagram box 3 and the "final" post: "Agent calls the Gateway with the user's JWT" | **Never demonstrated through an agent.** Tests ran as *curl → Gateway*. When a managed **harness** was put in the path it failed with `ResourceOauth2ReturnUrl`: its outbound OAuth step is separate from the inbound token. | Forwarding the user's token to the Gateway only worked after moving to a **Runtime** agent that forwards the `Authorization` header | Say how the agent gets the token, and that this needed a Runtime agent, not the managed harness |
| 2 | "Revised" post: `User → Cognito → Gateway` (no agent) | Skips the hop where the problem lived | See #1 | Show the agent in the path |
| 3 | Diagram JWT sample: `"username": "userb"`, `"cognito:groups": ["analyst"]` | Real access token here has `sub`, `username` (= the UUID), `client_id`, `scope`; **no** groups, **no** friendly name | Groups appear only after the user is in a Cognito group (not yet done) | Label it "illustrative" or use real fields (`sub`, `client_id`) |
| 4 | Diagram policy: `principal.hasTag("username") && principal.getTag("username") == "userb"` | The deployed policy used the principal id (`principal == AgentCore::OAuthUser::"<sub>"`), not tags. The tag form was **not tested**; and `username` here is a UUID, so `"userb"` would never match | Spike S2 will test claim/group-based rules | Show the policy you actually ran, or test tags first |
| 5 | "Decides … from token claims (principal tags)" | Only the principal id was used in practice | — | Say "from the caller's identity" until groups are tested |
| 6 | Hard-coded user in the rule | One policy per user does not scale | A group-based rule is the better design (to be proven) | Mention as a limitation |
| 7 | "Gateway returns an error" | Denial arrives as **HTTP 200** with a JSON-RPC error (`-32002`). A client that checks only the status thinks it succeeded | Observed | Add a one-line lesson |
| 8 | "Not inside the Lambda" | Right as the *first* line, but the Lambda should still re-check (defence in depth), since it is the last boundary | Design choice | Phrase as "not *only* in the Lambda" |
| 9 | "The JWT carries the user's identity into the request" | True at the Gateway, but the **Lambda does not receive the token** by default (to be confirmed with the `whoami` test) | Pending P3 | Don't imply the tool knows the user |

## What to do before publishing

Use `LinkedInPost_2_DRAFT.txt` (it already includes the agent-hop finding as the main lesson) with `architecture_post3.html` printed to PDF. Do not reuse the old `arch.html` PDF unless you first change its JWT sample and policy example to the real fields (items 3-5 above). The old text files in `Resume/LinkedINPosts/Post2/` are now superseded; keep them as history or delete them.
