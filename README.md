# CareLoop Clinics — No-Show Guard

> End-to-end analysis of outpatient appointment no-shows: cleaning, SQL analysis, machine learning risk scoring, and GenAI-assisted follow-up review.

**GitHub repository:** `PASTE_PUBLIC_GITHUB_REPOSITORY_URL_HERE`

## Business question

Which appointments are most likely to be missed, why do patients miss them, and what should CareLoop do first?

## Project structure

- `01_ingest_clean.ipynb` — Part 1, Tasks 1.1–1.6
- `rejected_rows.json` — rows rejected by the csv-module type conversion
- `appointments_clean.csv` — cleaned hand-off table used by Parts 2 and 3
- `clinics_enriched.csv` — clinic master plus Open-Meteo temperature
- `load_data.sql` — MySQL load statements
- `02_queries.sql` — Part 2 SQL
- `03_model.ipynb` — Part 3 model comparison, selection and thresholding
- `test_risk_scores.csv` — test-set risk probabilities and flags
- `04_genai.ipynb` — Part 4 Gemini + Pydantic workflow
- `notes_analysed.csv` — first 10 follow-up notes after GenAI processing
- `data/` — original input files; do not edit them

## Part 1 findings

- The raw export contains 1,020 rows; the csv-module type check accepts 988 and rejects 32.
- After duplicate removal and required date/status cleaning, 990 appointment rows remain.
- The cleaned table has 218 no-shows, a 22.0% overall no-show rate.
- 296 appointments (29.9%) have lead times above 14 days.

## Part 2 findings

- Among clinics with at least 95 appointments, Hadapsar Care Centre has the highest no-show rate at 27.3%, followed by Malviya Nagar Clinic at 26.8%.
- Without the 95-appointment filter, Gomti Nagar Care Point has the highest observed rate at 33.3%, but it has only 90 appointments.
- No-show risk increases sharply with lead time: the `31+` band has a 60.5% no-show rate versus 16.2% for `1-7` days.
- The monthly no-show rate ranges from 20.4% to 24.8% across January–June 2025.

## Part 3 findings

- The feature matrix has 11 columns; the split is 792 training rows and 198 test rows.
- The always-Attended baseline accuracy is 0.7525.
- Logistic regression achieved approximately 0.789 test ROC-AUC.
- In the fitted logistic model, `lead_days` and `prior_no_shows` are positive associations with no-show risk, while `reminder_sent` is strongly negative.
- Training-only cross-validation selected logistic regression as the final model. The chosen threshold is 0.35 because its out-of-fold F1 is higher than at 0.50 and recall is substantially higher.
- On the held-out test set at threshold 0.35, the model achieves approximately 0.57 precision, 0.57 recall and 0.79 ROC-AUC, with 49 appointments flagged in the reference run.

## Part 4 findings

The GenAI notebook must be run with the user's own free Gemini API key entered only at runtime. This repository version intentionally does not contain an API key or fabricated Gemini responses.

After running `04_genai.ipynb`, update this section with the actual `notes_analysed.csv` results and the reminder cross-tab.

## Recommendation

CareLoop should prioritize evening-before follow-up for appointments with long lead times and previous no-shows, while continuing reminders and using the risk score to allocate limited call capacity. The data show an overall no-show rate of 22.0% across 990 cleaned appointments; the `31+` lead-time group has a 60.5% no-show rate; the highest rate among sufficiently sized clinics is 27.3% at Hadapsar; and the selected logistic model provides about 0.79 ROC-AUC at a 0.35 operating threshold. The call-centre notes should then be used to distinguish reminder failures from transport, cost, illness and work/schedule issues, so interventions can be matched to the reason rather than treating every high-risk booking the same.

## Model choice

**Final model:** logistic regression pipeline with standard scaling.

**Selection rule:** 5-fold cross-validated ROC-AUC on the training data only.

**Threshold:** 0.35, selected using out-of-fold training probabilities because it produced higher F1 and materially higher recall than 0.50.

## Limitations

1. The first 10 follow-up notes are an illustration and cannot establish the true reasons clinic-wide.
2. The cleaning medians, quartiles and outlier fences were calculated on all 990 rows before the train/test split; a production pipeline should calculate these using training data only.
3. The same patient can appear in both train and test, and the random split is not a past-to-future evaluation.

## Reproducibility

The project uses:

- `random_state=42`
- `test_size=0.2`
- percentages rounded to one decimal where specified
- NumPy population standard deviation where specified
- pandas/NumPy default quartile method

## Security checklist

Before making the repository public:

- Enter the Gemini API key only with `input()` at runtime.
- Clear notebook output containing the key.
- Search the entire notebook for the first characters of the key.
- Do not commit the key, environment files, or other secrets.
