# Release 1.3: results of the eight improvements

Everything below was produced by the pipeline in `README.md` (steps 1 to 18) on 22 October 2026, from the raw Kaggle
data, on the same frozen splits as 1.2 (identical SHA-256 checksums). Test split: 46,127 applicants, never used for
fitting. The 1.2 release is kept in `reports/baseline_v1.2/` for comparison.

## Headline numbers that changed

| Measure (test split) | 1.2 | 1.3 |
| --- | --- | --- |
| Features | 449 | 401 |
| ROC-AUC, calibrated model (95% CI) | 0.785 | 0.785 (0.778 to 0.793) |
| ECE (95% CI) | 0.003 | 0.003 (0.002 to 0.006) |
| Deployed policy | group cut-offs at 5:1 | expected profit + group offsets |
| Approval rate, deployed | 91.3% | 91.3% |
| Default rate among approved, deployed | 5.9% | 6.2% |
| Gender gap, deployed (95% CI) | 0.9 pts | 2.1 pts (1.6 to 2.6) |
| Age gap, deployed (95% CI) | 0.8 pts | 2.2 pts (1.6 to 2.8) |
| Gap across four groups, deployed | | 3.3 pts |
| Single cut-off 1/6: gender / age gap | 3.3 / 10.3 pts | 3.6 / 11.6 pts |
| Recourse found for referred applicants | 85% | 96% |
| p95 scoring latency, explanation included | 8.3 ms | 2.4 ms |
| Tests | 43 | 60 |

The deployed gaps are larger than in 1.2 but well inside the 5-point limit at the top of their intervals. They are
the price of maximising profit: the 5:1 group cut-offs refitted on the 1.3 model still reach 0.2 and 1.2 points, but
earn 2.2% less.

## 1. Expected profit instead of a fixed 5:1 cost ratio

**Rule.** Approve loan *i* when its expected profit is at least its group's offset *h_g*:

E[profit] = (1 − PD) · m_i · EAD − PD · LGD · EAD − c,  so the PD cut-off is (m_i·EAD − c − h_g) / ((m_i + LGD)·EAD).

| Assumption | Value | Source |
| --- | --- | --- |
| EAD | loan amount (AMT_CREDIT) | |
| LGD | 45% | Basel framework CRE32.6: foundation-IRB supervisory LGD, senior unsecured. Sensitivity 25% to 75% |
| Gross lifetime interest | from each loan's instalment/amount at a 3.33% monthly rate | median implied rate of 312,536 approved previous cash loans (`previous_application`) |
| Share of interest kept, m | 25% | stated assumption (funding, running costs, interest lost on defaults). Sensitivity 15% to 40% |
| Fixed cost c | 5,000 per loan | stated assumption, about 1% of the median loan |

Offsets are fitted on the validation split to maximise realised profit with every group's repayer approval within a
3-point band, and may only lower a group's bar (levelling up): Women 35+ −852, Women <35 −7,756, Men 35+ −691,
Men <35 −13,654.

| Policy (test) | Approved | Default among approved | Realised profit per applicant | Gender gap | Age gap |
| --- | --- | --- | --- | --- | --- |
| Single cut-off 1/6 (5:1) | 86.7% | 5.1% | 81,192 | 3.6 | 11.6 |
| Group cut-offs, 5:1 (1.2 policy, refitted) | 90.5% | 5.8% | 81,907 | 0.2 | 1.2 |
| Expected profit, no fairness constraint | 87.7% | 6.0% | 83,780 | 4.5 | 12.0 |
| **Expected profit + group offsets (deployed)** | **91.3%** | **6.2%** | **83,700** | **2.1** | **2.2** |

- The cut-off now depends on the loan: median 18% for the smallest fifth of loans and 29% for the largest; 16% for
  short terms and 35% for long ones.
- Fairness costs 0.1% of profit (83,780 to 83,700 per applicant).
- Across all 18 combinations of LGD (25% to 75%) and interest kept (15% to 40%), the fair expected-profit policy earns
  2.0% to 10.0% more than the 5:1 group cut-offs, and the larger gap never exceeds 3.3 points
  (`runs/expected_profit_sensitivity.csv`).
- Side effect for the recourse section: a longer term earns more interest, so it raises the cut-off. Most recourse is
  now "the same loan over a longer term" (median loan cut 0%). A reviewer could see that as approving people who pay
  more interest; say so.

## 2. Fairness without gender and age at decision time

The 1.2 ExponentiatedGradient run minimised plain error rate, for which approving almost everyone is optimal at a 92%
repayment rate, hence 99.4% approvals. Fixed: it now minimises the same 5:1 cost (`ErrorRate(costs=...)`), uses an
XGBoost base learner (300 trees, depth 4) on 80,000 applicants, runs 50 iterations, and decides from the classifier
mixture's vote rather than a recalibrated hard label.

| Policy (test, cost at 5:1) | Uses gender/age to decide | Approved | Gender gap | Age gap | Cost |
| --- | --- | --- | --- | --- | --- |
| Single cut-off | no | 86.7% | 3.6 | 11.6 | 0.318 |
| Same-budget reference, no constraint | no | 87.6% | 3.5 | 10.3 | 0.323 |
| **ExponentiatedGradient eps = 0.01** | **no** | **90.2%** | **1.0 (to 1.6)** | **3.8 (to 4.4)** | **0.330** |
| ExponentiatedGradient eps = 0.03 | no | 90.1% | 1.7 | 5.9 | 0.330 |
| Group cut-offs at 5:1 | yes | 90.5% | 0.2 | 1.2 | 0.328 |

At eps = 0.01 the policy passes NFR3 at the top of its interval without seeing a protected attribute at decision time,
at about the cost of the group cut-offs. Its mixture score ranks poorly as a probability (AUC 0.66) because it is a vote
share; it is a decision rule, not a PD, so it cannot feed the expected-profit rule or the capital calculation. That is
the trade-off for the report: the deployed policy keeps group offsets for profit and calibrated PDs, and this policy
is the documented alternative where disparate treatment is prohibited.

## 3. Scorecard challenger

WoE bins (optbinning, monotonic where the data allow) and logistic regression, 34 variables, 600 points at 50:1 odds of
repaying, 20 points to double them. Same splits, isotonic calibration on valid, same gate.

| Test | XGBoost (champion) | Scorecard |
| --- | --- | --- |
| ROC-AUC | 0.785 | 0.763 |
| ECE | 0.003 | 0.002 |
| Approved (group cut-offs, 5:1) | 90.5% | 89.5% |
| Gender / age gap | 0.2 / 1.2 | 1.0 / 1.2 |
| Cost per applicant at 5:1 | 0.328 | 0.345 |
| Gate | pass | pass |

The champion gains 0.022 AUC and 5% lower cost; the scorecard is fully explainable point by point
(`reports/scorecard_points.csv`). Scores on test run from 498 (1st percentile) to 632 (99th), median 571.

## 4. Uncertainty and calibration within groups

**Intervals.** `metrics.bootstrap`: 1,000 resamples of the test split, fixed seed. The gate fails on the lower bound of
AUC and the upper bound of ECE and of each gap. On the 1.2 policy this gives an age gap of 0.8 points (0.2 to 1.3), as
the plan expected. The deliberately unfair commit still fails (age gap 11.6, upper bound 12.4).

**Calibration by group (test).**

| Group | Predicted | Observed (95% CI) | After per-group correction |
| --- | --- | --- | --- |
| Women 35+ | 6.5% | 5.9% (5.6 to 6.2) | 6.1% |
| Women <35 | 10.9% | 10.4% (9.7 to 11.1) | 10.2% |
| Men 35+ | **7.9%** | **9.3% (8.7 to 9.9)** | 9.3% |
| Men <35 | 11.8% | 11.8% (11.0 to 12.7) | 11.8% |

Men 35 and over are under-predicted and women 35 and over over-predicted, both outside the observed interval. A
per-group log-odds shift fitted on valid (four parameters) fixes all four. Refitting the deployed policy on the corrected
PDs changes the gaps from 2.1 / 2.2 to 2.2 / 1.0 points and profit by −0.1%. Without fairness offsets the correction
alone does not close the gaps (age 12.0 to 10.7). Not deployed: setting the PD by gender and age has the same legal
status as setting the cut-off by them.

## 5. Features a lender should not use

Removed: hour and weekday of application (and of previous applications), date of the last phone change, ID document
date, the mobile, home phone, work phone, email and mobile-reachable flags, the 20 document flags, the four
social-circle default counts, seller area codes and "last application that day". 449 to 401 features.

| Test | 1.2 (449) | 1.3 (401) |
| --- | --- | --- |
| AUC, application table only | 0.765 | 0.762 |
| AUC, all tables, unweighted | 0.785 | 0.782 |
| AUC, tuned and calibrated | 0.785 | 0.785 |
| PR-AUC, calibrated | 0.271 | 0.269 |
| ECE | 0.003 | 0.003 |
| Age gap, single cut-off | 10.3 | 11.6 |

The accuracy cost is nil for the deployed model. The single cut-off's age gap grows slightly, which the fairness
policy absorbs. "Time of application" and "Identity documents and contact details" can no longer appear on a
referral notice. The service ignores these fields if a client still sends them.

## 6. Stress test and capital

Basel IRB risk-weight function for other retail exposures (CRE31.15), LGD 45%, PD floor 0.05%. Stress: a log-odds
shift that scales the portfolio's mean PD by 1.5 (moderate) or 2.0 (severe, with downturn LGD 55%).

| Capital (bn) / larger gap (pts) | Single cut-off | 5:1 group cut-offs | Profit, no fairness | Profit + offsets (deployed) |
| --- | --- | --- | --- | --- |
| Baseline (mean PD 8.1%) | 1.25 / 11.6 | 1.32 / 1.2 | 1.36 / 12.0 | 1.39 / 2.2 |
| Moderate (12.2%) | 1.41 / 17.6 | 1.49 / 4.0 | 1.53 / 17.6 | 1.57 / **6.6** |
| Severe (16.3%) | 1.88 / 21.4 | 1.99 / 5.5 | 2.04 / 21.6 | 2.09 / **10.9** |

- The fairness offsets add 28 million of capital (2.1%) for 391 million more lending; capital per unit lent rises from
  5.26% to 5.29%.
- In a moderate recession the deployed policy's age gap exceeds 5 points, because its offsets are fixed in currency.
  The policy must be refitted when PDs shift, which is what the monitor's "fairness re-audit" action is for. Say this
  in the monitoring section.

## 7. Reject inference

- 290,678 refused previous applications against 1,036,781 approved. Refused applications were for larger loans (median
  183,000 against 102,000) and more often cash loans (57% against 30%). Main reason codes: HC 60%, LIMIT 19%, SCO 13%.
- Applicants with a past refusal defaulted on their later loan at 10.3%, against 7.0% for the rest.
- Reject sample: each train applicant's latest refused application, on their own application-level features (64,075
  rows; 13,826 for test). An accept/reject classifier separates them from accepts with AUC 0.92.
- The accepts-only model would approve 87.5% of them at 1/6: its view of rejects is an extrapolation.

| Application-level model | Test AUC | Test ECE | Rejects' mean PD |
| --- | --- | --- | --- |
| Accepts only | 0.762 | 0.003 | 8.1% |
| Parcelling (rejects at 2x their band's bad rate) | 0.760 | 0.002 | 11.0% |
| Augmentation (accepts weighted by 1 / P(accept)) | 0.763 | 0.001 | 8.1% |

Neither method gives a reliable gain on outcomes we can observe. Parcelling raises the rejects' PD only because its
uplift assumes they are worse. Report it as a known limitation that was investigated. The reject sample's construction
(personal attributes assumed unchanged; these people were later accepted) is itself a limitation.

## 8. Batch scoring

`POST /api/batch` and the Batch page: up to 5,000 rows per file. Each row is parsed, validated by the same contract as
`POST /score`, scored by the same `decide()` and logged with `source: "batch"`. The sample file (200 real test
applicants and four broken rows) gives four rejected rows, each with a precise reason on the right line: income
missing; "four hundred thousand" is not a number; loan type "Mortgage" not allowed (allowed: Cash loans, Revolving
loans); age 12 must be between 18 and 100. Results download as CSV with PD, decision, cut-off and reasons per row.

## Report and slides to update

- **Method, decision policy:** replace the 5:1 assumption with the expected-profit rule, its assumptions table and the
  sensitivity result. Keep 5:1 as the comparison.
- **Fairness:** the new deployed gaps with intervals; the no-protected-attribute alternative and its trade-off; the
  frontier figure (`reports/figures/frontier.png`).
- **Model:** feature removal (449 to 401) and its zero cost; the scorecard challenger table.
- **Validation and gate:** bootstrap intervals and the upper-bound rule; replace the gate table with
  `reports/gate_demo_output.txt`.
- **Calibration:** the per-group table and the decision not to deploy the correction.
- **Risk and capital:** the stress-test table and the recession finding (`reports/figures/stress_capital.png`).
- **Limitations:** reject inference; recourse now favours longer terms; offsets must be refitted when PDs shift.
- **System:** batch scoring; latency 2.4 ms p95; 60 tests.
