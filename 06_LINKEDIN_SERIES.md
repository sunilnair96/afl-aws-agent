# 06 — LinkedIn series plan (learning in public)

*2026-10-08. Post 1 (Harness → Gateway → Lambda → Athena) is out. The identity-aware authorization post ("Post 2") was **never published**: only draft files exist. It is now rewritten as `poc/LinkedInPost_2_DRAFT.txt` with the agent-hop finding, and is the next post to publish.
This plan turns the next two months into short posts that double as learning notes. Rule: **only post what you actually ran and observed** — no claims from my plan docs until they're tested.*

## Format that suits your existing posts

- ~250–300 words, one diagram, one honest "what surprised me / what failed", 3–5 bullets of lessons, closing line "Learning prototype in my personal AWS account. No client or proprietary data."
- 8–9 hashtags max, reused consistently (`#AmazonBedrock #AgentCore #AWSLambda #Cedar #AgenticAI #CloudArchitecture`).
- Lead with the question the reader has, not the service name.
- Show the failure path (denied request, bug you hit). It is what makes the post credible, and your `BUG_REPORT_defaultReturnUrl.md` is already a good seed.

## Series

| # | When | Working title | Evidence you'll have | Diagram |
|---|---|---|---|---|
| 2 (rewrite) | now (draft ready) | **"Whose identity is the agent acting on?"** token forwarding Runtime → Gateway, Cedar allow/deny, harness lesson | S1 evidence in `poc/TRACE_LOG.md` | `poc/architecture_post3.html` |
| 3 | after POC P3/P4 (≈ end Oct) | **"The Lambda never sees the user: where identity actually goes in an agent"** — what the tool receives vs. what the token contains; why an interceptor injects `user_id` instead of trusting the LLM | `whoami` output for A and B; forged `user_id` overwritten | identity flow with observed values at T1–T5 |
| 4 | ≈ mid-Nov | **"Group-based Cedar policies: one rule instead of one per user"** (or the honest "it didn't work, here's the fallback" if S2 fails) | LOG_ONLY vs ENFORCE logs | policy decision table |
| 5 | ≈ late Nov | **"Moving an agent from a managed harness to Runtime: what I gained and lost"** — the OAuth return-URL wall, token passthrough | bug report + working passthrough | harness vs Runtime comparison |
| 6 | ≈ early Dec | **"Same optimiser, three front doors: FastAPI, MCP, Lambda"** — reusing one solver (PuLP) behind each; Lambda container sizing/cold starts | measured cold start + latency | reuse diagram |
| 7 | ≈ mid Dec | **"Evaluating an agent: a test set + an authorization matrix as the release gate"** | eval results | test matrix |

## Post 3 skeleton (fill from real results; do not publish until the blanks are real)
*(Written when I thought Post 2 was published; its opening line refers to "my last post" — adjust to the Post 2 rewrite.)*

> In my last post I showed Cedar denying a request before it reached the tool. A reader's obvious question: **what does the tool itself know about who is calling?**
>
> I added a tiny `whoami` tool that returns exactly what the Lambda receives. Result: [observed — e.g. "only the tool arguments; no token, no groups"].
>
> So where does trusted identity come from? [observed flow]. Design rules I ended up with:
> • The LLM never supplies identity. [evidence: forged user_id attempt → …]
> • [rule 2]
> • [rule 3]
>
> What surprised me: [real surprise]. What failed first: [real failure].
>
> Diagram below. Learning prototype in my personal AWS account; no client or proprietary data.
> #AmazonBedrock #AgentCore #AWSLambda #Cedar #AgenticAI #AISecurity #CloudArchitecture

## Profile-building tips (low effort)

- Pin the series: reply to each new post with a link to the previous one so the thread reads as a learning journey.
- Put a short public write-up (sanitised) in a GitHub repo: the `07_AWS` repo can be made public later **after** removing account IDs, pool/client IDs and any tokens (do a secrets scan first). Docs and diagrams are the safe part; deployed-state files are not.
- Add "Amazon Bedrock AgentCore, AWS Lambda, Amazon Cognito, Cedar" to LinkedIn Skills only once you've done the POC.
- Consider an AWS cert (e.g. AI Practitioner / Developer Associate) later — your `AIP-Materials.zip` suggests you're already on this path; it pairs well with the posts.
