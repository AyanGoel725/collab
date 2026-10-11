# Standalone Google Colab workflow

Only the notebook and your input file need to be uploaded. Each notebook embeds
the Python implementation, Snowflake SQL, exact scientific dependency pins and
published reference/model artifacts for a separately labeled replay. There is no repository clone or separate
Python-module upload. Use a **CPU runtime** and **Runtime > Run all**.

## 1. Clean the CSV and train

Open `01_Cleaning_and_Model_Training.ipynb` in Google Colab. Select the original
`Case_Study_2_CLV_Predictive_Analytics.csv` in the upload prompt. You can also
upload the CSV in Colab's Files pane before running; a single CSV in `/content`
is detected automatically. The filename can differ, but preserve data and row
order to reproduce the submission.

The notebook installs packages in an isolated environment, audits and cleans the
data, creates the 15 features, and trains the full XGBoost/Random Forest workflow:
150/200 trees, five outer account-grouped folds, three inner grouped calibration
folds, seed 42, reference date 2026-10-01 and fixed raw ensemble weights 0.6/0.4.
Weighted/unweighted and raw/sigmoid/isotonic candidates, logistic and training
prevalence baselines are retained. Missing targets remain inference cases.

Download **CLV_TRAINING_ARTIFACTS.zip** when the final cell runs. It contains your
raw CSV, cleaning tables/audit, feature names, folds, all OOF predictions, the
final freshly fitted bundle, source/SQL and a manifest with artifact hashes.
On the unchanged case-study input it also includes the separately identified
frozen training run used by the published report. Frozen models are not used in
new model fitting.

## 2. Run evaluation, SHAP and scoring

Open `02_Backtesting_SHAP_and_Scoring.ipynb` and select **Run all**. In a fresh
runtime, upload **CLV_TRAINING_ARTIFACTS.zip**, not just the CSV. The ZIP already
contains the CSV and fitted models, so no second CSV upload or retraining is
needed. If using the same runtime/workspace, saved training artifacts are found
automatically. Use only a training ZIP produced by your own first notebook.

The second notebook produces:

- Metrics for all 21 candidates, paired account-bootstrap intervals (1,000
  repetitions; seed 20261011), proxy-cohort bias and signed historical exposure.
- Brier, ROC/precision-recall/calibration, cohort and paired-interval charts.
- Four XGB component SHAP plots, complete 15-feature explanation maps and
  raw-margin additivity/probability reconstruction checks.
- A validated 25-column `CLV_SCORED_OUTPUT.csv` for all arrays, input snapshot,
  training/scoring manifests and output hashes.
- Account array-event summaries, OOF coverage summaries, current-load capacity
  diagnostics, per-array fifteen-year constant-rate scenarios and discounted
  unit-cash-flow sensitivities.
- All five warehouse scripts and a generated `snowflake_handoff.sql` containing
  matching six-argument ingestion CALLs for fresh and published-replay scores.

On matching original input, it also **reproduces the published report** in a
separate branch: metrics are recomputed from the genuine frozen OOF predictions,
and your uploaded CSV is scored with the original saved model. The replay's
results are under `outputs/published_validation/`, `outputs/published_scores/`
and `outputs/published_shap_plots/`. Its model/evidence are replayed, **not newly
trained**. Fresh retraining remains under `outputs/runs/<new_run_id>/`,
`outputs/score_runs/`, `outputs/analysis_plots/` and `outputs/shap_plots/`.

Download **CLV_ANALYSIS_AND_SQL.zip** from the final cell. It contains the
complete training/evaluation/scoring handoff, plots, analytics and SQL. Save both
ZIPs before the runtime expires. Each fresh training/scoring run has new IDs and
UTC timestamps; the original repository outputs are preserved.

## Environment and numerical reproduction

The compute environment uses Python 3.12 and the original pinned scientific
library versions. Packages are installed separately from Colab's own imported
packages, and computations run in that environment without a kernel restart.
If the host uses another Python version, setup uses an isolated `uv==0.8.22`
bootstrap to obtain Python 3.12.10. Internet access is needed for the first
installation; GPU/TPU hardware is unnecessary.

This follows Colab's recommendation to install specific library versions for
reproducibility. Its optional runtime-version selector is documented in the
[official runtime FAQ](https://research.google.com/colaboratory/runtime-version-faq.html).
Both notebooks embed `requirements-colab.txt`; uploading that file is optional.
The Windows Jupyter distribution lock is not installed into Linux.

The embedded references and frozen model are tied to source revision
`c32ab4de1b813b7f555cfad7ef03bd8d56160ba1`. Fresh models are always trained from
the uploaded CSV. Linux XGBoost retraining was observed to produce different
predictions from Windows despite the same package versions, seeds and folds;
package pins alone do not guarantee identical sampled trees across platforms.
These differences are recorded as `FRESH_RETRAINING_DIFFERS_FROM_PUBLISHED`
without changing or disguising the fresh results. The **published replay** is
the strict reproduction path for the existing report. Its checks compare keyed OOF
probabilities, fold assignments, candidate metrics, intervals, cohorts, buckets,
scored values, full SHAP contributions and five analytics tables. Absolute
tolerances are 0.000002 for predictions/metrics/scored values and 0.00001 for
SHAP/account/scenario values; categorical values and row keys must match exactly.
Published-replay failures stop execution and leave a per-column JSON diagnostic. Different or
reordered input is explicitly marked `NOT_APPLICABLE_DIFFERENT_INPUT`, rather
than claimed to reproduce published results.

UUIDs, UTC timestamps, platform metadata, serialized-model/file hashes and image
bytes are not cross-platform equality targets. Every new run's own hashes are
checked internally. Runtime versions are recorded with the training handoff.
See `VALIDATION.md` for the actual checks performed on these notebooks.

## 3. Run SQL in Snowflake

Select your existing sandbox role, warehouse, database and schema. Then run:

1. `sql/setup.sql`.
2. `sql/ddl_clv_scored_output.sql`.
3. `sql/stored_proc_scoring_pipeline.sql`.
4. Upload the generated scored CSV to `CLV_STAGE/<score_run_id>/` and run the
   matching handoff from the analysis ZIP: `outputs/snowflake_handoff.sql` for
   fresh scores, or `outputs/published_snowflake_handoff.sql` for replay scores.
5. Exercise `sql/sandbox_acceptance.sql` and inspect the ledger/failure results
   with `sql/insert_scored_output.sql`; retain query IDs.

The handoff includes a commented SnowSQL PUT example whose local path must be
substituted. Snowflake access and execution are separate from the notebooks.
Serialize warehouse writers; local static checks do not certify live SQL,
transaction behavior or warehouse performance.

## Interpretation limits

The supplied CSV supports grouped cross-sectional evaluation, not chronological
historical back-testing. It has no observation/availability dates or verified
label maturity. Account bounds are summaries of array events, not cancellation
probabilities. Capacity arithmetic and supplied expansion tags are not learned
future-expansion outcomes. Expiry/expansion cohorts are proxies, not products.
Fifteen-year results are explicit constant-rate scenarios, not validated
long-term renewal or measured CLV; unit cash flows are not dollars. The guarded
temporal module is embedded for future genuine dated outcomes. These limits are
preserved from the original submission.

## Maintain or verify the notebook package

The original `src/`, `sql/`, three evidence-review notebooks and frozen outputs
remain separate. Regenerate this standalone package from the repository with:

```text
python tools/build_collab_notebooks.py
```

An end-to-end verifier executes both notebooks in different fresh workspaces,
transfers only the training ZIP to the second, and checks all numerical parity
results without reducing tree counts, folds or bootstrap repetitions. Browser
file picking is mocked using the `files.upload()` contract; the notebooks' real
upload branches execute. The verifier strictly checks published replay while
recording fresh cross-platform training differences:

```text
python tools/verify_collab_notebooks.py --output validation.json
```

The verifier controller needs `nbformat`, `nbclient` and `ipykernel`. An optional
`--python /path/to/pinned/python` uses an existing matching compute environment;
omit it to test the fresh dependency installer. Notebook-only environment
variables beginning `CLV_COLLAB_` are for automated verification and can remain
unset during normal Colab use.
