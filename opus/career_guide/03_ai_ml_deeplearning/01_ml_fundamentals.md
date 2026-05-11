# Machine Learning Fundamentals — Complete Deep Dive

## Why ML for Big Tech
Every FAANG company uses ML at its core. Even if you're not applying for ML roles, understanding ML concepts makes you a stronger engineer and opens ML Engineer / Data Scientist / MLOps paths.

---

## 1. Types of Machine Learning

```
┌───────────────────────────────────────────────────────────────────────┐
│                    Machine Learning                                     │
├────────────────┬──────────────────────┬───────────────────────────────┤
│  Supervised    │    Unsupervised      │    Reinforcement              │
│                │                      │                                │
│ Input: (X, y)  │ Input: X only        │ Input: Environment + Rewards  │
│ Learn mapping  │ Find patterns        │ Learn optimal actions         │
│                │                      │                                │
│ • Classification│ • Clustering         │ • Game playing (AlphaGo)     │
│ • Regression    │ • Dimensionality     │ • Robotics                   │
│                │   Reduction           │ • Recommendation             │
│                │ • Anomaly Detection  │ • RLHF (ChatGPT alignment)   │
└────────────────┴──────────────────────┴───────────────────────────────┘
```

---

## 2. Supervised Learning Algorithms

### Linear Regression
```
y = w₁x₁ + w₂x₂ + ... + wₙxₙ + b

Objective: Minimize Mean Squared Error (MSE)
  MSE = (1/n) Σ(yᵢ - ŷᵢ)²

Gradient Descent:
  w = w - α × ∂MSE/∂w   (α = learning rate)

Assumptions:
  - Linear relationship between features and target
  - Features are independent (no multicollinearity)
  - Homoscedasticity (constant variance of errors)
  
Regularization:
  L1 (Lasso): |w| → sparse weights (feature selection)
  L2 (Ridge): w² → small weights (prevents overfitting)
  Elastic Net: L1 + L2 combined
```

### Logistic Regression (Classification)
```
P(y=1|x) = σ(wᵀx + b) = 1 / (1 + e^-(wᵀx + b))

Output: Probability between 0 and 1
Decision boundary: Linear (hyperplane in feature space)
Loss: Binary Cross-Entropy = -[y·log(ŷ) + (1-y)·log(1-ŷ)]

Despite the name, it's a CLASSIFICATION algorithm!
```

### Decision Trees
```
         [Age > 30?]
         /          \
      Yes            No
       |              |
  [Income > 50K?]   [Student?]
   /        \        /      \
 Yes        No     Yes      No
  |          |      |        |
 Buy       Don't   Buy    Don't

Splitting criteria:
  - Gini Impurity: 1 - Σ(pᵢ²)  (probability of misclassification)
  - Entropy: -Σ(pᵢ · log₂(pᵢ)) (information gained by split)
  
Pros: Interpretable, handles non-linear, no feature scaling needed
Cons: Overfits easily, unstable (small data change → different tree)
```

### Random Forest (Ensemble)
```
Build N decision trees, each on:
  - Random subset of data (bagging)
  - Random subset of features (feature sampling)

Prediction: Majority vote (classification) or average (regression)

Why it works:
  - Each tree overfits differently
  - Averaging reduces variance (errors cancel out)
  - More robust than single tree
```

### Gradient Boosting (XGBoost, LightGBM)
```
Build trees SEQUENTIALLY. Each new tree corrects errors of previous ones.

Step 1: Fit model, compute residuals (errors)
Step 2: Fit new tree to predict residuals
Step 3: Add new tree's predictions to ensemble
Step 4: Repeat

XGBoost additions:
  - Regularized objective (prevents overfitting)
  - Efficient split finding (approximate algorithms)
  - Handles missing values natively
  - Column/row subsampling

XGBoost is the #1 algorithm for tabular data in competitions and production.
```

### Support Vector Machine (SVM)
```
Find hyperplane that maximizes margin between classes.

Key concepts:
  - Support vectors: Points closest to decision boundary
  - Margin: Distance between boundary and nearest points
  - Kernel trick: Map to higher dimensions for non-linear separation
    Linear: K(x,y) = xᵀy
    RBF: K(x,y) = exp(-γ||x-y||²)
    Polynomial: K(x,y) = (xᵀy + c)^d
```

### K-Nearest Neighbors (KNN)
```
Classify by majority vote of K nearest neighbors.
Distance: Euclidean, Manhattan, or Cosine.

No training phase (lazy learner)!
Prediction: O(n) — compare with every training point.
Optimization: KD-Tree or Ball Tree for faster lookup.

K too small → overfitting (sensitive to noise)
K too large → underfitting (too smooth)
```

---

## 3. Unsupervised Learning

### K-Means Clustering
```
1. Initialize K centroids randomly
2. Assign each point to nearest centroid
3. Recompute centroids as mean of assigned points
4. Repeat 2-3 until convergence

Choosing K:
  - Elbow method: Plot cost vs K, find "elbow"
  - Silhouette score: Measure cluster cohesion vs separation

Limitations:
  - Assumes spherical clusters
  - Sensitive to initialization (use K-Means++)
  - Need to specify K in advance
```

### PCA (Principal Component Analysis)
```
Reduce dimensions while preserving maximum variance.

Steps:
1. Standardize data (zero mean, unit variance)
2. Compute covariance matrix
3. Find eigenvectors (principal components)
4. Sort by eigenvalue (variance explained)
5. Keep top-K components

Use cases:
  - Visualization (100D → 2D)
  - Noise reduction
  - Feature extraction before ML model
  - Speed up training (fewer features)
```

---

## 4. Model Evaluation

### Metrics for Classification
```
Confusion Matrix:
              Predicted
              Pos    Neg
Actual Pos  [ TP  |  FN ]
Actual Neg  [ FP  |  TN ]

Accuracy:    (TP + TN) / Total
             ⚠️ Misleading with imbalanced data!

Precision:   TP / (TP + FP)  "Of predicted positives, how many are correct?"
             High precision = few false alarms

Recall:      TP / (TP + FN)  "Of actual positives, how many did we find?"
             High recall = few missed positives

F1 Score:    2 × (Precision × Recall) / (Precision + Recall)
             Harmonic mean — balances precision and recall

AUC-ROC:    Area under ROC curve (TPR vs FPR at all thresholds)
             0.5 = random, 1.0 = perfect

When to optimize what:
  - Spam detection: High precision (don't lose real emails)
  - Cancer detection: High recall (don't miss any cancer)
  - Balanced: F1 score
```

### Metrics for Regression
```
MAE:  Mean Absolute Error = (1/n) Σ|yᵢ - ŷᵢ|  (robust to outliers)
MSE:  Mean Squared Error = (1/n) Σ(yᵢ - ŷᵢ)²  (penalizes large errors)
RMSE: √MSE (same units as target)
R²:   1 - (SS_res / SS_tot) — proportion of variance explained (1 = perfect)
```

### Bias-Variance Trade-off
```
Total Error = Bias² + Variance + Irreducible Noise

High Bias (underfitting):
  - Model too simple
  - High training error AND test error
  - Fix: More features, complex model, less regularization

High Variance (overfitting):
  - Model too complex, memorizes training data
  - Low training error, HIGH test error
  - Fix: More data, regularization, simpler model, dropout

         Error
          ▲
          │  \  variance
          │   ─────────────\────────
          │         total    \
          │                   \
          │   /bias            \
          │──/─────────────────────
          │
          └─────────────────────────▶ Model Complexity
                 underfitting  |  overfitting
                            sweet spot
```

---

## 5. Feature Engineering

```
The most impactful skill in applied ML. Good features > complex models.

Techniques:
───────────
Encoding:
  - One-hot encoding: Categorical → binary columns
  - Label encoding: Categorical → integers (for trees)
  - Target encoding: Replace category with mean target

Scaling:
  - StandardScaler: (x - mean) / std → N(0,1)
  - MinMaxScaler: (x - min) / (max - min) → [0,1]
  - Required for: SVM, KNN, Neural Networks, Linear models
  - NOT needed for: Tree-based models

Feature Creation:
  - Polynomial features: x₁², x₁×x₂
  - Date features: day_of_week, is_weekend, month, hour
  - Text features: TF-IDF, word count, sentiment
  - Aggregations: rolling mean, group statistics

Feature Selection:
  - Correlation analysis (remove redundant)
  - Feature importance from Random Forest / XGBoost
  - Recursive Feature Elimination (RFE)
  - L1 regularization (automatic selection)
```

---

## 6. Cross-Validation & Hyperparameter Tuning

```
K-Fold Cross-Validation:
  Split data into K folds
  Train on K-1 folds, validate on 1
  Repeat K times, average results
  → More reliable estimate than single train/test split

Hyperparameter Tuning:
  Grid Search: Try all combinations (exhaustive, slow)
  Random Search: Random combinations (faster, often better!)
  Bayesian Optimization: Smart search using prior results
  Optuna: Modern framework, pruning of bad trials

Stratified K-Fold: Maintains class distribution in each fold
Time-Series Split: Respect temporal order (no future data leakage!)
```

---

## 7. ML Pipeline in Production

```
┌────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────┐
│  Data  │──▶│  Feature │──▶│  Model   │──▶│  Model   │──▶│Monitor │
│Ingestion│   │Engineering│   │ Training │   │ Serving  │   │& Retrain│
└────────┘   └──────────┘   └──────────┘   └──────────┘   └────────┘

Data Ingestion: Kafka, S3, database connectors
Feature Store: Feast, Tecton (compute once, reuse everywhere)
Training: SageMaker, Vertex AI, custom (GPU clusters)
Serving: FastAPI, TensorFlow Serving, Triton
Monitoring: Data drift detection, model performance tracking
Retraining: Scheduled or triggered by performance degradation
```
