# CONTEXT.md: read this first

The running handover note for this project. Anyone who changes the project (a teammate, or Claude in a new
session) reads this file and `git log` first, and adds an entry to **Change log** below with their change.
`CHANGELOG.md` is the release notes for the report; this file is the working history: who changed what, why,
and what is left.

## What the project is

CAP 501 capstone (ATLAS SkillTech University): credit default prediction with fairness constraints, on the
Home Credit Default Risk data (Kaggle). The product is called **Parity**: in the code, the web console, the
README and the progress-review deck. (It was called "Assay" until 16 Nov 2026; use Parity everywhere.)

- `src/` the pipeline (data → features → XGBoost → calibration → fairness policy → explanations → monitoring → gate)
- `api/` FastAPI scoring service, console endpoints, batch scoring
- `web/` React console (built into `web/dist`, served by the API at `/`)
- `models/`, `runs/`, `reports/` committed results; `tests/` 60 tests
- `README.md` how to set up and run everything, in order

## Where it stands (16 Nov 2026)

- **Release 1.3 is the code in this repo.** It implements all eight improvements of the October 2026 plan
  (expected-profit policy, fairness without protected attributes, scorecard challenger, bootstrap intervals and
  group calibration, dropped indefensible features, stress test and Basel capital, reject inference, batch scoring).
  Results: `reports/IMPROVEMENTS_1.3.md`. Gate passes; 60 tests pass with the raw data present.
- **Release 1.2 is kept for comparison** in `reports/baseline_v1.2/` (model, policy, results).
- **Presentation framing:** the progress-review deck (6 slides, "Parity") presents release 1.2 as done and the
  eight 1.3 improvements as *work in progress*, by the team's choice. Keep that in mind before quoting 1.3
  numbers in a presentation.

## Not in the repo (by design)

- Kaggle data and everything derived from it: `data/raw/`, `home-credit-default-risk.zip`,
  `data/processed/*.parquet` (incl. `gate_sample.parquet`, `demo_applicants.parquet`). Download the 7 CSVs
  from Kaggle into `data/raw/`, then run README steps 1, 2, 15 and 16. Without them the gate shows G3 as
  NOT RUN (a skip, not a failure) and the console's applicant picker is empty.
- `.venv/`, `web/node_modules/`, `logs/` (decision log), `.claude/`.
- Outside the repo: the Word/slide report, the progress deck and the Claude Docs report live in the team's
  own accounts, not here.

## Setup that worked (macOS)

`brew install python@3.12 libomp`, then `python3.12 -m venv .venv && .venv/bin/pip install -r requirements.txt`.
Full retrain ≈ 25 minutes; `src/fairness.py` reductions are cached in `runs/reductions_cache_v2.pkl`.

## Change log (newest first)

Add an entry for every change: date, who, what changed, why, how it was checked, what is left.

### 2026-11-16 · Anish (with Claude) · renamed Assay to Parity
- The product name is now Parity everywhere: README title, web console (nav, footer, browser tab:
  `web/src/components/Chrome.tsx` PRODUCT, `web/index.html`), package name `parity-credit-console`.
- Rebuilt `web/dist`; tests and gate re-run.

### 2026-11-16 · Anish (with Claude) · repository created
- Put the project under git and pushed it to a public GitHub repository (https://github.com/Anish-Biradar/cap501) so the team can use the link.
- Added this file and `.claude/` to `.gitignore`. No code changes.

### 2026-10-27 · Anish (with Claude) · progress deck, outside the repo
- Built a 6-slide progress deck in the style of the original 16-slide "CAP501 Deck": release 1.2 results, the
  eight improvements shown as in progress. Renamed the product to "Parity" in the deck.
- Wrote a simple-language speaker script for slides 13 to 16 of the original deck.

### 2026-10-22 · Anish (with Claude) · release 1.3: the eight improvements
- Implemented every item of `CAP501_improvement_plan.pdf`; details in `CHANGELOG.md` (1.3) and
  `reports/IMPROVEMENTS_1.3.md`. Key files: `src/policy.py` (one decision rule everywhere),
  `src/profit_policy.py` (writes the deployed `models/fair_policy.json`), `src/scorecard.py`,
  `src/calibration_groups.py`, `src/stress_capital.py`, `src/reject_inference.py`, `api/batch.py`,
  `web/src/pages/Batch.tsx`.
- Retrained from raw data; gate passes on bootstrap upper bounds; 60 tests pass.
- Findings worth knowing: in a moderate recession the fixed fairness offsets let the age gap reach 6.6 points
  (refit needed); recourse now favours longer terms (more interest raises the cut-off); reject inference gave no
  reliable gain.
- Environment: the `.venv` that came with the folder was from another machine; rebuilt with Homebrew Python 3.12.
- A Claude Docs report "CAP 501 Release 1.3: Improvements Report" summarises each improvement.

### 2026-10-22 · team · release 1.2: rebuilt from raw data, web console
- See `CHANGELOG.md` 1.2.

### 2026-10-14 · team · release 1.1: review fixes
- See `CHANGELOG.md` 1.1.

## Open items

- Update the written report and slides with the 1.3 numbers when the team decides to present them as done.
- The fairness offsets must be refitted when PDs shift (stress-test finding).
