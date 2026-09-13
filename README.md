# KELVORA Behavioral Segmentation

## Purpose

This analysis groups 620 prescribers into behavior-based segments using access, treatment-setting, dosing, product-preference, and persistence measures. It then estimates two addressable financial opportunities for each segment:

1. Recovering patients lost to failed prior authorization.
2. Retaining patients who discontinue early because of loss of response or tolerability.

The notebook exports an HCP-level segmentation master and displays a segment-level opportunity table. A priced funding recommendation is not generated automatically; the opportunity table must currently be transferred into the FY27 funding recommendation as a downstream step.

## Files

- `Kelvora_Growth_Question.ipynb` — production analysis notebook.
- `KELVORA_Case_Data.xlsx` — source workbook located at data folder.
- `Kelvora_HCP_Seg_Master.xlsx` — HCP-level output created by the notebook.

The source workbook must contain these four tabs:

- `hcp_master`
- `hcp_monthly_activity`
- `patient_therapy_cohort`
- `economics_reference`

For the supplied data, their respective dimensions are 620 HCPs, 14,880 HCP-month records, 3,942 patient referrals, and 12 economics-reference rows.

## Requirements

- Python 3
- `pandas`
- `numpy`
- `scikit-learn`
- `openpyxl`

Google Colab normally provides these packages. A local Jupyter environment may require installation.

## How to run

1. Open `Kelvora_Growth_Question.ipynb` in Jupyter or Google Colab.
2. Make sure `file_name` points to `KELVORA_Case_Data.xlsx`.
   - The current notebook uses `/KELVORA_Case_Data.xlsx`, an absolute path.
   - If the workbook is beside the notebook, change the line to `file_name = "KELVORA_Case_Data.xlsx"`.
   - In Colab, upload the workbook and use its actual Colab path.
3. Restart the kernel or runtime and run all cells from top to bottom.
4. Review the segment profile and opportunity tables.
5. Retrieve `Kelvora_HCP_Seg_Master.xlsx` from the working directory.

The final download commands use `google.colab.files`. In local Jupyter, the preceding `to_excel(...)` command still creates the file, but the Colab-specific import and download commands should be removed or skipped.

## Execution flow

### 1. Load the data

The notebook reads all four workbook tabs into named pandas DataFrames:

- `hcp_m` from `hcp_master`
- `hcp_mo_act` from `hcp_monthly_activity`
- `patient` from `patient_therapy_cohort`
- `enco` from `economics_reference`

### 2. Load economics inputs

The following inputs are read from `economics_reference`:

- Target maintenance dose in grams per kilogram per month
- Average patient weight
- Net revenue per gram
- Contribution-margin percentage
- Assumed PA recovery rate

The dose target and patient weight are first used during dose-feature engineering. Revenue, margin, and the PA recovery rate are subsequently used during opportunity sizing.

### 3. Engineer the seven clustering features

Patient referrals and monthly HCP activity are aggregated to one row per HCP. The final clustering features are:

| Feature | Definition |
|---|---|
| `median_days_to_infusion` | Median referral-to-first-infusion time among patients who initiated therapy |
| `pct_hopd` | Share of an HCP's patient records assigned to hospital outpatient care |
| `pct_home_infusion` | Share of an HCP's patient records assigned to home infusion |
| `pa_failure_rate` | Share of PA-required referrals with an outcome of `Abandoned - no response` or `Denied - not appealed` |
| `dose_vs_label_ratio` | Average actual monthly dose per kilogram divided by the label maintenance target |
| `kelvora_start_share` | KELVORA starts divided by KELVORA plus competitor starts across monthly activity records |
| `early_discontinuation_rate` | Share of evaluable initiated patients who discontinued before six months |

For the six-month persistence feature:

- A known discontinuation is evaluable because the outcome is observed.
- An ongoing patient observed for at least six months is evaluable and is not an early discontinuation.
- An ongoing patient observed for fewer than six months is excluded because the six-month outcome is not yet known.

This leaves 2,601 evaluable patients among 3,441 initiated patients.

### 4. Merge and impute

All feature tables are left-joined onto `hcp_master`, ensuring that all 620 HCPs remain in the analysis. Missing feature values are filled with the cohort median because k-means cannot accept missing values.

Before each fill, the notebook creates a corresponding `*_was_imputed` indicator in the working feature table. These indicators are used for auditability during analysis but are not currently included in `Kelvora_HCP_Seg_Master.xlsx`.

### 5. Standardize and select k

The seven features are standardized with `StandardScaler` so that features measured in days do not dominate features measured as rates or ratios.

The notebook tests `k = 3` through `k = 7` using:

- `n_init = 10`
- `random_state = 42`
- Silhouette score for cluster separation
- A minimum cluster-size floor of 5% of the 620-HCP cohort

| k | Silhouette | Smallest cluster share | Passes 5% floor |
|---:|---:|---:|:---:|
| 3 | 0.239 | 16.1% | Yes |
| 4 | 0.226 | 10.5% | Yes |
| 5 | 0.238 | 6.6% | Yes |
| 6 | 0.262 | 4.8% | No |
| 7 | 0.229 | 5.2% | Yes |

Although `k = 6` has the highest silhouette score, its smallest cluster contains only 30 HCPs and fails the 5% size rule. Among the eligible solutions, `k = 3` has the highest silhouette score and is selected.

### 6. Name and profile the segments

Numeric k-means labels are not treated as permanent business names. The notebook assigns names according to the feature that defines each segment:

- Highest `pa_failure_rate` → **Access-Blocked**
- Highest `early_discontinuation_rate` → **Underdosed & Dropping Off**
- Highest `kelvora_start_share` → **KELVORA Loyalists**

For the saved run:

| Numeric label | Segment | HCPs |
|---:|---|---:|
| 0 | Access-Blocked | 100 |
| 1 | Underdosed & Dropping Off | 234 |
| 2 | KELVORA Loyalists | 286 |

Numeric labels may change after changes to the data or model. Downstream reporting should use `segment_name`, not assume that a particular numeric label always represents the same segment.

### 7. Size the opportunity

The notebook calculates both mechanisms for every segment and sums them. It does not select only the segment's dominant mechanism.

#### Mechanism A: PA recovery

A PA failure is defined as either:

- `Abandoned - no response`
- `Denied - not appealed`

Recoverable patients and value are calculated as:

```text
PA recoverable patients = PA failures × assumed PA recovery rate
PA value = PA recoverable patients × 12 months × monthly contribution per patient
```

The PA recovery rate is read from `economics_reference`. It resolves to 0.45 in the supplied data; it is not hard-coded as `0.45` in the notebook.

#### Mechanism B: early-discontinuation recovery

For financial sizing, early discontinuation is narrower than the clustering feature. Only patients who discontinue before six months for either `Loss of response` or `Tolerability` are treated as addressable. Switches to SCIg, payer/access discontinuations, remission/taper, and ongoing patients are excluded from this value calculation.

```text
Retention-recoverable patients = addressable early discontinuations × 0.25
Retention value = retention-recoverable patients × 12 months × monthly contribution per patient
```

#### Assumptions

Three recovery/value assumptions are used:

| Assumption | Value | Source |
|---|---:|---|
| `assumed_pa_recovery_rate` | 0.45 | Supplied through `economics_reference` |
| `RETENTION_RECOVERY_RATE` | 0.25 | Analyst-defined notebook constant |
| `INCREMENTAL_MONTHS` | 12 | Analyst-defined notebook constant |

All three can be challenged and rerun. Changes to the PA recovery rate should be made in the source workbook; changes to the other two assumptions should be made in the opportunity-sizing cell.

### 8. Export the HCP master

The notebook exports `Kelvora_HCP_Seg_Master.xlsx`, containing one row for each of the 620 HCPs. The saved run contains 32 columns covering:

- HCP identifier and segment fields
- HCP/practice context
- The seven clustering features
- Patient/referral aggregates
- Monthly-activity aggregates

The file does not currently include the `*_was_imputed` indicators or the segment-level opportunity table.

## Analytical choices

### Volume and decile

`syndicated_patient_decile` and raw monthly activity counts are excluded as direct clustering features. This prevents the model from primarily recreating a large-practice-versus-small-practice segmentation.

One derived monthly-activity measure, `kelvora_start_share`, is included because it measures product preference rather than absolute practice size. After clustering, affected-patient counts are used to size the financial opportunity. Decile and monthly activity aggregates are added to the HCP master only for post-cluster context and reporting.

### Redundant candidate features

`conversion_rate` and `pa_required_rate` are not included in `FINAL_FEATURES`. The production notebook contains only the retained seven-feature model and does not rerun the candidate-feature diagnostics.

### Persistence window

The production model uses a six-month evaluable cohort. Ongoing patients with fewer than six months of observation are excluded rather than treated as failures, reducing bias caused by referral timing.

## Validation analyses not reproduced in the production notebook

The following findings document sensitivity analyses performed outside the final production notebook. They are retained as analytical rationale but cannot be regenerated by running this notebook alone:

- Adding volume and decile reorganized the segments around practice size: mean decile changed from 7.0 / 5.2 / 5.2 to 8.1 / 6.7 / 3.9, while the underdosing signal moved from approximately 0.78 to 0.90.
- `conversion_rate` correlated approximately -0.99 with `pa_failure_rate`; in the supplied data, non-conversion tracked failed prior authorization closely enough that retaining both would count the same access problem twice.
- `pa_required_rate` correlated near zero with the other candidate features, and PA requirements were approximately 91%–94% across the four payer channels.
- An earlier 12-month persistence definition correlated 0.42 with referral timing. After adopting the six-month evaluable-cohort rule, the documented correlation was -0.01.
- Applying a 60-HCP minimum would leave only `k = 3` and `k = 4` eligible; `k = 3` would still have the higher silhouette score.
- A proposed SCIg-switch mechanism was removed after documented testing found similar referral-to-infusion times for switchers and comparison groups (17.4 versus 17.6 and 17.1 days). The documented effect was a $1.9 million reduction from the preliminary opportunity estimate.

If these findings must be independently reproducible, their diagnostic code and outputs should be added to a separate validation appendix rather than mixed into the production workflow.

## Current workflow boundary

The notebook completes behavioral segmentation, segment profiling, two-mechanism opportunity sizing, and HCP-master export. It does not currently:

- Price intervention levers
- Allocate FRM resources
- Calculate program cost or ROI
- Produce a final funding recommendation
- Export the segment-level opportunity table

Those activities remain downstream of the notebook.
