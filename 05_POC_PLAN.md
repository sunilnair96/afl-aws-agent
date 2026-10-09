# 05 — POC: "How does the user's identity travel, and who controls what they can access?"

*v0.1, 2026-10-08. Purely a learning proof-of-concept. It is **not** the AFL app — it has no team building, no real UI.
It answers one question you can show on a diagram and in a LinkedIn post:*

> User A and User B send the **same** request. Which component decides they get different answers, and what does each component actually see?

## 0. How we work (your rule: understand every command)

- **You run the commands; I explain them first.** Each command in this POC gets a 4-line card before you run it:
  **What** it does · **Why** we need it · **Expect** (the output you should see) · **Undo** (how to remove what it created).
- One step at a time. If a card isn't clear to you, we stop and I rewrite it. Nothing is "just run this script".
- Every step ends with a **trace note** (see §3): *what identity information existed at this point?* Those notes become the learning material.
- Code is small and commented line-by-line where it isn't obvious. No frameworks we don't need.

Worked example of a card (we will do this one in step P0-2):

```
Command : python -c "import base64,json,sys; p=sys.argv[1].split('.')[1]; p+='='*(-len(p)%4); print(json.dumps(json.loads(base64.urlsafe_b64decode(p)),indent=2))" <token>
What    : Splits a JWT on '.', takes the middle part (the payload), base64-decodes it and pretty-prints the JSON claims.
Why     : A JWT is NOT encrypted — it is signed. Reading the claims shows exactly what the Gateway and Cedar will see (sub, client_id, groups).
Expect  : JSON with "sub", "iss" (your Cognito pool URL), "client_id", "token_use": "access", "exp".
Undo    : Nothing to undo — read-only. (Never paste a live token into a chat or a git repo.)
```

## 1. What the POC proves

| Question | How the POC answers it |
|---|---|
| What does a JWT contain, and who issues/validates it? | Decode tokens for user A and B; compare claims |
| How does the user's identity get from the browser/CLI to the tool? | Trace points T1–T5 (below) |
| Who enforces "B only sees 2025+"? | Toggle each layer on/off and watch the result change |
| What does the **Lambda** actually know about the caller? | A `whoami` tool that echoes its own input event |
| Can the LLM bypass the rule by lying (prompt injection)? | Tell the agent "pretend I'm user A / use user_id=…" and show it fails |

## 2. Cast

- **User A** = `full-history` group. **User B** = `recent-only` group. Both are you, signed in with two Cognito accounts
  (Gmail plus-addressing, e.g. `<you>+usera@gmail.com` and `+userb@`, lands in your one inbox for verification codes).
  You already have one existing test user (`<USER_B_SUB>-…`, the "User B" in the current policy) — we reuse it as B.
- **Tools** (both tiny):
  - `get_player_performance(player_name, season)` — exists today.
  - `whoami()` — **new**; returns the `event` and `context` the Lambda received, so we can see what identity reached it.

## 3. Trace points (the heart of the POC)

```
 T1 Cognito token      what claims exist? (sub, client_id, cognito:groups)
 T2 Runtime            did the Authorization header reach the agent code?   (S1)
 T3 Gateway            token validated? which interceptor input/output?      (S3)
 T4 Cedar policy       decision per call: ALLOW / DENY (LOG_ONLY, then ENFORCE)  (S2)
 T5 Lambda             what is in `event`? is user identity in it at all?
```

Expected learning (to confirm, not assume): the Lambda target receives only the tool arguments, **not** the JWT — which is exactly
why the interceptor is needed to inject trusted `user_id`/`min_season`, and why the LLM must never be the source of those values.

## 4. Stages

| Stage | Build | Test | Time | Learn |
|---|---|---|---|---|
| **P0-1** | Account hygiene + budget alarm (`04_RUNBOOK_PHASE0.md` §1) | Budget email arrives | 1–2 h | IAM, Budgets, root-user safety |
| **P0-2** | Replay what you already built: get a token (your `cognito_test.py`), decode it, call the existing Gateway with curl | Positive (2025) and negative (2023) calls for user B | 2 h | JWT anatomy, MCP-over-HTTP, the Gateway endpoint |
| **P1** | Two Cognito **groups** and users A/B; add `whoami` Lambda as a second Gateway target | A and B tokens show different `cognito:groups` | 3 h | Cognito groups, Lambda event shape, Gateway targets |
| **P2** | **Cedar by group** instead of by user `sub` (S2); run LOG_ONLY then ENFORCE | Matrix rows 2–4 in `01_PLAN.md` §6, from curl, no agent | 4 h | Cedar syntax, default-deny, policy logs |
| **P3** | **Interceptor** Lambda that overwrites `user_id`/`min_season` (S3) | `whoami` as A vs B shows injected values; forged `user_id` is overwritten | 4 h | Gateway interceptors, trust boundaries |
| **P4** | **Agent on AgentCore Runtime** forwarding the token (S1), model = cheapest adequate (Haiku-class) | Same prompt as A and as B → different, correct outcomes; injection attempt fails | 6–8 h | Runtime, Strands, header allow-list, Bedrock |
| **P5 (optional)** | One static `index.html` + ~80 lines of JS: login button, "Ask" box, panel showing decoded claims + response. Served locally with `python -m http.server`. Nothing hosted on AWS. | Screenshot A vs B side-by-side for the post | 3 h | OAuth2 code+PKCE from a browser, CORS |

Exit: the **A/B matrix** passes at curl level (P2/P3) *and* through the agent (P4). Then we stop and write the post.

### Stage P4 note — what if token forwarding doesn't work?
Fallbacks, in order: (1) retry the harness after a CLI update; (2) Lambda-side token exchange; (3) agent runs locally (`agentcore dev`) while the
Gateway/Cedar/Lambda stay in AWS — still proves A vs B authorization, just not Runtime hosting. We write up whichever is true.

## 5. Is the "React SPA" in the plan a React app? — what P5 actually is

- **SPA (single-page application)** = one HTML page that loads once; JavaScript then changes what's on screen and talks to APIs, instead of the browser loading a new page per click. Gmail and Google Maps behave this way.
- **React** = a popular JavaScript library for building such pages from small reusable pieces ("components"). Your existing `app/frontend/` is already a React + TypeScript + Vite app (the thing you run with `npm run dev`).
- **For the POC we do not use React.** P5 is a single hand-written HTML file — fewer moving parts, and you can read every line.
- **For the proper app (after the POC)** we reuse your existing React frontend and add (a) Cognito login and (b) a chat panel. That is the "React SPA" box in the architecture diagram. Hosting it on S3 + CloudFront is the last, optional step — local `npm run dev` is fine for months.

## 6. Cost and free-tier notes (check these yourself in the console)

- Your 170 days run to about **27 Mar 2027**; your 2-month learning window (to ~8 Dec 2026) sits inside it.
- Newer AWS free plans work on a **credit balance + time limit** rather than per-service free tiers: check *Billing → Credits* and *Billing → Free tier* for your real numbers, and what happens when credits/time run out (accounts on the free plan can be closed or restricted unless upgraded). I have not seen your account, so treat this as something to verify, not a fact.
- Likely spend drivers: **Bedrock model tokens** and AgentCore Runtime/Gateway/Memory usage. Lambda, Cognito (small user counts), S3, DynamoDB at POC scale should be negligible.
- Cheap-POC rules: Haiku-class model while testing; `maxIterations` ≤ 5; `maxTokens` small; delete what you're not using (`agentcore remove` + `agentcore deploy`); keep the Budget alarm at 1.4 of the runbook.
- Do **not** use Athena for the POC (not needed; the `whoami` + existing performance tool suffice).

## 7. Deliverables of the POC

1. A trace table (T1–T5) for A and B filled with real observed values.
2. An automated mini test script `poc_matrix.py` (calls Gateway as A and B, asserts allow/deny). Lives in the GitHub repo: `C:\005awsprojects\afl-analytics\AFLStartingTeamAgent\tests\poc_matrix.py`.
3. One diagram of the identity flow with the *observed* data at each hop.
4. LinkedIn post #3 (`06_LINKEDIN_SERIES.md`).
