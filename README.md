# Phishing URL Detection — Leak-Free Tabular Classification

A leak-free binary classification pipeline detecting phishing URLs from intra-URL lexical and structural features only (no page content, DOM, HTML, or CSS). Built for **FIT Competition 2026, Track III (Identity and Data Exploitation)**, preliminary round.

**Authors**: Riantama Putra, Athilla Zaidan Zidna Fann, Nicholas Wise Saragih Sumbayak (Institut Teknologi Bandung)

## Results (held-out test set)

| Model | F1 (phishing class) | ROC AUC | Recall | Accuracy |
|---|---|---|---|---|
| **LightGBM (final)** | **0.8991** | **0.9656** | 0.8763 | 0.9045 |
| Boosting ensemble (avg) | 0.8931 | 0.9627 | 0.8667 | 0.8992 |

LightGBM alone outperforms the averaging ensemble on every metric — the three boosters correlate at ~0.99, so there's no error diversity left for averaging to exploit; it only dilutes the strongest learner. At the F1-optimal operating threshold (0.445, tuned on OOF, not on test), recall rises to 0.892 and F1 to 0.9010, converting roughly 190 missed phishing URLs into caught ones.

## The Core Problem: Duplicate-Driven Data Leakage

This is the central methodological contribution of the analysis, not an afterthought. The dataset maps each URL to 41 coarse integer counts and flags (length, digit count, entropy, special characters, etc.). Because this encoding is coarse, **47.6% of rows (118,172 of 247,950) are exact duplicates** of some other row's feature vector. Under a naive random split, identical feature vectors leak across train and test, and the model partially memorizes labels instead of generalizing — inflating F1 by roughly 2 points versus the honest, deduplicated evaluation (quantified directly in the notebook, Section 8.2).

**Mitigation**: exact duplicates are dropped before any split. Stratified 5-fold CV then runs on the deduplicated data (class balance ~49/51 after dedup). The only vectors that can still appear on both sides of a split are the ~0.3% of rows with identical features but conflicting labels — an irreducible label-noise floor, not leakage, and left untouched rather than over-engineered away.

## Approach

1. **EDA** — quantify the duplicate structure (the central finding), audit dead/near-constant features, compare phishing vs. legitimate URL characteristics (phishing URLs are longer, higher-entropy, more likely to carry a query string), rank features by correlation and mutual information.
2. **Feature engineering** — evaluated but largely rejected: hand-built ratio features (e.g. digit density) added negligible OOF gain because LightGBM can already reconstruct such ratios from raw counts via splits. Decisions to *not* transform are stated explicitly rather than skipped silently.
3. **Preprocessing** — deduplicate before splitting (the one transformation the data forces).
4. **Cross-validation** — stratified 5-fold on deduplicated data, held-out 20% test set, single global seed, no early stopping on validation folds (avoids tuning leakage).
5. **Models** — Logistic Regression baseline vs. LightGBM, XGBoost, CatBoost. Tree ensembles chosen because the 41 features are heterogeneous counts/flags with strongly nonlinear, interaction-driven class boundaries (e.g. length × entropy × presence of a query string).
6. **Evaluation** — full metric set (Accuracy, Precision, Recall, F1, ROC AUC, PR AUC) on both OOF and held-out test, confusion matrices, ROC/PR curves, explicit leakage-inflation demonstration, and security-aware threshold tuning (false negatives — missed phishing — are costlier than false positives here).
7. **Interpretation** — tree-based importance consensus + SHAP on the final LightGBM model. URL length, entropy, digit counts, and special-character counts dominate, consistent with known phishing tradecraft (long, obfuscated, auto-generated URLs).

## Repository Structure

```
.
├── README.md
├── requirements.txt
├── data/
│   └── Dataset.csv                    # 247,950 URLs × 41 features (CC BY 4.0, see Dataset section)
├── notebooks/
│   └── Notebook_Preliminary_IndomaretLabtekV.ipynb   # Full pipeline: EDA → leakage analysis → modeling → SHAP
└── docs/
    └── Institut_Teknologi_Bandung_IndomaretLabtekV.docx   # Full written report
```

## Setup

```bash
pip install -r requirements.txt
jupyter notebook notebooks/Notebook_Preliminary_IndomaretLabtekV.ipynb
```

The notebook expects `Dataset.csv` in its working directory (edit `CFG.CANDIDATE_PATHS` in the config cell if running from `notebooks/` with data elsewhere).

## Dataset

**[Dataset of suspicious phishing URL detection](https://doi.org/10.3389/fcomp.2024.1308634)** — Tamal, M. A., Islam, M. K., Bhuiyan, T., & Sattar, A. (2024). *Frontiers in Computer Science*, 6:1308634. Licensed CC BY 4.0 (redistribution permitted with attribution), included directly in `data/` for reproducibility. 247,950 URLs, 41 integer/binary lexical and structural features, binary target (`Type`: 0 = legitimate, 1 = phishing).

## Key Takeaways

- Duplicate-driven leakage is the dominant methodological risk in coarsely-encoded tabular URL datasets — always check for it before trusting a leaderboard-style split.
- A correctly deduplicated, single strong model (LightGBM) beat a naive ensemble here, because the boosters had no meaningful error diversity to exploit.
- Threshold tuning against the cost asymmetry (missed phishing > false alarm) meaningfully improves the deployed operating point over a default 0.5 cutoff.
- Feature engineering that a tree model can already reconstruct from raw inputs (e.g. simple ratios) adds no value — worth checking via OOF gain before keeping engineered features.

## References

- Tamal, M. A., Islam, M. K., Bhuiyan, T., & Sattar, A. (2024). Dataset of suspicious phishing URL detection. *Frontiers in Computer Science*, 6:1308634. https://doi.org/10.3389/fcomp.2024.1308634
- Ke, G., et al. (2017). LightGBM: A Highly Efficient Gradient Boosting Decision Tree. *NeurIPS*.
- Chen, T., & Guestrin, C. (2016). XGBoost: A Scalable Tree Boosting System. *KDD*.
