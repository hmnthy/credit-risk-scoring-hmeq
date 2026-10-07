[Français](README.md) · **English** · [Tiếng Việt](README.vi.md)

# Credit Risk Scoring – HMEQ

Team project from the Master 2 MOSEF programme (Statistical Modelling for Economics and Finance), academic year 2024-2025.

The goal is to predict default on 5,960 home equity loans, then turn the results into a risk segmentation grid that can guide lending policy.

## Key results

All models are evaluated on the same test set (1,192 loans, 265 defaults) at a 0.5 decision threshold:

| Model | Recall | Precision | F1 | ROC-AUC |
|---|---|---|---|---|
| Logistic regression (Lasso, discretised features, class weights) | 0.69 | 0.43 | 0.53 | 0.79 |
| **XGBoost + class weights** | **0.80** | **0.89** | **0.84** | **0.96** |
| LightGBM + class weights | 0.81 | 0.85 | 0.83 | 0.96 |

- On the test set, XGBoost catches 212 of the 265 defaults and raises 27 false alarms among 927 good borrowers.
- Lowering the threshold to the Youden point on the ROC curve (≈ 0.27) raises recall to 0.89, with precision at 0.77. Where to set the threshold remains a business decision.

## Problem

Recall is the priority metric, because a missed default costs far more than a rejected good customer. The data is imbalanced (about 20 % defaults) and has missing values, especially in the debt-to-income ratio `DEBTINC` (21 % missing).

## Data

The data is the public **HMEQ** (Home Equity) dataset: 5,960 loans, 12 explanatory variables and a binary target `BAD` (1 = default, 1,189 cases).

| Variable | Description |
|---|---|
| `LOAN` | Requested loan amount |
| `MORTDUE` | Amount due on the existing mortgage |
| `VALUE` | Property value |
| `REASON` | Loan purpose: `DebtCon` (debt consolidation) or `HomeImp` (home improvement) |
| `JOB` | Occupational category (6 levels) |
| `YOJ` | Years at current job |
| `DEROG` | Number of major derogatory reports |
| `DELINQ` | Number of delinquent credit lines |
| `CLAGE` | Age of the oldest credit line (months) |
| `NINQ` | Number of recent credit inquiries |
| `CLNO` | Number of credit lines |
| `DEBTINC` | Debt-to-income ratio |

## Approach

1. **Exploratory analysis.** We looked at univariate and bivariate distributions, default rates by category and by bin, and class overlap (pairplots). A full profiling report is included (`hmeq_report.html`).
2. **Model shortlisting.** LazyPredict (around thirty classifiers) was run under 4 preprocessing set-ups: default, outlier removal (IQR), optimised preprocessing (iterative imputation, scaling, one-hot encoding), and that last set-up combined with SMOTE. Models were compared on recall and ROC-AUC, with run time as a third criterion. `MORTDUE` was dropped because it is highly correlated with `VALUE`.
3. **Logistic regression (baseline).** We checked the model assumptions (logit linearity, VIF, Cramér's V) and tried three feature sets (raw, discretised, combined). We used Ridge, Lasso and Elastic Net regularisation, and handled imbalance with class weights or SMOTE.
4. **Gradient boosting (XGBoost, LightGBM).** GridSearchCV (5-fold stratified cross-validation, scored on recall) tuned the hyperparameters and the preprocessing choices together. We compared no rebalancing, SMOTE and class weights (`scale_pos_weight`). Class weights were kept because they gave the best recall without generating synthetic data.
5. **Score grid.** The grid gives observed default rates on the training set by occupation × purpose × incidents (`JOB` × `REASON` × `DEROG` × `DELINQ`, 203 segments), along with the size of each segment.

## What the grid shows

- The lowest-risk profiles are office workers (`Office`) with no incident: 7.8 % default for debt consolidation and 1.2 % for home improvement.
- Riskier profiles are sales staff (`Sales`, 23 % for debt consolidation) and the self-employed (`Self`, 18 %).
- A first major derogatory report (`DEROG` = 1) clearly raises the risk. With `DEROG` ≥ 2 or `DELINQ` > 2, observed rates often reach 100 %, but these segments each hold less than 0.1 % of borrowers.

## Limitations

- The grid uses **observed default rates**, not model scores. Very small segments, with rates of 0 % or 100 %, are not robust. They should be merged or smoothed before any operational use.
- The evaluation relies on a single 80/20 train/test split (not stratified), with no out-of-time validation, since HMEQ has no time dimension.
- Results are reported at a 0.5 threshold. No threshold based on a real cost matrix was defined.
- Logistic regression and the boosting models use different preprocessing, so the comparison is between the best configuration of each family.
- The notebook was built for Google Colab (it loads data from Google Drive), and the environment versions are not pinned.

## Team

The team was Sara LAVAL-JEANTET, Chahla TARMOUN, Minh Nhat Thy HUYNH and Salma BENMOUSSA, supervised by Mr Ibrahim TOURE.

Minh Nhat Thy HUYNH's contribution covered model shortlisting (LazyPredict and the preprocessing comparison), XGBoost and LightGBM modelling, and building the score grids.

## Repository contents

| File | Contents |
|---|---|
| `Scoring.ipynb` | Full notebook: exploration, modelling, evaluation, score grids |
| `Rapport_Scoring.docx` | Project report (in French) |
| `hmeq_report.html` | Data profiling report (ydata-profiling). Download it and open it in a browser |
| `grille_de_score_weight.xlsx` | Exported score grid (203 segments) |
| `requirements.txt` | Libraries and versions of the original environment |

## Running the notebook

1. Download the public HMEQ dataset (`hmeq.csv`), for example from [Kaggle](https://www.kaggle.com/datasets/ajay1735/hmeq-data).
2. Open `Scoring.ipynb` in Google Colab.
3. In the "Chargement des données" (data loading) cell, change the file path, or use the manual upload option in that same cell.

Original environment: Google Colab, Python 3.10. Library versions are listed in `requirements.txt`.
