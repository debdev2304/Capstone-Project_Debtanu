# CareLoop Clinics — No-Show Guard

**Repository:** https://github.com/debdev2304/Capstone-Project_Debtanu

**Business question:** Which appointments are most likely to be missed, why do patients miss them, and what should CareLoop do first?

## Summary (one finding per Part)
1. **Data:** the raw export of 1,020 rows cleans to **990 appointments**: 20 duplicates, 6 unreadable dates and 4 blank statuses were removed, 5 impossible ages were fixed and 29 outliers capped. **218 are no-shows (22.0%)**.
2. **SQL:** no-shows cluster in long-lead bookings. Bookings made **31+ days ahead are missed 60.5% of the time**, against **16.2%** for bookings made 1–7 days ahead. Gomti Nagar (Lucknow) has the highest clinic rate at **33.3%**, from only 90 appointments.
3. **Model:** a logistic regression scores risk with a **test ROC-AUC of 0.789**. At a 0.35 threshold it flags 49 of 198 test bookings and catches **57.1% of the real no-shows** (28 of 49). In this model, a reminder goes with **about 69% lower odds** of a no-show (odds × 0.31).
4. **GenAI:** the two notes that report forgetting or getting no reminder (N001, N007) both belong to bookings where **no reminder was sent (2 of 2)**. Overall, **5 of the 10** notes belong to bookings without a reminder.

## Recommendation
CareLoop should first **make sure every booking gets a reminder**, and then, each evening, **phone the bookings the model flags**, starting with long-lead ones. Today 22.0% of the 990 cleaned appointments are missed. The strongest patterns point the same way: bookings made more than a month ahead are missed 60.5% of the time (16.2% for bookings made within a week), and in the model a reminder goes with roughly 69% lower odds of a no-show, holding the other features constant. The call-centre notes agree: both "forgot / no reminder" notes (N001, N007) came from bookings where no reminder was sent. An evening call list at the 0.35 threshold covers about a quarter of bookings (49 of 198 in the test set) and catches 57.1% of the real no-shows, which is far more useful than finding out the next day. The next step is to check Gomti Nagar's 33.3% rate: it is the highest, but it rests on only 90 appointments, so watch it for another quarter before acting on that clinic alone. Note that the reminder effect is an association from observational data, so the cleanest proof would be a small A/B test that sends reminders to a random half of the bookings that currently get none.

## Model choice
**Logistic regression** (StandardScaler + LogisticRegression in a pipeline), chosen **only from training data**. Its 5-fold cross-validated ROC-AUC on the training set was **0.774**, the highest of the four candidates (tuned random forest 0.767, random forest 0.747, depth-3 tree 0.678). It is also easy to explain: each coefficient shows how a feature moves the risk. The **threshold of 0.35** was chosen from out-of-fold training probabilities because it had the higher F1 (0.489 vs 0.347 at 0.5) and nearly twice the recall (0.467 vs 0.249). A missed warning costs an empty slot, while an extra call costs a minute. One evaluation on the test set: **ROC-AUC 0.789**, precision 0.571, recall 0.571, 49 flagged. Scores for every test booking are in `test_risk_scores.csv`.

## Limitations
1. **The 10 notes are only an illustration.** They are too few to show the real reasons clinic-wide, and all come from the first rows of one file.
2. **The data are observational, so associations are not causes.** Patients who get reminders may differ in ways the data do not record.
3. **The split is random, not past-to-future, and the same patient can appear in both train and test.** The cleaning medians and outlier fences were also computed on all 990 rows before the split. All three make the test score a little optimistic compared with a live system trained on the past and used on the future.

## Files
| File | Part | What it is |
|---|---|---|
| `data/` | — | The three original files, unedited |
| `01_ingest_clean.ipynb` | 1 | Tasks 1.1–1.6 |
| `rejected_rows.json` | 1 | 32 rows that failed the type check in Task 1.1 |
| `appointments_clean.csv` | 1 | 990 × 16 cleaned table (input to Parts 2–4) |
| `clinics_enriched.csv` | 1 | Clinics + `temp_c` from Open-Meteo |
| `load_data.sql` | 1→2 | 998 INSERT statements (8 clinics first, then 990 appointments) |
| `02_queries.sql` | 2 | Schema, load checks, duplicate-key error, Tasks 2.2–2.5 |
| `03_model.ipynb` | 3 | Tasks 3.1–3.5, plus the pandas check of Task 2.2 |
| `test_risk_scores.csv` | 3 | 198 test bookings: `appointment_id, no_show, risk_prob, flag` |
| `04_genai.ipynb` | 4 | Tasks 4.1–4.4(a); the key is entered at run time and is not in the file |
| `notes_analysed.csv` | 4 | First 10 notes, structured and validated |

Run order: `01` → `02_queries.sql` (in MySQL, with `load_data.sql`) → `03` → `04`. Environment for the saved outputs: scikit-learn 1.9, pandas 3.0, NumPy 2.5.
