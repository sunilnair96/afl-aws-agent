# 10. run_sql (Option D): read-only SQL tool for the harness

Built 2026-10-11 (evening). Nothing committed yet. Follows `09_FIND_PLAYERS_BUILD_LOG.md`.

## 1. Why

The harness answered "players aged under 28 who averaged over 100 in 2024, 2025 and 2026" with the right per-season lists but a wrong final "ALL three seasons" line: the model merged long lists by eye. The owner did not want a query-specific change to `find_players`, and chose a general capability: let the model write SQL and let Athena do the set arithmetic. (Option C, the Code Interpreter, was deferred; D only.)

## 2. What was built

| Piece | Where |
|---|---|
| Lambda `afl-run-sql`: one tool `run_sql(sql)` | `AFLHarness/lambda/run_sql/lambda_function.py` |
| Tool description (includes table schemas and rules) | `lambda/run_sql/tools.json` |
| IAM policy template (placeholders) and trust policy | `lambda/run_sql/policy.template.json`, `trust-policy.json` |
| 43 unit tests | `AFLHarness/tests/test_run_sql.py` |
| Live check through the deployed Lambda (10 checks) | `AFLHarness/tests/live_check_run_sql.py` |
| New gateway target `RunSqlTargetIAM` (id `QBWBWS73MY`) | `AFLHarness/agentcore/agentcore.json` |
| Routing instructions | `AFLHarness/app/quickHarness/system-prompt.md` and `harness.json` |

AWS resources created: Athena workgroup `afl-readonly-sql` (output location enforced, **10 MB scanned-per-query cap**; the whole data set is under 1 MB), IAM role `afl-run-sql-role` (+ inline `RunSqlAthena`, AWSLambdaBasicExecutionRole), Lambda `afl-run-sql` (python3.12, 256 MB, 60 s). quickHarness is now **version 7**; clientToolHarness untouched (version 1).

## 3. Safety (in layers; IAM is the one that matters)

1. **IAM**: role can read only Glue tables `glue_player_match_stats` and `glue_player_master` and their S3 prefixes; write only to `athena-results/`; Athena only in `afl-readonly-sql`. Checked with `aws iam simulate-principal-policy`: other table = implicitDeny, write to `processed/` = implicitDeny, `primary` workgroup = implicitDeny, delete in `athena-results/` = implicitDeny.
2. **Workgroup**: 10 MB scan cap, enforced output location.
3. **Code** (defence in depth): one statement; must start with SELECT or WITH; no `;`, comments, double quotes, backticks or backslashes; write/DDL keywords refused (inside quoted text values they are ignored); every table after FROM/JOIN must be on the allow-list or defined by the query's own WITH. Limitation: comma-joins (`FROM a, b`) are not inspected by the parser; IAM still blocks any unlisted table.
4. **Output**: at most 200 rows; `truncated` flag and a note when cut; errors (including Athena's) returned as `{"error": ...}` so the model can correct the query.

## 4. Results (observed)

Live check: **10 of 10 PASS** (three-season answer = local answer: Gulden, Sheezel, Daicos; 668 players = 668 local; refusals for DROP, `;`, unlisted table, information_schema; Athena error returned; 200-row cap flagged).

Harness tests (version 6), tool chosen and answer vs local ground truth:

| Question | Tool chosen | Answer correct? |
|---|---|---|
| Under 28, averaged over 100 in each of 2024/25/26 | run_sql | Yes (3 players) |
| How many averaged over 90 in 2026, and how many aged 23 or under | run_sql | Yes (56 and 13) |
| What did Nick Daicos average in 2026 | get_player_performance | Yes |
| Top 5 ruck averages 2026 | find_players | Yes |
| How many scored 130+ at least twice in both 2025 and 2026 | run_sql | Yes (13) |

## 5. What went wrong on the way (honest notes)

- **Version 4: the model ignored run_sql** (Haiku 4.5 used find_players and merged by eye, and even listed a 28-year-old as "under 28"). The tool was visible (listed by the harness); the soft instruction was not enough. Version 5 made the routing a MUST with an example query: both test questions then went to run_sql.
- **Version 5: wrong sub-count (11, truth 13).** Two separate causes: the model wrote `GROUP BY player, team, age` (age changes mid-season, which splits a player into two rows), and it counted "aged 23 or under" by eye from a 56-row list. Version 6: tool description and instructions say never group by age (use `MAX(age)`) and compute every count inside the query. Result then correct (56 and 13).
- **Version 6: games played came back as 1 for every player.** The model's SQL used `COUNT(DISTINCT CASE WHEN ... THEN 1 END)`, which counts distinct values of the constant 1 (so at most 1). The model noticed and flagged it but did not fix it. Version 7: tool description and instructions say count games with `COUNT(*)` or `COUNT(CASE WHEN ... THEN 1 END)`, never `COUNT(DISTINCT 1)`, and to fix and rerun a query whose count looks wrong. Rerun: Gulden 23/10/11, Sheezel 21/23/22, Daicos 23/23/22 (same as the correct earlier run).
- Lesson: a tool makes the right answer possible, but the model must be told when it is mandatory and how to avoid known traps. Only a counted test set shows whether it works.
- The workgroup cap is a safety net only; real protection is IAM.
- Data quirk found: the `age` column can change within a season (birthdays). `find_players` uses `MAX(age)`, which is consistent.

## 6. Commands run (Git Bash, from `AFLHarness`)

```bash
aws athena create-work-group --name afl-readonly-sql --region ap-southeast-2 --configuration "ResultConfiguration={OutputLocation=s3://<BUCKET>/athena-results/},EnforceWorkGroupConfiguration=true,BytesScannedCutoffPerQuery=10485760,PublishCloudWatchMetricsEnabled=true"
aws iam create-role --role-name afl-run-sql-role --assume-role-policy-document file://lambda/run_sql/trust-policy.json
aws iam attach-role-policy --role-name afl-run-sql-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam put-role-policy --role-name afl-run-sql-role --policy-name RunSqlAthena --policy-document file://<policy with placeholders filled in>
python -c "import zipfile; zipfile.ZipFile('run_sql.zip','w').write('lambda/run_sql/lambda_function.py','lambda_function.py')"
aws lambda create-function --function-name afl-run-sql --runtime python3.12 --handler lambda_function.lambda_handler --role arn:aws:iam::<ACCOUNT_ID>:role/afl-run-sql-role --zip-file fileb://run_sql.zip --timeout 60 --memory-size 256 --region ap-southeast-2
cd .. && uv run --with pytest pytest AFLHarness/tests/test_run_sql.py -q       # 43 passed
uv run python AFLHarness/tests/live_check_run_sql.py                           # 10 PASS
cd AFLHarness && agentcore deploy --target default --yes -v                    # run 3 times: versions 4, 5, 6
agentcore invoke --harness quickHarness "<question>"                          # add --verbose to see which tool was called
```

(The policy file used a temporary copy with the account and bucket filled in; it was deleted after use. The Lambda zip was deleted.)

## 7. Undo

```bash
agentcore remove ...   # or delete the RunSqlTargetIAM block from agentcore/agentcore.json and agentcore deploy
aws lambda delete-function --function-name afl-run-sql --region ap-southeast-2
aws iam delete-role-policy --role-name afl-run-sql-role --policy-name RunSqlAthena
aws iam detach-role-policy --role-name afl-run-sql-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name afl-run-sql-role
aws athena delete-work-group --work-group afl-readonly-sql --region ap-southeast-2
```

## 8. Open

- Five tests is a small set: grow it to about ten and count tool-choice hits.
- Comma-join not inspected by the parser (IAM covers it).
- Decide on Option C (Code Interpreter) later, only if SQL proves insufficient (modelling, charts).
- Age caveat ("roster age, approximate") still not reliably stated by the model.
