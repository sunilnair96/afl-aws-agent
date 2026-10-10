# 08 — SageMaker notes: running (and improving) the projection on AWS

*Written 2026-10-11 for the learning project. Facts marked **[docs]** come from the AWS pages listed at the end (read on 2026-10-11). **[verify]** means not yet checked or not yet tried. No SageMaker resource has been created yet. Identifiers are placeholders.*

## 1. What SageMaker is, in one paragraph

SageMaker AI is AWS's machine-learning service. For us, the useful part is simple: **it runs your own script on a machine it starts for you, then shuts the machine down.** You give it a script, a container image, an instance type and S3 locations. It copies the input from S3, runs the script, copies the output back to S3, and bills for the time the instance ran. Everything else (notebooks, Studio, endpoints, AutoML, Canvas) is optional and some of it costs money while idle.

## 2. What `build_player_master_list.py` is (read 2026-10-11)

- **Input:** `fantasy_price_roster_{year}.csv` and `supercoach_price_roster_{year}.csv` in the local processed folder.
- **What it does:** takes the roster fields (id, name, team, games, age, height, origin, position, the four eligibility flags) from the Fantasy roster, adds each game's season-start **price estimate**, and outer-merges on `FootywirePlayerId`. A player present in only one roster is kept, with the other game's price blank (SC-only players get their demographic fields back-filled from the SC roster). Output: `masterData/player_master_list_{year}.csv` (810 players for 2026).
- **It does not assign unique IDs.** The unique key is `FootywirePlayerId`, which comes from Footywire's rosters (I checked the 2026 file: 810 rows, 810 distinct ids, ranging 1,611 to 7,481). The AWS side then turns this CSV into the player master Parquet and, in `build_player_identity.py`, builds the combined `PlayerId` (internal ids for historical players, the Footywire id for 2026 players).
- **Two small observations:** (1) the merge is `outer` on the id with no check that ids are unique within each roster; a duplicate id would silently multiply rows. A one-line uniqueness check would make that fail loudly. (2) A line sets `sc_fill["AF_SeasonStartPriceEstimate"] = None` that nothing uses; harmless.
- **Why it matters for SageMaker:** this file is a *data preparation* step, which SageMaker Processing could run, but it is cheap and rarely run, so it is **not** a reason to move to SageMaker. The projection model is.

## 3. What would run on SageMaker, and what it needs

The projection code (local `season_projection.py` and its helpers) uses:

| Item | Local version (from `uv.lock`) |
|---|---|
| Python | 3.11 (`.python-version`) |
| pandas | 3.0.5 |
| numpy | 2.4.6 |
| scikit-learn | **1.7.2** (the project caps it below 1.9 because 1.9.0 broke a training call) |
| scipy | 1.17.1 |
| joblib / threadpoolctl | 1.5.3 / 3.6.0 |

The models are `HistGradientBoostingRegressor` and `RidgeCV` (plus the component and match-level models). The data is small (a season of match data is about 160 KiB as Parquet; the whole bucket is 7.6 MiB). The base projection takes about 4.5 minutes locally.

**What is required to use SageMaker (checklist):**

1. **A region where it is available.** The account works in `ap-southeast-2`. **[verify]** the instance types needed are offered there.
2. **An execution role** that SageMaker can assume (trust `sagemaker.amazonaws.com`), with read and write on the project bucket's prefixes, permission to write CloudWatch logs, and permission to pull the container image. Your own user also needs permission to pass that role (`iam:PassRole`). Best to write a small custom policy, not attach a broad managed one.
3. **Data in S3.** Inputs must be in S3 (Athena or Redshift can also be inputs) **[docs]**. We have that already: `processed/player-match-stats/...` plus the other inputs the local code reads (fixtures, rosters) which are **not in S3 yet**.
4. **A container image.** Either AWS's managed scikit-learn image or your own (section 5).
5. **The SDK or CLI to submit the job.** The SageMaker Python SDK (`pip install sagemaker`) or plain `boto3` / the AWS CLI. In SDK v3 the training classes are replaced by a single `ModelTrainer` with a `SourceCode` object (`source_dir`, `requirements`, `entry_script`) **[docs: search results; not tried]**. How `SKLearnProcessor` maps onto v3 was **not confirmed**.
6. **Service quotas.** New accounts can have low or zero default limits for some instance types. Check Service Quotas before the first job **[verify]**.
7. **A budget guard.** The $-budget alarm we planned, plus checking that no notebook, Studio app or endpoint is left running.

**What you do NOT need:** a SageMaker Studio domain, a notebook instance, or an endpoint. Jobs can be submitted from your own machine. (The Glue console banner offering "SageMaker Unified Studio" is a separate setup; skip it for now.)

## 4. Will it give a different result?

**Short answer: the same, if we run the same code on the same data with the same library versions; different if any of those change, and different by design if we use a SageMaker algorithm instead of ours.**

| Situation | Expected result |
|---|---|
| Same script, same inputs, **same pinned library versions and seeds** | Same numbers. Expect agreement to many decimal places; tiny floating-point differences are possible across CPU types and thread counts. The projections are rounded to one decimal, so a rare 0.1 flip is conceivable. **[verify]** by a parity run (section 6). |
| Same script on the **managed scikit-learn image** | **Likely different**: the image's newest listed version is **scikit-learn 1.4-2** (the repo says it is no longer maintained for new versions; upgrades are tracked in `aws/deep-learning-containers`), while the local project uses **1.7.2** and **pandas 3.0.5**. Different library versions can change defaults and algorithms. **[docs]** |
| Same script, library versions installed at start from a `requirements.txt` | Works if the container's Python can run the pinned versions (pandas 3 needs Python 3.11 or newer; the image's actual Python version was **not** stated on the pages I read) **[verify]** |
| **SageMaker built-in algorithm** (for example XGBoost, Linear Learner) or **Autopilot** | A **different model**, so different numbers. That is a comparison experiment, judged by the same evaluation harness (AF MAE 7.95 is the bar). |
| **Automatic model tuning** over our own `HistGradientBoostingRegressor` settings | Different parameters, so different numbers, found by search. The tuning would have to stay honest about the 2026 hold-out (see the Phase 1 plan, section 3). |

**So SageMaker plays two roles:** (1) an *execution venue* (should not change results; proves we can run the job on AWS), and (2) a *modelling toolbox* (can change results; this is where "compare against the local model" happens).

## 5. Ways to run it (options)

| Option | What it is | Fit for us | Concerns |
|---|---|---|---|
| **A. Processing job, managed scikit-learn image** | Runs a script on a managed machine; you provide the script and pinned `requirements.txt` | Simplest start; good for a "run the existing job" parity test | Image scikit-learn is old (1.4-2 listed); pins may or may not install; the SDK v3 mapping for processing is unconfirmed |
| **B. Training job (`ModelTrainer`), managed image** | Same idea but shaped for model fitting: model artifact to S3, metrics, tuning, Experiments | Right shape for hyperparameter tuning and for tracking runs | Same image-version concern |
| **C. Your own container (ECR), used by A or B** | A Dockerfile with Python 3.11 plus the exact locked libraries, pushed to Amazon ECR | **Best for reproducing the local result exactly** | Needs a Docker build (Docker on this Windows machine, or a cloud build); a one-time setup |
| **D. SageMaker built-in algorithm** | XGBoost and others, no custom code | A real alternative model to compare | Different numbers by design; XGBoost expects its own data format |

**Recommendation:** start with **C + a Processing or Training job** for a parity run, because the aim of that step is "same code, same answer". Only then try D (and tuning) as comparison experiments. If Docker is a blocker, try A with pinned requirements first and treat any difference as a finding.

## 6. A parity test (plan, not run)

1. Local: run the projection (cached tables) and save the output; record the content hash.
2. Put the input data in S3 (the fixtures and rosters are missing from S3 today).
3. Build the image (option C) or prepare the script and `requirements.txt` (option A).
4. Run one job; download the output.
5. Compare the 782 rows for AF and for SC: maximum absolute difference, number of rows that differ at one decimal. **Pass bar to agree** (proposal: identical after rounding, or at most a handful of 0.1 differences explained by floating point).
6. Run it twice to check the job reproduces itself.
7. Record counts, not clock times: job runs, instance type, billable seconds, output size, files, errors.

## 7. Cost and free tier

- **Billing:** Training and Processing are billed by the instance type you choose and how long it runs **[docs]**. The page I read did not state per-second billing for either.
- **SageMaker AI Free Tier [docs]:** for the first 2 months from your first SageMaker resource, including **50 hours of Training on m4.xlarge or m5.xlarge**, 250 hours of notebook/Studio notebooks on ml.t3.medium, and 125 hours of real-time inference on m4.xlarge or m5.xlarge. **Processing is not in that list.**
- **Your account is credit-based** (about $158 of $160 left on 2026-10-10), which may follow different free-tier rules than the page describes. **[verify]** on the Billing Free Tier page before relying on any of this. The job itself is tiny, so even full price is likely small; measure it.
- **Idle-cost traps [docs]:** real-time **endpoints** bill hourly for as long as they run; notebook and Studio compute bills while running; Studio storage accrues until deleted. A batch job we start and let finish has none of these.

## 8. Cheat sheet of the words

- **Processing job:** run a script on managed compute (data prep, evaluation).
- **Training job:** run training; model artifact saved to S3; supports tuning and metrics.
- **Estimator / `ModelTrainer`:** the SDK object that describes a training job (v3 uses `ModelTrainer`).
- **Script mode:** you supply your own script and the container runs it.
- **Execution role:** the IAM role the job runs as.
- **ECR:** the AWS registry that holds your container images.
- **Automatic model tuning:** SageMaker runs many training jobs with different settings and keeps the best.
- **Endpoint:** a model served for live requests; costs while it exists. Not needed for us.

## 9. Sources (read 2026-10-11)

- [SageMaker Processing](https://docs.aws.amazon.com/sagemaker/latest/dg/processing-job.html)
- [Using scikit-learn with SageMaker AI](https://docs.aws.amazon.com/sagemaker/latest/dg/sklearn.html)
- [SageMaker AI pricing and Free Tier](https://aws.amazon.com/sagemaker/ai/pricing/)
- [SageMaker scikit-learn container (GitHub)](https://github.com/aws/sagemaker-scikit-learn-container)
- [Script mode in SDK v3 (AWS blog)](https://aws.amazon.com/blogs/machine-learning/bring-your-own-model-with-amazon-sagemaker-ai-script-mode-in-sdk-v3/) and the [SDK v3 Model Training docs](https://sagemaker.readthedocs.io/en/stable/training/index.html) (seen in search results only)

## 10. Open questions

1. ~~Is Docker available on this machine (for option C)?~~ **Yes (owner, 2026-10-11).** Option C is practical.
2. Which Python version do the managed images actually run, and does `requirements.txt` get installed in a Processing job the way it does in a Training job? **[verify]**
3. Do you want the parity run first, or the XGBoost / tuning comparison first?
4. The fixtures and rosters the projection needs are not in S3, and the **2027 fixture does not exist yet** (owner, 2026-10-11). The parity run therefore uses the 2026 data and fixture; the loader must put those in S3 first.
