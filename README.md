# Time Series Clustering Module

This repository implements a **time series clustering component**, designed to be used as the final stage of preprocessing in a larger forecasting pipeline.

The goal is to group series having the same characteristics. Clusters will be used to:
- Decide local vs global model.
- Suggest a suitable model family.
- Explain why the series should/should not be trained together.

---

## Design philosophy

Clustering quality is governed by two competing forces:

- **Inner variance**: how tight samples are within a cluster  
- **Outer variance**: how different clusters are from each other  

This module is designed as a variance-controlled pipeline, where each step has a clear role in shaping these two quantities.

Key principles:

- Feature processing decisions are config-driven.
- Redundancy and noise are removed before clustering.
- PCA is used as a geometry filter, not a compression trick.
- Clustering is evaluated on geometry, explained on the original features.

---

## High-level pipeline

```
Raw time series
↓
Feature extraction
↓
Feature hygiene & validation
↓
Dimensionality reduction (PCA)
↓
Clustering & validation
↓
Feature explanation
↓
Refinement

```

---

## Repository structure

```

ts_clustering/
├── README.md
├── requirements.txt
├── setup.py
├── config/
│   └── default.yaml
├── data/
│   ├── raw/                  # raw time‑series or tsfeatures (not versioned)
│   ├── processed/            # cleaned & transformed features
│   └── intermediate/         # results of each step (e.g., PCA embeddings)
├── clustering_component/
│   ├── data_loader.py        # functions to load tsfeature data
│   ├── feature
│   │   ├── compute_features.py
│   │   └──feature_hygiene.py       # hygiene & validation methods
│   ├── dimensionality/
│   │   └── pca.py             # PCA fitting & loadings interpretation
│   ├── clustering/
│   │   ├── clustering.py          # clustering algorithms
│   │   ├── validation.py          # silhouette, DBI, dispersion
│   │   └── explanation.py         # summary_stats, boxplot, ANOVA, MI, overlap analysis
│   ├── pipeline.py            # orchestrate the steps using classes/functions
│   ├── utils/                     # outlier_detection
│   └── results/                   # store params, validation & explaination result
└── tests/

````

---

## Step-by-step pipeline logic

### Step 1 - Feature hygiene  
**Goal: prevent artificial inner variance**

Problems addressed:
- Bad scaling → distances explode
- Heavy tails → PCA axes dominated by outliers
- Near-constant features → noise without signal

Actions:
- Standardization
- Log-transform heavy-tailed features
- Clipping extreme values
- Removing near-constant features

This step makes inner variance measurable and comparable.

---

### Step 2 - Feature validation  
**Goal: avoid fake variance**

Questions asked per feature:
- Does this feature vary randomly?
- Does it add instability inside clusters?

If yes:
- It inflates inner variance
- It blurs cluster boundaries
- It is removed

This step improves clustering *before* any algorithm is applied.

---

### Step 3 - Dimensionality reduction (PCA, etc.)  
**Goal: reduce inner variance caused by redundancy**

Key idea:
- Correlated features stretch clusters into elongated shapes
- PCA rotates the space to make clusters compact and isotropic

What PCA does here:
- Removes redundant variance
- Filters low-signal noise directions

Dimensionality reduction is used improve **geometry**, not semantics. It does not maximize outer variance. 

---

### Step 4 - Clustering  
**Goal: explicitly optimize inner variance**

Examples:
- K-means: minimizes within-cluster variance
- GMM: allows elliptical clusters
- Density-based: emphasize separation

This step:
- Directly minimizes inner variance
- Outer variance emerges indirectly

Algorithm choice affects the trade-off between compactness and separation.

---

### Step 5 - Cluster validation  
**Goal: evaluate inner vs outer balance**

Metrics:
- Silhouette score  
  - Inner distance ↓
  - Nearest-cluster distance ↑
- Davies-Bouldin index  
  - Penalizes loose clusters
  - Penalizes overlapping clusters

This step answers: Are clusters tight enough and sufficiently separated?

---

### Step 6 - Feature explanation  
**Goal: measure outer variance in original feature space**

Analysis performed on unscaled, cleaned features:
- Mean/median differences
- ANOVA / Kruskal–Wallis
- Mutual Information with cluster labels
- Overlap analysis (median separation vs IQR)

This step answers:
- Which features truly differentiate clusters?
- Which features are explanatory vs redundant?

---

### Step 7 - Refinement 
**Goal: rebalance variances**

- If **inner variance too high**:
  - Remove noisy features
  - Reduce PCA dimensions
  - Change clustering algorithm

- If **outer variance too low**:
  - Add or swap discriminative features
  - Adjust number of clusters
  - Rebalance feature groups

Refinement continues until:
- Clusters are compact
- Differences are explainable
- Results are stable under small perturbations

Refinement is executed by re-running the same pipeline with modified config.

Example:

* Removing a feature (e.g. `wpe`) is done by editing `features`
* PCA, clustering, validation, and explanation are automatically re-executed

---

## Configuration-driven design

All decisions are controlled via YAML config.

Example:

```yaml
features:
    - acf1
    - acf_seasonal
    - seasonality_strength
    - lumpiness
    - spike_strength
    - anomaly_ratio
    - wpe

hygiene:
  standardize: true
  log_transform: true
  clip_quantile: 0.99
  variance_threshold: 1e-4

reduction:
  method: pca
  variance_retained: 0.95

clustering:
  algorithm: kmeans
  k_range: [3, 8]
  random_state: 42

iteration:
  allow_feature_ablation: true
  ablation_scope: base
````
