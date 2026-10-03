# WarfarinTwin: A GIFT-Anchored Digital Twin for Post-Arthroplasty Warfarin Dosing

> **Status:** Working prototype built for the Happiest Health *Digital Twin Challenge 2026*.
> **Disclaimer:** Research and educational prototype only. It is **not** a medical device and must not be used for real dosing decisions.
> **Evidence level:** Part 1 uses real patient data (IWPC). Part 2 is a **simulation study**: it shows the method works under stated assumptions, not that it works on real patients yet.

---

## 1. The problem

Warfarin is a blood thinner often given after hip or knee replacement to prevent clots. It has a **narrow therapeutic index**: too much causes bleeding, too little allows clots. The dose a person needs varies widely, driven partly by genetics (**CYP2C9**, **VKORC1**, **CYP4F2**), age, body size, and interacting drugs. Doctors track the effect with the **INR** blood test (goal about 2 to 3; **INR >= 4** is a bleeding warning sign).

**Clinical anchor: the GIFT trial** (Gage et al., *JAMA* 2017;318(12):1115-1124). In 1,650 patients aged 65+ undergoing elective hip or knee arthroplasty, genotype-guided warfarin dosing gave fewer adverse events (10.8% vs 14.7%), driven mainly by fewer INR >= 4 episodes.

WarfarinTwin asks: *can a virtual patient, updated by each day's INR, forecast what happens next, warn before INR >= 4, and tell a clinician what a different dose would do?*

## 2. What we built

| Part | What it does | Data | Notebook |
|---|---|---|---|
| **1. Starting-dose model** | Predicts the stable weekly dose from age, weight, height, sex, ancestry, CYP2C9, VKORC1, interacting drugs | Real: IWPC (5,410 patients) | `notebooks/01_starting_dose_IWPC.ipynb` |
| **2. Digital twin** | Simulates post-surgery patients; forecasts INR 1 and 2 days ahead; alerts for INR >= 4; compares candidate doses | Simulated (calibrated to Part 1) | `notebooks/02_digital_twin_simulation.ipynb` |

### Architecture

```
 IWPC (real patients) --> Part 1: starting-dose models --> predicted stable dose
                                                              |
                                                              v
        Virtual patient simulator (CYP2C9 -> elimination speed, response delay, hidden dose
        requirement, day-to-day noise, random drug interactions)
                                                              |
                          dosing strategies --> daily INR + dose histories
                                                              |
              +-----------------------+-----------------------+
              v                       v                       v
   [A] Mechanistic Bayesian   [B] Gradient-boosting    [C] Hybrid = twin forecast
       twin (updates a            forecaster                + ML correction
       distribution over this                               + conformal 90% bands
       patient's sensitivity)
              |                       |                       |
              +-----------------------+-----------------------+
                                      v
       INR forecast (1 and 2 days) | P(INR >= 4) alert | dose "what-if" comparison
```

## 3. Results

### Part 1: starting dose (real IWPC data, 5,410 patients)

Hold-out test (random 80/20, stratified by ancestry; imputation fit on training data only):

| Model | MAE (mg/week) | R2 | Within 20% of true dose |
|---|---|---|---|
| Fixed 5 mg/day | 12.82 | -0.065 | 29.0% |
| IWPC clinical equation | 9.69 | 0.282 | 40.6% |
| IWPC pharmacogenetic equation | 8.73 | 0.413 | 43.7% |
| Ridge (sqrt dose) | 8.57 | 0.420 | 44.4% |
| Random Forest | 8.85 | 0.376 | 44.3% |
| Gradient Boosting | 8.58 | 0.404 | 45.4% |
| **Stacked Ensemble** | **8.49** | **0.423** | **45.2%** |

On **unseen hospital sites** (5-fold grouped by site), the IWPC pharmacogenetic equation (MAE 8.88, 43.7% within 20%) is as good as the ensemble (MAE 9.04, 42.7%).

**Honest reading:** machine learning improves only marginally over the published IWPC equation. Genotype information matters far more than model choice (VKORC1 is the most important feature, then age, CYP2C9, weight). Dosing is hardest for patients with unknown VKORC1 and for high-dose patients (>49 mg/week).

*Note:* `INR on therapeutic dose` was deliberately **excluded** as a feature because it is measured after the stable dose is reached (data leakage).

### Part 2a: in-silico mini-trial (1,500 virtual patients, same patients in every arm)

| Starting strategy | INR >= 4 (any, days 1-11) | INR >= 5 | Time in range 2-3 (days 4-11) |
|---|---|---|---|
| Flat 5 mg/day (reference) | 51.1% | 23.1% | 32.6% |
| Clinical-factors start (no genotype) | 25.3% | 6.8% | 38.6% |
| **Genotype-guided start** | **16.3%** | **2.3%** | **43.6%** |

Genotype-guided vs clinical-factors start: **35.8% relative reduction** in INR >= 4 (absolute difference 9.1 points, bootstrap 95% CI 7.1 to 11.1). The result depends on how much of the unexplained dose variability is real biology (`VAR_SCALE`):

| VAR_SCALE | INR >= 4 clinical | INR >= 4 genotype | Relative reduction |
|---|---|---|---|
| 0.5 | 24.4% | 14.0% | 42.6% |
| 0.7 (default) | 27.7% | 17.7% | 36.1% |
| 1.0 | 32.2% | 23.1% | 28.3% |

We did **not** tune the simulator to reproduce GIFT's published numbers. Absolute event rates here are higher than GIFT's; compare only the direction and rough size of the effect.

### Part 2b: forecasting INR (4,000 virtual patients, split by patient 60/20/20)

| Horizon | Model | MAE (INR) | Within +-0.5 INR |
|---|---|---|---|
| 1 day | Persistence (INR today) | 0.373 | 73.8% |
| 1 day | Mechanistic twin | 0.252 | 86.7% |
| 1 day | ML only | 0.216 | 90.2% |
| 1 day | **Hybrid** | **0.215** | **90.1%** |
| 2 days | Persistence | 0.706 | 47.7% |
| 2 days | Mechanistic twin | 0.355 | 78.1% |
| 2 days | ML only | 0.310 | 82.2% |
| 2 days | **Hybrid** | **0.305** | **82.1%** |

Conformal 90% bands reached **90.3%** (1 day) and **89.7%** (2 days) coverage on held-out patients.

### Part 2c: INR >= 4 alert (about 5% of rows are events)

| Horizon | Model | AUROC | AUPRC |
|---|---|---|---|
| 1 day | Persistence | 0.950 | 0.575 |
| 1 day | Mechanistic twin | 0.958 | 0.675 |
| 1 day | ML only | 0.980 | 0.785 |
| 1 day | **Hybrid** | **0.982** | **0.790** |
| 2 days | Persistence | 0.819 | 0.219 |
| 2 days | Mechanistic twin | 0.911 | 0.525 |
| 2 days | ML only | 0.966 | 0.663 |
| 2 days | **Hybrid** | **0.967** | **0.672** |

### Part 2d: the key test, "what if I change the dose?"

The simulator gives ground truth, so we re-ran 600 test patients with the day-4 and day-5 doses multiplied by 0, 0.5, 0.75, 1.25 and 1.5, and compared predicted vs true change in INR two mornings later.

| Model | Correlation (predicted vs true change) | Direction correct | Mean error (INR) |
|---|---|---|---|
| **Mechanistic twin** | **0.973** | **100%** | **0.058** |
| ML only | -0.440 | 52.1% | 0.378 |
| Hybrid | -0.131 | 76.1% | 0.381 |

**Why this matters:** in training data, doses were chosen *in response to INR* (a high INR leads to a held dose). Pure ML learns "dose held goes with high INR" and gets cause and effect wrong (confounding by indication). The mechanistic twin models how the drug actually works, so it does not. For routine forecasting all models are similar; for dose decisions, only the twin is reliable. The hybrid is best for point forecasts, the twin is the engine for comparing doses.

Plots are in `outputs/`: `holdout_plots.png`, `feature_importance.png`, `simulator_sanity.png`, `forecast_whatif_plots.png`, `example_patients.png`.

## 4. Assumptions and limitations (please read)

| Assumption | Value | Where in code |
|---|---|---|
| Relative elimination speed by CYP2C9 genotype | 1.0 / 0.85 / 0.65 / 0.70 / 0.50 / 0.30 (*1/*1 ... *3/*3) | `CL_MULT` (illustrative; not taken from a paper) |
| Share of IWPC dose-residual spread treated as real biology | 0.7 (sensitivity tested above) | `VAR_SCALE` |
| INR response curve and delay | Imax 12, gamma 2.5, 3 transit compartments, ~2.5 day mean | constants |
| Titration protocol | simple inpatient-style rules | `TitrationPolicy` |
| Virtual patient age | mean 72, SD 6, range 65-90 | `sample_cohort` (align with GIFT Table 1 later) |
| Random drug-interaction event | 8% of patients | `sample_cohort` |
| Target INR | 2.5 | `TARGET_INR` |

- **Part 2 is a simulation study.** The twin shares the simulator's structure, so its accuracy and its edge in the what-if test are **optimistic**. Validation on real longitudinal INR data (e.g. MIMIC-IV, which requires credentialed access) is the main future work.
- The simulator's structure is *inspired by* published warfarin K-PD models (e.g. Hamberg et al.), but its parameters are **not copied** from those papers.
- The IWPC equation coefficients in Part 1 were entered from the published paper and sanity-checked (the genotype equation beats the clinical one, MAE about 8.7); please verify against Klein et al. 2009 before relying on them.
- IWPC data are from 2008 and under-represent some populations. Unknown genotypes are modelled as their own category.
- Absolute event rates in the simulation are not calibrated to real-world rates.

## 5. Repository layout

```
warfarin-twin/
├── README.md
├── requirements.txt
├── LICENSE
├── notebooks/
│   ├── 01_starting_dose_IWPC.ipynb
│   └── 02_digital_twin_simulation.ipynb
├── models/                      # trained outputs of the notebooks (re-creatable by running them)
│   ├── starting_dose_model.joblib
│   └── inr_forecaster_bundle.joblib
├── outputs/                     # result tables (CSV) and figures (PNG)
└── data/
    └── iwpc/                    # see Data section
```

**What the files are:**
- `notebooks/` is the **code**: everything needed to reproduce every result.
- `models/*.joblib` are the **saved trained models** produced by the notebooks, so others can use them without retraining. They are optional: running the notebooks recreates them. If a model fails to load (different scikit-learn version), just rerun the notebooks.
- `outputs/` holds the result tables and figures quoted in this README.

## 6. How to run (Google Colab)

1. Open `notebooks/01_starting_dose_IWPC.ipynb` in Colab (File > Open notebook > GitHub, or upload it). Run all cells and upload the IWPC `.xls` when asked. This creates `models/starting_dose_model.joblib`.
2. Open `notebooks/02_digital_twin_simulation.ipynb`. Upload the IWPC `.xls` and `starting_dose_model.joblib` when asked (if the model is missing, a quick fallback is trained). Run all cells.
3. Total runtime is a few minutes. Python 3 with numpy, pandas, scipy, scikit-learn, matplotlib, joblib, xlrd (see `requirements.txt`).

Using the twin on one patient (end of Notebook 2): `forecast_patient(profile, inr_history, dose_history, candidate_doses_tonight)`.

## 7. Data

- **IWPC dataset** ("Warfarin Consortium Combined Data Set", March 2008), from PharmGKB / ClinPGx: https://www.pharmgkb.org/downloads . 5,700 patients; 5,410 usable (reached stable dose with a recorded dose). The file is distributed by PharmGKB under its own license (CC BY-NC-SA 4.0 according to the bio.tools listing); please download it from the source and respect its terms. Credit: International Warfarin Pharmacogenetics Consortium; PharmGKB.
- **Simulated cohorts** are generated by the notebooks; no patient data other than IWPC is used.

## 8. Related work

- GIFT trial: Gage et al., *JAMA* 2017.
- IWPC algorithm: Klein et al., *NEJM* 2009.
- CPIC guideline for pharmacogenetics-guided warfarin dosing (Johnson et al., 2017 update).
- LSTM INR modeling (*Frontiers in Cardiovascular Medicine*, 2022) and warfarin PK/PD Bayesian decision support (Hamberg et al.).
- Warfarin dose-prediction pipelines trained on IWPC (static dose prediction).

**What is different here:** a GIFT-anchored post-arthroplasty setting, a twin with live Bayesian updating, dose what-if comparison validated against simulator ground truth, uncertainty bands, and a demonstration of why mechanistic structure matters for causal questions.

## 9. Team and institution

| Name | Role |
|---|---|
| **B. Ananya Raghu (Anu)** | Project lead and modeling. Third-year B.Pharm student. |
| **[Teammate name]** | Data acquisition. [Year and course] |

**Institution:** Saraswathi Vidya Bhavan's College of Pharmacy, Dombivli (East), Maharashtra (University of Mumbai).
**Challenge:** Happiest Health Digital Twin Challenge 2026.
**Contact:** [email address]

## 10. License

Code: MIT License (add a `LICENSE` file). Third-party data remain under their original licenses.

## 11. References

1. Gage BF, et al. Effect of genotype-guided warfarin dosing on clinical events and anticoagulation control among patients undergoing hip or knee arthroplasty: the GIFT randomized clinical trial. *JAMA*. 2017;318(12):1115-1124.
2. International Warfarin Pharmacogenetics Consortium; Klein TE, et al. Estimation of the warfarin dose with clinical and pharmacogenetic data. *N Engl J Med*. 2009;360:753-764.
3. Johnson JA, et al. CPIC guideline for pharmacogenetics-guided warfarin dosing: 2017 update. *Clin Pharmacol Ther*. 2017;102(3):397-404.
4. Hamberg AK, et al. A PK-PD model for predicting the impact of age, CYP2C9, and VKORC1 genotype on individualization of warfarin therapy. *Clin Pharmacol Ther*. 2007.
