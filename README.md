# Battery SOH Estimation – Short Research Task

## 1. Task Overview

This repository contains a preliminary study on **State of Health (SOH) estimation** of lithium-ion batteries using a publicly available dataset.

The goal is **not** to achieve the highest possible accuracy, but to demonstrate:

- Understanding of the problem
- Proper data handling and evaluation strategy
- Awareness of data leakage risks
- Basic application of explainable AI

## 2. Dataset

**NASA Lithium-Ion Battery Aging Dataset** (processed cycle-level version)

- Source: NASA Ames Prognostics Center of Excellence (PCoE)
- 34 lithium-ion 18650 cells
- 1415 discharge cycles in total
- Columns used: `battery_id`, `cycle`, `voltage`, `temperature`, `capacity`, `soh`, `rul`
- SOH is defined relative to the first-cycle capacity of each battery (values slightly above 1.0 are therefore possible and normal)

Data: [NASA Battery Degradation Dataset ](https://www.kaggle.com/datasets/yashxss/nasa-battery-cycle-level-dataset/data)

## 3. Method Summary

### Model Inputs
- `voltage` (aggregated characteristic voltage of the cycle)
- `temperature`
- `capacity` (discharge capacity of the cycle)

### Model Output
- Continuous SOH value

### Preprocessing
- No missing values present
- Standard scaling applied before model training
- No complex feature engineering (kept simple for interpretability)

### Training / Testing Strategy
**GroupKFold (5-fold) by `battery_id`**

Entire batteries are held out in each fold.  
This is the correct way to evaluate generalization to unseen cells and avoids the common leakage that occurs when cycles from the same battery appear in both train and test sets.

### Models
1. Random Forest Regressor (main model)
2. XGBoost (optional comparison)

### Evaluation Metrics
- MAE
- RMSE
- R²
- MAPE

### Explainability
- Feature importance from Random Forest
- SHAP values

## 4. Key Findings

### Quantitative Results (GroupKFold by battery)

| Model                  | MAE             | RMSE            | R²              | MAPE           |
|------------------------|-----------------|-----------------|-----------------|----------------|
| RF (no cycle)          | 0.0606 ± 0.0265 | 0.0814 ± 0.0387 | 0.4890 ± 0.3383 | 7.80% ± 3.92%  |
| RF (with cycle)        | 0.0514 ± 0.0267 | 0.0805 ± 0.0461 | 0.3995 ± 0.5976 | 6.44% ± 3.56%  |
| XGBoost (no cycle)     | 0.0625 ± 0.0242 | 0.0863 ± 0.0356 | 0.4266 ± 0.3229 | 7.96% ± 3.46%  |

### Main Observations
- Random Forest without the `cycle` feature gives reasonable performance while remaining more realistic for deployment.
- Adding the `cycle` number improves some metrics because SOH is strongly correlated with cycle count under fixed laboratory conditions. This is a classic form of **data leakage**.
- Performance varies significantly across folds (some batteries are much harder to predict than others). This is expected when testing on completely unseen cells.
- `capacity` is by far the most important feature (importance ≈ 0.63), followed by `voltage` (≈ 0.25) and `temperature` (≈ 0.12).

### Feature Importance (Random Forest)
```
capacity       0.6315
voltage        0.2490
temperature    0.1195
```

## 5. Data Leakage Discussion

Under constant-current laboratory cycling, capacity (and therefore SOH) decreases almost monotonically with cycle number.  
Including `cycle` as a feature therefore leaks future information about the degradation trajectory.

In a real Battery Management System we cannot rely on a global cycle counter that starts from the beginning of life for every cell.  
The experiments with and without `cycle` illustrate this important point.

## 6. Limitations

- Laboratory constant-current data only (not representative of real EV driving profiles)
- Limited number of cells and chemistry diversity
- Capacity is used as an input feature (in practice it must often be estimated)
- No multi-temperature or multi-chemistry transfer learning in this preliminary study
- High variance between folds shows that generalization across different batteries remains challenging

## 7. Possible Future Improvements

- Physics-informed features (Incremental Capacity, Differential Voltage, etc.)
- Transfer learning across different temperatures and cell types
- Online / recursive SOH estimation suitable for BMS
- Combination with state-of-charge (SOC) estimation
- Uncertainty quantification (Gaussian Process, Quantile Regression, Ensemble)
- Stronger sequential models (LSTM / Transformer) that can capture temporal degradation patterns without relying on cycle number

## 8. References

- Saha, B., & Goebel, K. (2007). Battery data set. NASA AMES prognostics data repository.
- Original NASA PCoE repository and various public processed versions (Kaggle cycle-level CSVs).

