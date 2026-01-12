# Network Intrusion Detection - Classification Project

## Overview

This project implements a complete machine learning pipeline for **network intrusion detection** using the KDD Cup 1999 dataset. The goal is to classify network connections as either **normal traffic** or **attacks**, which is a critical task in cybersecurity.

I chose this as a binary classification problem because in real-world scenarios, the primary concern is detecting whether a connection is malicious or not.

---

## Dataset Description

The **KDD Cup 1999** dataset is a well-known benchmark for intrusion detection systems. It contains network connection records captured from a simulated military network environment.

### Key Statistics
- **Total samples:** 494,021 network connections
- **Features:** 41 attributes describing each connection
- **Target:** Connection label (normal or attack type)

### Feature Categories

The 41 features are organized into four logical groups:

| Category | Features | What They Capture |
|----------|----------|-------------------|
| **Basic TCP** | duration, protocol_type, service, flag, src_bytes, dst_bytes | Fundamental connection properties like duration, protocol used, and bytes transferred |
| **Content** | hot, num_failed_logins, logged_in, root_shell, etc. | Information extracted from the packet payload - login attempts, shell access, file operations |
| **Time-based Traffic** | count, srv_count, serror_rate, same_srv_rate, etc. | Patterns in the last 2 seconds - how many connections to same host/service |
| **Host-based Traffic** | dst_host_count, dst_host_srv_count, dst_host_same_srv_rate, etc. | Patterns over last 100 connections to the same destination host |

### Class Distribution

The dataset is **imbalanced**:
- **Attack connections:** ~80% (397,924 samples)
- **Normal connections:** ~20% (97,097 samples)

This imbalance is important because it affects how we evaluate our models and which cross-validation strategy we use.

---

## Preprocessing Pipeline

Before feeding data to the classifiers, I performed several preprocessing steps. Here's what I did for each feature type and why:

### 1. Categorical Features → One-Hot Encoding

**Features:** `protocol_type`, `service`, `flag`

**Why:** These are nominal categories with no inherent order. For example, `tcp`, `udp`, and `icmp` are just different protocols - one isn't "greater" than another. One-hot encoding creates binary columns for each category, allowing algorithms to treat them properly.

**Result:** 3 original columns expanded to ~84 binary columns

### 2. Continuous Numerical Features → Standard Scaling

**Features:** `duration`, `src_bytes`, `dst_bytes`, `count`, `srv_count`, etc. (16 features)

**Why:** These features have vastly different scales. For example:
- `duration` ranges from 0 to thousands of seconds
- `src_bytes` can be millions of bytes
- `count` typically ranges from 0 to 500

Without scaling, algorithms like KNN and MLP would be dominated by large-scale features. Standard scaling transforms each feature to have mean=0 and standard deviation=1.

### 3. Binary Features → No Change (Passthrough)

**Features:** `land`, `logged_in`, `root_shell`, `su_attempted`, `is_host_login`, `is_guest_login`

**Why:** These are already binary (0 or 1). No transformation needed.

### 4. Rate Features → No Change (Passthrough)

**Features:** `serror_rate`, `srv_serror_rate`, `same_srv_rate`, etc. (15 features)

**Why:** These are already percentages in the range [0, 1]. They're already normalized by definition.

### 5. Dropped Features

**Feature:** `num_outbound_cmds`

**Why:** This feature has zero variance - every single value in the dataset is 0. A constant feature provides no discriminative information, so I removed it.

---

## Performance Metrics

I used four metrics to evaluate model performance:

| Metric | What It Measures |
|--------|------------------|
| **Accuracy** | Overall percentage of correct predictions |
| **Precision** | Of all predicted attacks, how many were actually attacks? |
| **Recall** | Of all actual attacks, how many did we detect? |
| **F1-Score** | Harmonic mean of precision and recall |

### Why Recall is the Most Important Metric

In intrusion detection, **missing an attack is far more dangerous than a false alarm**.

Consider the consequences:
- **False Negative (Missing an attack):** A real attack goes undetected. This could lead to data breaches, system compromise, or security incidents.
- **False Positive (False alarm):** We flag normal traffic as an attack. This causes extra investigation work but no actual security damage.

Therefore, I prioritized **Recall** (also called Sensitivity or True Positive Rate) because it measures how well our model catches all actual attacks. A high recall means fewer attacks slip through undetected.

I also monitor F1-Score as a secondary metric to ensure precision doesn't drop too low (too many false alarms would make the system impractical).

---

## Classification Algorithms

I implemented four different classifiers to compare their performance:

### 1. Decision Tree (Mandatory)

**How it works:** Builds a tree structure by recursively splitting data based on feature thresholds. At each node, it finds the best feature and threshold to separate classes.

**Advantages:**
- Highly interpretable - you can visualize the decision rules
- Fast training and prediction
- Handles both numerical and categorical features

**Key parameters tuned:**
- `max_depth`: Controls how deep the tree can grow
- `criterion`: Splitting criterion (gini or entropy)
- `min_samples_split`: Minimum samples required to split a node

### 2. Random Forest

**How it works:** An ensemble of many decision trees. Each tree is trained on a random subset of data (bagging) and considers only a random subset of features at each split. Final prediction is by majority vote.

**Advantages:**
- Reduces overfitting compared to single decision tree
- Robust to noise and outliers
- Provides feature importance rankings

**Key parameters tuned:**
- `n_estimators`: Number of trees in the forest
- `max_depth`: Maximum depth of each tree
- `max_features`: Number of features to consider at each split

### 3. K-Nearest Neighbors (KNN)

**How it works:** Classifies a sample based on the majority class among its k nearest neighbors in the feature space. "Nearest" is defined by a distance metric (Euclidean, Manhattan, etc.).

**Advantages:**
- Simple and intuitive
- No training phase - all computation at prediction time
- Naturally handles multi-class problems

**Key parameters tuned:**
- `n_neighbors`: Number of neighbors to consider
- `weights`: Uniform or distance-weighted voting
- `metric`: Distance measure (Euclidean, Manhattan)

### 4. Multi-Layer Perceptron (MLP)

**How it works:** A neural network with input layer, one or more hidden layers, and output layer. Each neuron applies a weighted sum followed by a non-linear activation function. The network learns through backpropagation.

**Advantages:**
- Can learn complex non-linear relationships
- Flexible architecture
- Good for high-dimensional data

**Key parameters tuned:**
- `hidden_layer_sizes`: Number and size of hidden layers
- `activation`: Activation function (relu, tanh)
- `learning_rate_init`: Initial learning rate

---

## Cross-Validation Strategy

### Method: Stratified 5-Fold Cross-Validation

**How it works:** The training data is split into 5 equal parts (folds). The model is trained on 4 folds and tested on the remaining 1 fold. This process repeats 5 times, each time with a different fold as the test set. Final metrics are averaged across all 5 runs.

### Why Stratified?

Our dataset is imbalanced (~80% attack, ~20% normal). With standard K-Fold:
- Some folds might randomly have 85% attacks, others 75%
- This causes inconsistent model evaluation
- Metrics would vary significantly between folds

**Stratified K-Fold ensures each fold maintains the same 80/20 class ratio as the original data.** This provides:
- More reliable performance estimates
- Consistent representation of both classes in every fold
- Better generalization assessment

---

## Hyperparameter Tuning

I conducted **16 experiments** across all four algorithms, varying key parameters to find optimal configurations.

### Experiment Distribution

| Algorithm | # Experiments | Parameters Explored |
|-----------|---------------|---------------------|
| Decision Tree | 4 | max_depth, criterion, min_samples_split, min_samples_leaf |
| Random Forest | 4 | n_estimators, max_depth, min_samples_split, max_features |
| KNN | 3 | n_neighbors, weights, metric |
| MLP | 5 | hidden_layer_sizes, activation, learning_rate_init, solver |

### Key Findings from Tuning

**Decision Tree:**
- Unlimited depth leads to overfitting; depth 10-20 works best
- Gini and entropy criteria perform similarly
- Minimum samples constraints help prevent overfitting

**Random Forest:**
- More trees (100+) improve stability but with diminishing returns
- `max_features='sqrt'` improves generalization through diversity
- Moderate depth (15-25) balances complexity and accuracy

**KNN:**
- Small k (3-5) captures local patterns well
- Distance weighting outperforms uniform voting
- Manhattan and Euclidean distances perform comparably

**MLP:**
- Deeper networks (3 layers) can learn more complex patterns
- ReLU activation generally outperforms tanh
- Adaptive learning rate helps convergence

---

## Results Summary

All experiments are recorded in a results table with the format:

| Algorithm | Parameters | Accuracy | Recall | F1-Score |

### Best Performing Model

**Random Forest** with the following configuration achieved the best balance of high recall and F1-score:
- `n_estimators=100`
- `max_features='sqrt'`
- `max_depth=25`

This makes sense because:
1. Ensemble methods average out individual tree errors
2. Feature randomization reduces overfitting
3. Sufficient depth captures complex attack patterns

---

## How to Run the Notebook

1. Ensure you have the required libraries:
   ```bash
   pip install pandas numpy matplotlib seaborn scikit-learn
   ```

2. Place `kddcup.data_10_percent.csv` in the same directory as the notebook

3. Run cells sequentially - each cell is independent for easy debugging

4. Results and visualizations will be generated inline

---

## File Structure

```
Classification/
├── README.md                      # This file
├── kddcup.data_10_percent.csv     # Dataset
├── classification_project.ipynb   # Main notebook with all code
└── experiment_results.csv         # Generated results table (after running)
```

---

## Conclusion

This project demonstrates a complete machine learning workflow for network intrusion detection:

1. **Explored** the dataset to understand its characteristics
2. **Preprocessed** features appropriately based on their types
3. **Selected metrics** that align with the security-critical nature of the task
4. **Compared** multiple algorithms with different learning paradigms
5. **Used stratified cross-validation** to handle class imbalance
6. **Tuned hyperparameters** systematically to optimize performance

The final model achieves high recall, which is crucial for catching network attacks in real-world deployment scenarios.
