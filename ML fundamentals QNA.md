# ML Fundamentals — Interview Q&A

A reference covering core ML theory, optimization and training, losses and metrics, representation, classic algorithms, feature engineering, deep-learning practicals, and applied scenarios — with bridge questions linking classic ML to GenAI.

## Table of Contents

- [Core ML Theory](#core-ml-theory)
- [Optimization & Training](#optimization--training)
- [Loss Functions & Metrics](#loss-functions--metrics)
- [Representation & Data](#representation--data)
- [Bridge Questions (Classic ML → GenAI)](#bridge-questions-classic-ml--genai)
- [Classic ML Algorithms (Most-Asked)](#classic-ml-algorithms-most-asked)
- [Feature Engineering & Data Prep (Most-Asked)](#feature-engineering--data-prep-most-asked)
- [Deep Learning Practical (Most-Asked)](#deep-learning-practical-most-asked)
- [Applied / Scenario Block](#applied--scenario-block)

---

## Core ML Theory

### Bias-variance tradeoff.

Error = bias² + variance + irreducible noise. High bias = underfit (too simple, wrong assumptions); high variance = overfit (memorizes noise, sensitive to data). More capacity/less regularization → ↓bias ↑variance. Manage via model complexity, regularization, more data, ensembling. In LLMs the framing shifts (overparameterized models generalize despite zero train loss — double descent), but the tradeoff still governs small fine-tuning sets.

### Overfitting: detect and prevent.

Detect: train loss ≪ val loss, or val metric rising then falling. Prevent: more/augmented data, regularization (L2, dropout), early stopping, simpler model, cross-validation. Fine-tuning analogs: LoRA (fewer trainable params), low LR, fewer epochs, hold-out eval.

### L1 vs L2 regularization.

- **L1 (Lasso, +λ‖w‖₁):** drives weights to exactly 0 → sparse, feature selection.
- **L2 (Ridge, +λ‖w‖₂²):** shrinks weights smoothly, never exactly 0, handles correlated features.

Elastic net = both. L2 ≈ weight decay (the standard in deep nets / AdamW).

### Generative vs discriminative models.

Discriminative learns P(y|x) directly (logistic regression, BERT classifier) — better when you only need the decision boundary. Generative learns P(x,y) or P(x) (naive Bayes, GPT, diffusion) — can sample new data. LLMs are generative: trained to model P(next token | context).

---

## Optimization & Training

### Gradient descent variants.

- **Batch:** full dataset per step — stable, slow, memory-heavy.
- **SGD:** one sample — noisy, escapes local minima, slow convergence.
- **Mini-batch:** the practical default (32–1024).

Update: `w ← w − η∇L`. η too high → divergence; too low → slow/stuck.

### Backpropagation.

Reverse-mode autodiff: forward pass caches activations, backward pass applies chain rule to compute ∂L/∂w layer by layer, reusing downstream gradients. Cost ≈ one forward pass. Vanishing/exploding gradients arise from repeated multiplication through depth — mitigated by residual connections, normalization, careful init.

### Adam / AdamW.

Adam = momentum (1st moment) + per-parameter adaptive LR (2nd moment, RMSProp-style). m̂, v̂ bias-corrected; `w ← w − η·m̂/(√v̂+ε)`. AdamW decouples weight decay from the gradient update (correct L2) — standard for transformers. Pair with warmup + cosine decay.

### Vanishing/exploding gradients.

Vanishing: gradients shrink through depth/time → early layers stop learning (sigmoid/tanh saturation, deep RNNs). Exploding: blow up → NaNs. Fixes: ReLU/GELU, residuals, LayerNorm/BatchNorm, proper init (Xavier/He), gradient clipping, LSTM gating (pre-transformer).

### Normalization: BatchNorm vs LayerNorm.

BatchNorm normalizes across the batch per feature — depends on batch stats, weak for small/variable batches and sequences. LayerNorm normalizes across features per token — batch-independent, the transformer default. RMSNorm (Llama) drops mean-centering for speed.

---

## Loss Functions & Metrics

### Cross-entropy loss.

`−Σ y·log(ŷ)`. Penalizes confident wrong predictions heavily; for classification it pairs with softmax. LLM training = token-level cross-entropy (= maximize log-likelihood of next token). Perplexity = `exp(mean CE)` — the intrinsic LM metric.

### Pick a classification metric.

Accuracy fails on imbalance. Precision = TP/(TP+FP) — cost of false alarms (spam). Recall = TP/(TP+FN) — cost of misses (disease, fraud). F1 = harmonic mean — balance. ROC-AUC = ranking quality across thresholds (balanced); PR-AUC better under heavy imbalance. Choose by the cost of each error type.

### Softmax + why temperature.

`softmax(zᵢ) = e^{zᵢ/T}/Σe^{zⱼ/T}` → probability distribution. T<1 sharpens, T>1 flattens — exactly the LLM sampling knob. Numerically: subtract max(z) before exp to avoid overflow.

---

## Representation & Data

### Word2Vec / embeddings intuition.

Map tokens to dense vectors where geometric closeness ≈ semantic similarity ("king−man+woman≈queen"). Word2Vec (skip-gram/CBOW) learns from co-occurrence; static (one vector per word). Transformers give contextual embeddings (same word, different vector by context) — the basis for modern retrieval/RAG.

### Curse of dimensionality.

As dims grow, data sparsifies, distances concentrate (everything ~equidistant), volume explodes → distance metrics degrade. Why we use dimensionality reduction (PCA), learned low-dim embeddings, and ANN indexes (HNSW/IVF) instead of brute-force search in vector DBs.

### Handle class imbalance.

Resampling (SMOTE/oversample minority, undersample majority), class-weighted loss, threshold tuning, focal loss, and evaluate with PR-AUC/F1 not accuracy. For LLM fine-tuning: balance or weight examples so rare intents aren't drowned out.

---

## Bridge Questions (Classic ML → GenAI)

### Why did transformers replace RNNs/LSTMs?

RNNs process sequentially (no parallelism, slow training) and lose long-range signal through vanishing gradients despite gating. Attention is parallel over the sequence and gives O(1) path length between any two tokens → captures long-range dependencies and scales on GPUs. Cost: O(n²) attention vs O(n) recurrence.

### Self-supervised learning — why it unlocked LLMs.

Labels come free from the data itself (next-token / masked-token prediction), so training scales to internet-sized corpora without annotation. Pretrain general representations self-supervised → adapt with small labeled sets (fine-tune) or none (prompting). This is the whole pretrain→adapt paradigm.

### Train/val/test + cross-validation.

Train fits params, val tunes hyperparams/early-stop, test is the untouched final estimate. K-fold CV averages over k splits for low-data robustness. Pitfall: data leakage (preprocessing on full set, temporal leakage, near-duplicates) — fit transforms on train only. In LLM work, watch benchmark contamination (eval data in pretraining).

---

## Classic ML Algorithms (Most-Asked)

### Bagging vs boosting.

Both ensembles reducing error.

- **Bagging (Random Forest):** train many models in parallel on bootstrap samples, average/vote — reduces variance, robust to overfitting, parallelizable.
- **Boosting (XGBoost, LightGBM, AdaBoost):** train sequentially, each model corrects the previous's errors (fit residuals/reweight) — reduces bias, higher accuracy, but overfits if unchecked and is sequential.

Bagging for variance/noise; boosting for max accuracy on structured/tabular data (still the Kaggle default).

### Random Forest vs gradient boosting — when to pick.

RF: parallel, hard to overfit, less tuning, good baseline, gives feature importance. GBM (XGBoost/LightGBM): usually higher accuracy on tabular, but sensitive to hyperparams (learning rate, depth, n_estimators) and slower to tune. Pick RF for a fast robust baseline; GBM when squeezing out accuracy and you can tune. Both beat deep nets on most tabular problems.

### Logistic regression essentials.

Linear model through a sigmoid → probability; decision boundary is linear in feature space. Trained by minimizing log loss (cross-entropy), not MSE (non-convex + bad gradients with sigmoid). Coefficients = log-odds, interpretable. Regularize (L1/L2) to control overfit/sparsity. Strong, explainable baseline for binary classification.

### Decision tree splitting.

Greedily split on the feature/threshold that most reduces impurity: Gini (1−Σpᵢ²) or entropy (−Σpᵢlog pᵢ); regression uses variance/MSE reduction. Splits until a stop criterion (depth, min samples). Pros: interpretable, no scaling needed, handles non-linearity. Con: high variance/overfits — pruning or ensembling (RF/GBM) fixes it.

### SVM & the kernel trick.

Finds the max-margin hyperplane separating classes; support vectors are the boundary points. Kernel trick: implicitly map to higher-dim space (RBF, polynomial) via a kernel function so non-linearly-separable data becomes separable — without computing the mapping. C controls margin-vs-misclassification tradeoff. Strong for small/medium high-dim data; scales poorly to huge datasets.

### KNN & Naive Bayes.

- **KNN:** no training; classify by majority of k nearest neighbors (distance-based) — simple, but slow at inference and curse-of-dimensionality sensitive; scale features.
- **Naive Bayes:** applies Bayes' rule assuming feature independence — fast, works well for text/spam despite the "naive" assumption; needs little data.

Both are common baseline/contrast questions.

### PCA / dimensionality reduction.

PCA projects data onto orthogonal directions (principal components = eigenvectors of the covariance matrix) that capture maximum variance; keep top-k to reduce dims while retaining most variance. Use for visualization, denoising, speed, multicollinearity. Linear only — t-SNE/UMAP for non-linear visualization. Standardize first (variance-sensitive).

### K-means & choosing k.

Partition into k clusters by minimizing within-cluster variance; iterate assign → recompute centroids until stable. Choose k via elbow method (inertia vs k) or silhouette score. Sensitive to init (use k-means++), scale, and assumes spherical clusters. Mention DBSCAN for arbitrary shapes/noise.

---

## Feature Engineering & Data Prep (Most-Asked)

### Standardization vs normalization — and which models need it.

Standardization: (x−μ)/σ → mean 0, unit variance. Normalization (min-max): scale to [0,1]. Needed by distance/gradient-based models: KNN, SVM, k-means, PCA, neural nets, logistic/linear with regularization. Not needed by tree-based models (split-invariant to scale). Fit the scaler on train only, apply to test (avoid leakage).

### Encoding categorical features.

- **One-hot:** nominal, low cardinality — no false ordinality, but dimension blowup.
- **Label/ordinal:** ordered categories.
- **Target/mean encoding:** high cardinality — encode by target mean (leak-prone; use CV/smoothing).
- **Embeddings:** very high cardinality / deep models.
- **Frequency encoding** as a cheap alternative.

Pick by cardinality + whether order is meaningful + model type.

### Handling missing data & outliers.

Missing: drop (if few/MCAR), impute (mean/median/mode, KNN, model-based), or add a missing-indicator flag (missingness can be signal). Outliers: detect (IQR, z-score, isolation forest), then cap/winsorize, transform (log), or remove if erroneous. Decide by mechanism (MCAR/MAR/MNAR) and model robustness — trees tolerate outliers, linear models don't.

### Feature selection.

Filter (correlation, mutual information, chi²) — fast, model-agnostic. Wrapper (RFE, forward/backward) — model-driven, expensive. Embedded (L1/Lasso, tree importance) — selection during training. Goals: reduce overfitting, speed, interpretability. Watch multicollinearity (drop/aggregate correlated features; inflates variance of linear coefficients, VIF to detect).

---

## Deep Learning Practical (Most-Asked)

### Dropout & batchnorm at inference (gotcha).

Dropout: randomly zero activations during training (regularization, prevents co-adaptation); at inference it's OFF — all units active, scaled appropriately. BatchNorm: uses batch mean/var during training but running (population) averages at inference. Forgetting to switch to eval mode (`model.eval()`) is a classic production bug → wrong predictions.

### Activation functions.

Sigmoid/tanh: saturate → vanishing gradients, mostly legacy (sigmoid still for binary output). ReLU: max(0,x) — cheap, no positive saturation, default for CNNs/MLPs; dying-ReLU risk → Leaky/Parametric ReLU. GELU/SiLU: smooth, default in transformers. Softmax: multi-class output layer. Pick ReLU-family for hidden layers, task-appropriate output activation.

### Learning rate & schedules.

Most important hyperparameter. Too high → diverge/oscillate; too low → slow/stuck. Warmup (ramp up early to stabilize) + decay (cosine/step/linear) is standard for transformers. Adaptive optimizers (Adam) help but still need a base LR. Use LR-range test / scheduler; watch the loss curve.

### Hyperparameter tuning.

Grid search: exhaustive, exponential cost. Random search: samples the space — more efficient, finds good configs faster (most dims don't matter). Bayesian optimization: models the objective to pick promising configs — sample-efficient for expensive training. Always tune on validation/CV, not test. Early-stop weak trials to save compute.

---

## Applied / Scenario Block

### Diagnose an overfitting model (interview drill).

Symptom: train metric ≫ val metric. Diagnose: plot learning curves (train vs val over epochs/data). Fixes in order: more/augmented data, stronger regularization (L2/dropout), reduce model complexity, early stopping, feature selection, cross-validate. If both train and val are poor → underfitting (add capacity/features, train longer, reduce regularization). Always change one lever and re-measure.

### Model works offline but fails in production.

Likely causes: training-serving skew (different preprocessing/feature code paths), data drift (input distribution shifted), data leakage inflated offline scores, label delay, or covariate shift. Debug: log + compare prod feature distributions vs training, check the serving pipeline matches training transforms, re-evaluate on recent data. Prevent: shared feature pipeline/feature store, drift monitoring, retraining triggers, regression eval set.

### Build a fraud-detection model end to end (system thinking).

Frame: highly imbalanced binary classification, cost-asymmetric (missing fraud ≫ false alarm). Data: features (transaction, velocity, device, historical), avoid leakage (no future info), temporal split (not random). Model: gradient boosting baseline; handle imbalance (class weights, focal loss, careful resampling). Metric: PR-AUC / recall at fixed precision, not accuracy; tune threshold by business cost. Production: low-latency scoring, drift monitoring, feedback loop on confirmed fraud, human review queue for borderline. Mention adversarial/concept drift — fraud evolves.

### Pick a model for a given problem (how to answer).

Tabular + need accuracy/interpretability → gradient boosting / logistic regression. Images → CNN / pretrained vision model. Text/sequence → transformer (or fine-tune an LLM). Small data → simpler model + regularization + transfer learning. Real-time/low-latency → lightweight model / distillation. Always start with a simple baseline, measure on a proper eval split, then justify added complexity with metrics + cost/latency tradeoffs.
