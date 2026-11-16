# Changelog

## 1.3 (22 Oct 2026): the eight improvements of the October plan

Retrained from the raw data; every number in the report changes. `reports/IMPROVEMENTS_1.3.md` has the results and
what to change in the report and slides. The 1.2 release is kept in `reports/baseline_v1.2/` for comparison.
Gate: PASSED (G1 to G3, every requirement on its 95% bound). Tests: 60 pass with the raw data present.

### Model
1. **Expected profit instead of a fixed 5:1 cost ratio** (`src/policy.py`, `src/profit_policy.py`). Approve when
   (1 − PD) × m × EAD − PD × LGD × EAD − c is at least the group's offset. LGD 45% (Basel CRE32.6), m from each loan's
   own instalment and the 3.33% monthly rate implied by 312,536 approved previous cash loans, 25% of it kept, c = 5,000.
   Cut-offs now depend on the loan (median 18% for the smallest fifth of loans, 29% for the largest). The deployed
   policy earns 2.2% more realised profit on test than the refitted 5:1 group cut-offs, and more under all 18
   LGD × margin assumptions (2.0% to 10.0%); gaps 2.1 and 2.2 points.
2. **Fair without protected attributes at decision time** (`src/fairness.py`). ExponentiatedGradient now minimises
   the 5:1 cost (`ErrorRate(costs=...)`), with a stronger base learner, 80,000 applicants and 50 iterations. At
   eps = 0.01 it approves 90.2% with gaps of 1.0 (gender) and 3.8 points (age), upper bounds 1.6 and 4.4, at a cost of
   0.330 against 0.328 for the group cut-offs. It is on the frontier as the policy that needs no gender or age.
3. **Scorecard challenger** (`src/scorecard.py`). WoE bins + logistic regression, 34 variables, 600 points at 50:1,
   PDO 20. Test AUC 0.763 against 0.785, ECE 0.002, gaps 1.0 / 1.2 with group cut-offs; passes the same gate. Points
   table in `reports/scorecard_points.csv`.
4. **Uncertainty and calibration by group** (`src/metrics.py: bootstrap`, `src/gate.py`, `src/calibration_groups.py`).
   The gate checks every requirement on the unfavourable end of a 1,000-resample interval. Men 35 and over are
   under-predicted (7.9% against 9.3%, outside the 95% interval); a per-group log-odds shift fixes all four groups and,
   refitted, lowers the age gap from 2.2 to 1.0 points for 0.1% of profit. Kept as an analysis, not deployed.
5. **Indefensible features removed** (`src/prepare.py: NOT_DEFENSIBLE`). Hour and weekday of application, date of the
   last phone change, ID document date, contact-detail flags, 20 document flags, social-circle defaults and opaque
   previous-application codes: 449 features to 401. Calibrated AUC 0.7849 to 0.7853. The service ignores these fields
   if sent (`data_quality.ignored_fields`), and "Time of application" and "Identity documents and contact details"
   can no longer appear as reasons.
6. **Stress test and capital** (`src/stress_capital.py`). Basel IRB other-retail capital: the deployed book ties up
   1.39 bn, 2.1% more than the same rule without fairness offsets. Under a moderate recession (mean PD 8.1% to 12.2%)
   the fixed offsets let the age gap reach 6.6 points: the policy must be refitted in a downturn.
7. **Reject inference** (`src/reject_inference.py`). 64,075 refused applications: an accept/reject classifier
   separates them with AUC 0.92, and applicants with a past refusal default at 10.3% against 7.0%. Parcelling and
   augmentation change test AUC by −0.002 and +0.001: no reliable gain, reported as a limitation.

### Website
8. **Batch scoring** (`api/batch.py`, `web/src/pages/Batch.tsx`). Upload a CSV; each row is validated and scored by
   the same code as `POST /score` and logged with `source: "batch"`; rejected rows are listed by line with the
   problem in plain words; approval by group; results CSV to download; a sample file with four broken rows.

### Also
- `src/policy.py` is the single decision rule for training, the gate, the service, the console and the scripts.
- `api/app.py`: `decide()` shared by `/score` and the batch; responses carry `expected_profit`.
- The Model page shows the gate's intervals, the economics, the challenger, the no-protected-attribute policy,
  calibration by group, the stress test and reject inference. The Fairness page explains the new policy.
- New tests: `tests/test_policy.py`, `tests/test_batch.py`, and gate, API and console tests for the above.
- `requirements.txt`: `optbinning==1.0.0`, `scipy`.

## 1.2 (22 Oct 2026): raw-data verification and web console

### Reproduced from the raw data
- `data.py` and `prepare.py` rebuilt from the Kaggle files reproduce the committed work exactly: 307,507 applicants,
  identical split checksums, identical EDA, proxy audit and transformation log, the same 449 features in the same
  order, and a difference of 0.0 between the committed model's predictions and `runs/pred_test.parquet` on all
  46,127 test applicants.
- The scoring service, given real applicants as raw forms, returns exactly the saved probabilities (3,000 checked).
  Its validation rejects 14 of all 307,507 applicants: the 12 with no instalment and the 2 with no family size.
- Gate check G3 now runs: `data/processed/gate_sample.parquet` (1,999 test applicants) re-scored with difference 0.0.
- `runs/r7_latency.json` re-measured on 1,000 real applicants in the raw-form format: p95 8.3 ms, none rejected,
  referral rate 8.3% (identical to the original run).
- All 43 tests pass with the raw data present, including the real-data leakage test.

### Web console
- `web/`: landing page, Assess, Fairness and Model pages, served by FastAPI at `/` from `web/dist`.
- `api/console.py`: applicants, recourse, fairness simulator and curve, frontier, model card, live gate, decision log.
- `src/make_demo_applicants.py`: 400 real test applicants for the picker, balanced across group, decision and outcome.
- `src/gate.py`: `collect()` returns the checks so the console can show them; output is unchanged.
- `api/features.py`: validation now reports out-of-range values together with missing fields in one response.
- `tests/test_console.py`: 10 tests, including that recourse options really get approved when re-scored and that
  every listed applicant scores exactly as shown.

## 1.1 (14 Oct 2026): review fixes

Four defects found in review, fixed and covered by tests (`pytest -q tests`: 32 pass, 1 skipped without raw data).

### 1. Continuous integration and the container could not install
`xgboost==3.4.1` and `shap==0.53.0` publish no wheels for Python 3.11, which `gate.yml` and the Dockerfile used.
Both now use Python 3.12. The workflow also builds the container, so a broken Dockerfile fails the pull request.

### 2. The gate graded saved predictions, not the committed model
`gate.py` read `runs/pred_test.parquet` and never opened `models/xgb.json`, so a swapped model still passed.
New integrity checks run before the requirement checks:
- **G1** `models/manifest.json` (written by `train.py`) binds the model, calibrator, feature list and saved predictions by SHA-256. Any change to one without the others fails.
- **G2** the committed calibrator must reproduce the saved calibrated PDs exactly.
- **G3** the committed booster re-scores a frozen sample of test applicants (`src/make_gate_sample.py`; needs the raw data once, then the sample is committed).
- **POL** a policy that leaves any gender x age group without a cut-off now fails, instead of silently refusing that group.

### 3. The scoring service approved anything
An empty request returned PD 0.0, "approve", band A; so did negative income and an external score of 50.
Root cause: the service accepted the 449 encoded columns and filled anything absent with NaN, including one-hot and
flag columns that were never missing in training, so the trees followed branches they had never learned.
The service now accepts the raw application form and builds the row itself (`api/features.py`):
- required fields, ranges, allowed category values, unknown keys, protected attributes and server-derived ratios are all validated; HTTP 422 lists every problem at once;
- one-hot columns, missing-value flags and ratios are built with the same formulas as `src/data.py` and `src/prepare.py`; a test checks the two agree;
- `GET /schema` documents the contract for any client or UI.

### 4. Referral reasons were not usable as reasons
A referred applicant was shown features that *lowered* their risk, and the same concept twice
(`EXT_SOURCE_MEAN` and `EXT_SOURCE_2`). SHAP values are now summed per concept (they are additive) and the response gives:
- `adverse_action_reasons`: on a referral only, up to four concepts that raised the risk, largest first; a history table with no record says "none on file";
- `top_factors`: the five largest concepts in either direction, for the loan officer.

### Also
- Every decision is logged to `logs/decisions.jsonl` with model, calibrator and policy hashes and an input hash (EU AI Act Art. 12 record-keeping, without storing personal data).
- `src/latency.py` sends real test applicants in the new raw-form format. Synthetic re-measurement: p95 5.4 ms (budget 50 ms).

### Report and slides to update
- Section 7: request/response format, validation, decision log; re-run `src/latency.py` once the raw data is available.
- Section 8: the gate now checks the model's integrity (G1 to G3); replace the gate table with `reports/gate_demo_output.txt`.
- Section 2: the CI claim is now true once the repository is pushed with the new workflow.
- Section 11, Art. 12 row: point to the decision log.
