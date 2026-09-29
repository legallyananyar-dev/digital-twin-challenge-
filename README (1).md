# WarfarinTwin: A GIFT-Anchored Digital Twin for Post-Arthroplasty Warfarin Dosing

> **Status:** In development. Built for the Happiest Health *Digital Twin Challenge 2026* (Reimagining and Reforming Healthcare in India Summit, Bengaluru).
> **Disclaimer:** Research and educational prototype only. It is **not** a medical device and must not be used to make real dosing decisions.

---

## 1. The problem in plain language

Warfarin is a widely used blood thinner, often given after hip or knee replacement surgery to prevent clots. It has a **narrow therapeutic index**: too much causes bleeding, too little allows clots. The dose a person needs varies enormously between individuals, driven partly by genetics (notably **CYP2C9**, **VKORC1**, and **CYP4F2**), age, body size, and interacting drugs.

Doctors track the effect using the **INR** (International Normalized Ratio), a blood clotting measure. The usual therapeutic goal after arthroplasty is around INR 2 to 3, and an **INR of 4 or higher** is a warning sign for bleeding.

## 2. Clinical anchor: the GIFT trial

This project is built around the **Genetic Informatics Trial of Warfarin to Prevent Deep Vein Thrombosis (GIFT)** (Gage et al., *JAMA* 2017;318(12):1115-1124):

- 1,650 patients aged 65+ undergoing elective hip or knee arthroplasty.
- Randomized to **genotype-guided** vs **clinically guided** warfarin dosing for the first 11 days, using the WarfarinDosing.org algorithm (genotype arm added VKORC1, CYP2C9, and CYP4F2 variants).
- Primary composite outcome (major bleeding, INR ≥4 within 30 days, VTE within 60 days, or death within 30 days): **10.8% genotype-guided vs 14.7% clinically guided**.
- The benefit was driven mainly by fewer episodes of INR ≥4.

WarfarinTwin asks: *can we build a virtual patient that reproduces this kind of trajectory and warns clinicians before an unsafe INR happens?*

## 3. What is a digital twin here?

A **digital twin** is a virtual replica of a patient that is continuously updated with new data and used to predict future states and test "what-if" scenarios. WarfarinTwin's twin holds:

| Layer | Contents |
|---|---|
| **Static profile** | Age, sex, height, weight, CYP2C9, VKORC1, CYP4F2 genotype, interacting drugs (e.g. amiodarone, CYP2C9 inducers) |
| **Dynamic state** | Daily warfarin dose, INR readings, optional wearable/vitals signals |
| **Outputs** | Predicted next-day INR, risk of INR ≥4, predicted time in therapeutic range, dose what-if comparison |

The twin **synchronizes** after each new INR: it updates its estimate of that patient's drug response and re-forecasts.

## 4. What the system predicts

1. **Next-day INR** (regression).
2. **Risk of INR ≥4 in the next 48 hours** (binary classification, the headline safety alert).
3. **Time in therapeutic range (TTR)** across days 0 to 11 (Rosendaal interpolation).
4. **What-if simulation:** predicted INR curve for candidate dose A vs dose B for the same virtual patient.
5. **Initial dose estimate** from the IWPC and Gage algorithms (baseline / starting point).

## 5. Architecture

```
            +---------------------------+
            |   Data layer              |
            |  - Synthetic cohort       |
            |  - IWPC (PharmGKB)        |
            |  - MIMIC-IV (validation)  |
            +-------------+-------------+
                          |
                          v
            +---------------------------+
            |   Patient profile builder |
            |  covariates + genotypes   |
            +-------------+-------------+
                          |
                          v
   +----------------------------------------------------+
   |                  TWIN ENGINE                       |
   |                                                    |
   |  [A] Mechanistic PK/PD (Hamberg-style)             |
   |        + Bayesian per-patient updating             |
   |  [B] Sequence model (GRU/LSTM)                     |
   |  [C] Gradient boosting (lag features)              |
   |                                                    |
   |  Hybrid stack: A's prediction -> feature for B, C  |
   |  Final blend + conformal prediction intervals      |
   +--------------------------+-------------------------+
                              |
                              v
            +---------------------------+
            |  Outputs & dashboard      |
            |  - INR forecast + band    |
            |  - INR >= 4 risk alert    |
            |  - Dose what-if           |
            +---------------------------+
```

### 5.1 Baselines
- **IWPC pharmacogenetic dosing algorithm** (Klein et al., *NEJM* 2009): age, height, weight, race, VKORC1, CYP2C9, and enzyme inducer/inhibitor use.
- **Gage algorithm** (WarfarinDosing.org), as used in GIFT.
- **Fixed 5 mg/day** reference dose.

### 5.2 Models
- **Mechanistic:** a population PK/PD model in the style of Hamberg et al., solved as differential equations, with Bayesian (MAP) updating after each observed INR.
- **Machine learning:** GRU/LSTM over the daily sequence (dose, INR, covariates), and XGBoost/LightGBM on lag features.

### 5.3 Ensemble
- **Hybrid residual stack:** the mechanistic prediction is fed to the ML models as an input feature so they learn what the PK/PD model misses.
- **Weighted blend** of mechanistic + GRU + boosting, with weights fit on a validation split.
- **Conformal prediction intervals** for calibrated uncertainty bands on the INR curve.

## 6. Data

| Source | Use | Access |
|---|---|---|
| **Synthetic cohort (this repo)** | Primary training and simulation data | Generated locally by `data/synthetic/` |
| **IWPC dataset (PharmGKB)** | Starting-dose model and baseline comparison (cross-sectional, no daily INR) | Public download at `pharmgkb.org/downloads` |
| **MIMIC-IV (PhysioNet)** | Real-world validation of INR trajectories (no genotypes) | Credentialed access: PhysioNet account, CITI "Data or Specimens Only Research" training, signed data use agreement |

### Synthetic cohort generation
Because no public dataset combines genotype, daily post-arthroplasty INR, and dosing, we simulate virtual patients:

1. Sample covariates (age ≥65, sex, weight, height, interacting drugs) from GIFT-like distributions.
2. Sample CYP2C9, VKORC1, and CYP4F2 genotypes from published allele frequencies (with a population switch for Indian-relevant frequencies).
3. Simulate 11-day INR trajectories with a Hamberg-style PK/PD model, adding inter-individual variability and measurement noise.
4. Apply two dosing arms (genotype-guided vs clinically guided) to reproduce a GIFT-style comparison.

**Important:** synthetic data reflects the assumptions of the simulator. Results on it show that the pipeline works, not that it will match real patients. Real-world validation uses MIMIC-IV where possible.

> **Data licensing:** MIMIC-IV and other credentialed data must **never** be committed to this repository. The data use agreement prohibits redistribution. Only code and synthetic data are tracked in Git.

## 7. Evaluation plan

- **Splitting:** by patient (grouped cross-validation), never by visit, to prevent leakage.
- **INR forecast:** MAE, RMSE, and percentage of predictions within ±0.5 INR.
- **INR ≥4 alert:** AUROC, AUPRC, recall at fixed precision, calibration plot.
- **Baseline comparison:** IWPC, Gage, fixed-dose, mechanistic-only, ML-only, and full ensemble.
- **Trial-level sanity check:** in simulation, does the genotype-guided arm show a GIFT-like relative reduction in adverse events?
- **External check:** INR trajectory performance on MIMIC-IV warfarin patients.
- **Subgroup analysis:** performance by genotype group and by ancestry-linked allele frequency setting.
- **Uncertainty:** empirical coverage of conformal intervals.

### Results
*To be filled in as experiments complete.*

| Model | INR MAE | % within ±0.5 | INR ≥4 AUROC |
|---|---|---|---|
| Fixed dose | TBD | TBD | TBD |
| IWPC / Gage | TBD | TBD | TBD |
| PK/PD + Bayesian | TBD | TBD | TBD |
| GRU | TBD | TBD | TBD |
| Boosting | TBD | TBD | TBD |
| **Hybrid ensemble** | TBD | TBD | TBD |

## 8. Repository structure

```
warfarin-twin/
├── README.md
├── data/
│   ├── synthetic/        # simulator and generated cohorts
│   ├── iwpc/             # place downloaded IWPC file here (not committed)
│   └── mimic/            # local only (not committed, DUA)
├── src/
│   ├── simulator/        # PK/PD virtual patient generator
│   ├── baselines/        # IWPC, Gage, fixed dose
│   ├── models/           # PK/PD+Bayes, GRU, boosting, ensemble
│   ├── evaluation/       # metrics, TTR, conformal intervals
│   └── app/              # dashboard (Streamlit)
├── notebooks/
├── tests/
├── requirements.txt
└── .gitignore            # excludes data/iwpc, data/mimic
```

## 9. Getting started

```bash
git clone https://github.com/<your-username>/warfarin-twin.git
cd warfarin-twin
pip install -r requirements.txt

# 1. Generate synthetic cohort
python -m src.simulator.generate --n 5000 --out data/synthetic/cohort.csv

# 2. Train and evaluate
python -m src.models.train --config configs/ensemble.yaml
python -m src.evaluation.run --config configs/eval.yaml

# 3. Launch the dashboard
streamlit run src/app/app.py
```
*(Commands are the intended interface and will be finalized as the code lands.)*

## 10. Limitations

- Synthetic training data depends on simulator assumptions.
- IWPC is cross-sectional and does not contain daily INR trajectories.
- MIMIC-IV is a single US center and lacks genotype data.
- Early algorithms performed better in Europeans than in Asian and African populations; population-specific variants (e.g. CYP4F2) and diverse data are needed for fairness.
- Not clinically validated. Not for patient care.

## 11. Related work

- GIFT trial: Gage et al., *JAMA* 2017.
- IWPC algorithm: Klein et al., *NEJM* 2009.
- CPIC guideline for pharmacogenetics-guided warfarin dosing (2016/2017 update): Johnson et al.
- LSTM INR modeling (*Frontiers in Cardiovascular Medicine*, 2022) and the AI-WAR application.
- Hamberg PK/PD model and Bayesian decision support tool (*BMC Medical Informatics and Decision Making*, 2014).
- CURATE.AI applied to warfarin dosing.

WarfarinTwin builds on this work and adds: (1) anchoring to the GIFT post-arthroplasty setting, (2) a twin framing with live updating and what-if simulation, (3) inclusion of CYP4F2, and (4) a population-diversity analysis.

## 12. Team

- **[Your name]**: project lead, modeling
- **[Friend's name]**: data acquisition
- **[Other members]**

## 13. License

Code released under the MIT License (add a `LICENSE` file). Synthetic data is released under CC BY 4.0. Third-party datasets remain under their original licenses.

## 14. References

1. Gage BF, et al. Effect of genotype-guided warfarin dosing on clinical events and anticoagulation control among patients undergoing hip or knee arthroplasty: the GIFT randomized clinical trial. *JAMA*. 2017;318(12):1115-1124.
2. International Warfarin Pharmacogenetics Consortium; Klein TE, et al. Estimation of the warfarin dose with clinical and pharmacogenetic data. *N Engl J Med*. 2009;360:753-764.
3. Johnson JA, et al. CPIC guideline for pharmacogenetics-guided warfarin dosing: 2017 update. *Clin Pharmacol Ther*. 2017;102(3):397-404.
4. Hamberg AK, et al. A PK-PD model for predicting the impact of age, CYP2C9, and VKORC1 genotype on individualization of warfarin therapy. *Clin Pharmacol Ther*. 2007.
5. Johnson AEW, et al. MIMIC-IV. *Sci Data*. 2023.
