# 09 — Build log: search tools for the harness (find_players)

*Built and tested by Claude on 2026-10-11 while the owner was away (authorised in chat: "build and test, don't wait for me to execute"). Everything below was actually run; results are observed. Identifiers are placeholders: `<ACCOUNT_ID>`, `<BUCKET>` (the project's S3 bucket).*

## 1. The problem and the answer

In the Harness playground, `AFLHarness_quickHarness` was asked: *"List players who averaged less than 60 in the AFL Fantasy season but has scored over 80 in some games."* It correctly replied that its only tool looks up **one named player in one named season**, so it could not search.

- **The fix is tools, not a smarter agent.** I added a search tool and a game-log tool over the same Athena data.
- **Sub-agents are not needed.** One agent with three well-described tools is simpler, faster and cheaper. Revisit sub-agents only if the tool list grows past roughly a dozen or splits into clearly separate jobs.
- **Which harness:** `quickHarness` (IAM gateway). Historical stats are the same for every user. `AFLHarness_clientToolHarness` (the per-user one) was **not changed** (still version 1).

## 2. Result (observed)

| Question to the harness | Answer | Ground truth (local data) |
|---|---|---|
| AF 2026, average under 60, over 80 in some games, top 5 | Zach Reid, Brady Hough, Sam Banks, Taylor Walker, Tom Brown; "108 players in total" | 108 players; same top five |
| Top 3 rucks by SuperCoach average, 10+ games | Gawn 128.0, Grundy 119.9, L. Jackson 117.5 | Identical |
| Nick Daicos AF 2026: average and three best rounds | 116.7 over 22 games; R15 148, R16 143, R3 140 | Identical |
| Jordan Dawson 2025 (the old tool) | 25 games, 108.88, high 147, low 78 | Same as the earlier test |
| "Who will win the 2027 premiership?" | Declined: no tool for predictions | (no tool exists) |

One thing to watch: in the Daicos answer the model added "consistently scoring in the 95-140 range". That sentence is the model's own wording, not a tool field, and is loose (his range was 69 to 148; one game below 95). The numbers it quoted from the tool were exact.

## 3. Files (all in `C:\005awsprojects\afl-analytics\AFLHarness\`)

| File | Status | What |
|---|---|---|
| `lambda/find_players/lambda_function.py` | **new** | The Lambda: validates arguments, builds the Athena SQL, two tools (`find_players`, `get_player_game_log`) |
| `lambda/find_players/tools.json` | **new** | The tool descriptions the model reads ("over 80" means `score_at_least` 81) |
| `lambda/find_players/trust-policy.json` | **new** | Lets Lambda assume the role |
| `lambda/find_players/policy.template.json` | **new** | The role's permissions, with `<ACCOUNT_ID>` and `<BUCKET>` placeholders |
| `tests/test_find_players.py` | **new** | 30 unit tests (no AWS), including attempts to smuggle SQL through parameters |
| `tests/live_check_find_players.py` | **new** | Runs the tool against real Athena and compares with the local data (5 checks) |
| `agentcore/agentcore.json` | edited (via `agentcore add gateway-target`) | New target `FindPlayersTargetIAM` on `AFLGatewayIAM` |
| `app/quickHarness/system-prompt.md` and `app/quickHarness/harness.json` | edited | New instructions: the three tools, defaults (2026, AF), "over N means N+1", report the total |
| `agentcore/.cli/deployed-state.json` | changed by the deploy | Bookkeeping |

Not mine and untouched: the unrelated uncommitted edits under `src/afl_analytics/bedrock/` and `src/afl_analytics/config.py`.

## 4. AWS resources created or changed

| Resource | What |
|---|---|
| IAM role `afl-find-players-role` | **New.** Narrow permissions: Athena (primary workgroup only), Glue read on the `afl_analytics` database, S3 read on `processed/*`, S3 write on `athena-results/*`, plus the basic Lambda logging policy |
| Lambda `afl-find-players` | **New.** Python 3.14, 128 MB, 60 s, env `CURRENT_SEASON=2026` |
| Gateway target `FindPlayersTargetIAM` (id `<TARGET_ID>`) | **New**, on `AFLGatewayIAM` |
| Gateway's role policy | **Changed**: may now also invoke `afl-find-players` |
| Harness `quickHarness` | **Changed**: version 1 → **2** (new instructions) |
| Stack `AgentCore-AFLHarness-default` | `UPDATE_COMPLETE` (checked independently) |

## 5. Commands run, in order (Git Bash; run from `AFLHarness` unless stated)

```bash
# 0. Unit tests (no AWS)
cd /c/005awsprojects/afl-analytics
uv run --with pytest pytest AFLHarness/tests/test_find_players.py -q          # 30 passed

# 1. Live check of the SQL against real Athena, before deploying anything
uv run python AFLHarness/tests/live_check_find_players.py                      # 5 PASS

# 2. Role
cd AFLHarness
ACCT=$(aws sts get-caller-identity --query Account --output text)
sed "s/<ACCOUNT_ID>/$ACCT/g; s/<BUCKET>/<BUCKET>/g" lambda/find_players/policy.template.json > policy.rendered.json
aws iam create-role --role-name afl-find-players-role --assume-role-policy-document file://lambda/find_players/trust-policy.json
aws iam attach-role-policy --role-name afl-find-players-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam put-role-policy --role-name afl-find-players-role --policy-name FindPlayersAthena --policy-document file://policy.rendered.json
rm policy.rendered.json

# 3. Lambda (zip contains only lambda_function.py)
python -c "import zipfile; zipfile.ZipFile('find_players.zip','w').write('lambda/find_players/lambda_function.py','lambda_function.py')"
aws lambda create-function --function-name afl-find-players --runtime python3.14 --handler lambda_function.lambda_handler \
  --role arn:aws:iam::$ACCT:role/afl-find-players-role --zip-file fileb://find_players.zip --timeout 60 --memory-size 128 \
  --environment "Variables={CURRENT_SEASON=2026}" --region ap-southeast-2
aws lambda wait function-active-v2 --function-name afl-find-players --region ap-southeast-2

# 4. Direct invoke test (see section 7 for the payloads)

# 5. Gateway target, preview, deploy
agentcore add gateway-target --name FindPlayersTargetIAM --description "Search players by season statistics and read a player's game log" \
  --gateway AFLGatewayIAM --type lambda-function-arn --lambda-arn arn:aws:lambda:ap-southeast-2:$ACCT:function:afl-find-players \
  --tool-schema-file lambda/find_players/tools.json
agentcore deploy --target default --diff        # preview: 1 new target + gateway policy gains the new function
agentcore deploy --target default --yes -v      # deploy (harness instructions edited first)

# 6. Check and ask
aws cloudformation describe-stacks --stack-name AgentCore-AFLHarness-default --region ap-southeast-2 --query "Stacks[0].StackStatus" --output text
agentcore invoke --harness quickHarness "List players who averaged less than 60 in the AFL Fantasy season but scored over 80 in some games. Show the top 5."
```
The scratch file `find_players.zip` was created in a temp folder; delete any copy in the project folder.

## 6. How to test it yourself

In the console Harness playground, pick `AFLHarness_quickHarness`, **start a new session** (an old session keeps the old tool list), and ask for example:
- "Which forwards averaged over 90 in SuperCoach in 2026?"
- "Show Tim English's scores round by round in 2025."
- "Which players scored at least 120 in three or more games but averaged under 100?"

From the terminal: `agentcore invoke --harness quickHarness "your question"` (in `AFLHarness`).

## 7. Direct Lambda test payloads (observed results)

| Payload | Result |
|---|---|
| `{"average_below":60,"score_at_least":81,"limit":3}` | `total_matched` 108, three players |
| `{"player_name":"Nick Daicos","season":2026}` | 23 games (22 home-and-away plus one final), average 116.7 |
| `{"limit":500}` | `{"error": "Invalid request: limit must be a whole number from 1 to 50"}` |
| `{"team":"Power'); DROP TABLE x; --"}` | `{"error": "Invalid request: team may contain only letters, spaces, dots, apostrophes and hyphens"}` |

## 8. Safety design (why it is built this way)

- **No free-form SQL.** The model fills named parameters; the code validates each (numbers by range, text by an allow-list or a strict pattern, apostrophes doubled) and builds the query itself. 15 hostile inputs are rejected in the tests; no generated query contains a semicolon.
- **Results are limited** (default 25, maximum 50) and always carry `total_matched`.
- **Errors are returned as results**, so the model can correct a bad call instead of the call failing.
- The role can only run queries in the `primary` Athena workgroup, read the `afl_analytics` table definitions, read `processed/*` and write `athena-results/*`.

## 9. Known limits and ideas

- Data is the 2026 season and earlier in the table; a player traded mid-season appears as two rows (one per club).
- Position labels come from the data (for example "Defender Midfield"); `position` filters on containing a word.
- Each query writes a small result file to `athena-results/`, adding to the clutter; an expiry rule is still undecided.
- Ideas, not built: a season leaderboard tool; consistency measures (standard deviation, games over a threshold as a share); a link to the player master (age, price) for questions such as "cheapest players who averaged over 90".
- The three tools are all the harness sees; the playground shows one tool call per step only in the trace.

## 10. Undo

```bash
cd /c/005awsprojects/afl-analytics/AFLHarness
agentcore remove gateway-target --name FindPlayersTargetIAM       # then: agentcore deploy --target default --yes
git checkout -- app/quickHarness/harness.json app/quickHarness/system-prompt.md      # restores the old prompt; deploy again
aws lambda delete-function --function-name afl-find-players --region ap-southeast-2
aws iam delete-role-policy --role-name afl-find-players-role --policy-name FindPlayersAthena
aws iam detach-role-policy --role-name afl-find-players-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name afl-find-players-role
```
(Check the exact `agentcore remove` syntax with `agentcore remove --help` first.)

## 11. State left behind

Nothing is committed. In `afl-analytics`: new `AFLHarness/lambda/`, two new test files, and edits to `agentcore.json`, the two prompt files and `deployed-state.json`. In `07_AWS`: this log. Cost so far: Lambda invocations and a few Athena queries on a 7.6 MiB data set, which should be cents or less (**check Cost Explorer by service**).

## 12. Follow-up (2026-10-11, afternoon): the age filter

**What the owner tested:** "…who averaged under 60, scored over 80 in at least one game, sorted by number of 80+ scores, aged 23 or less as of today." The harness (version 2) answered honestly that its tools could not filter by age, then listed all 108 players, showing 14, and said the list "includes all 108" (it had only been given 50). Two problems: no age filter, and an overstated answer.

**Fixes (all deployed and tested):**

| Change | File |
|---|---|
| New filters `min_age` and `max_age`; `age` added to each result row; an `age_note` field saying what age means | `lambda/find_players/lambda_function.py` |
| The two new parameters, described honestly ("roster age in whole years, NOT an exact birthday calculation") | `lambda/find_players/tools.json` |
| Instructions: show every returned player unless asked for fewer; say how many were left out; never claim "all matches" unless returned equals `total_matched`; say age is approximate | `app/quickHarness/system-prompt.md` and `harness.json` |
| 8 new unit tests (38 in total); 2 new live checks (7 in total) | `tests/test_find_players.py`, `tests/live_check_find_players.py` |

**What "age" means here (important):** the data stores each player's age in that season's roster, in whole years. It has no birth dates (only the draft dossier does), so "as of today" can only be approximated. For 2026 data in October the roster age is the right order of magnitude but can be a year out for a player whose birthday fell after the roster was recorded.

**Commands run (from `AFLHarness`; Git Bash):**

```bash
# unit tests and live check (the live check compares with the local match file)
cd /c/005awsprojects/afl-analytics
uv run --with pytest pytest AFLHarness/tests/test_find_players.py -q            # 38 passed
uv run python AFLHarness/tests/live_check_find_players.py                        # 7 PASS (age 23 or under: 46 = 46)

# new function code
cd AFLHarness
python -c "import zipfile; zipfile.ZipFile('find_players.zip','w').write('lambda/find_players/lambda_function.py','lambda_function.py')"
aws lambda update-function-code --function-name afl-find-players --zip-file fileb://find_players.zip --region ap-southeast-2
aws lambda wait function-updated-v2 --function-name afl-find-players --region ap-southeast-2

# new tool descriptions and instructions
agentcore deploy --target default --yes -v          # quickHarness version 2 -> 3; stack UPDATE_COMPLETE
agentcore invoke --harness quickHarness "List the player who averaged less than 60 in AFL Fantasy in 2026 and scored over 80 in at least one game, sorted by the number of scores over 80, aged 23 or less as of today"
```

**Result (observed):** 46 players, sorted by games of 81 or more (Brady Hough 4, Sam Banks 4, Tom Brown 4, Nate Caddy 3, then the 2s and 1s), all aged 18 to 23, "All 46 matched players are included" (true this time). The local calculation gives the same 46 and the same top eight.

**One thing to watch:** although the instructions say to call age approximate, the answer simply said "aged 23 or less" and did not mention the roster-age caveat. The tool result does carry the `age_note`. If that matters, say so and I will make the instruction firmer or put the caveat in the tool result's first line.

**State:** `quickHarness` is on **version 3** (`clientToolHarness` still version 1). Nothing committed.
