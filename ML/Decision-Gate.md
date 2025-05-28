# HTB Operation Blackout - Decision Gate

## Challenge Overview

**Challenge Name:** Decision Gate  
**Category:** Machine Learning / Reverse Engineering  
**Difficulty:** Medium  
**Flag:** `HTB{tr33_p4th_tr4c3d_and_tr1gg3r3d}`

During a breach into a Volnayan AI research node, Task Force Phoenix uncovered a dormant decision system—its logic locked behind a concealed execution path. Intelligence suggests it was used to authorize classified operations. The correct path must be uncovered before it's lost to blackout.

## Given Files

- `tree_model.joblib` - A serialized machine learning model (26.5MB)
- `example_input.npy` - A NumPy array containing sample input data

## Technical Analysis

### Model Structure Analysis
```python
import joblib
from sklearn.tree import export_text

# Load the model
model = joblib.load('tree_model.joblib')

# Basic model information
print(f"Model type: {type(model)}")
print(f"Number of features: {model.n_features_in_}")
print(f"Number of classes: {model.n_classes_}")
print(f"Classes: {model.classes_}")
```

**Key Findings:**
- Model type: DecisionTreeClassifier
- Number of features: 5
- Number of classes: 1000
- Classes: `['UNLOCK_FLAG_PATH', 'class_000', 'class_001', ..., 'class_998']`

### Vulnerability Assessment
The first class `UNLOCK_FLAG_PATH` represents the target execution path hidden within the decision tree structure.

## Attack Chain

### 1. Example Input Analysis
```python
import numpy as np
example_input = np.load('example_input.npy')
print(f"Shape: {example_input.shape}")
print(f"Values: {example_input}")

# Test prediction
prediction = model.predict(example_input)
print(f"Prediction: {prediction}")
```

**Results:**
- Shape: (1, 5) - 5-dimensional input required
- Values: `[[ 1.71961357  2.39369826 -3.81596672  0.53771899  0.87461951]]`
- Prediction: `['class_053']` - Not the target path

### 2. Decision Tree Structure Analysis
```python
tree_rules = export_text(model, feature_names=[f'feature_{i}' for i in range(5)])
print(tree_rules)
```

**Critical Discovery:**
```
|--- feature_1 <= -5000002.47
|   |--- class: UNLOCK_FLAG_PATH
|--- feature_1 >  -5000002.47
|   |--- [complex tree with 999 other classes]
```

The very first decision triggers the unlock path when `feature_1 <= -5000002.47`.

### 3. Exploit Development
```python
# Create input that triggers UNLOCK_FLAG_PATH
unlock_input = np.array([[0.0, -5000003.0, 0.0, 0.0, 0.0]])

# Verify it works
prediction = model.predict(unlock_input)
print(f"Prediction: {prediction}")
```

**Output:** `['UNLOCK_FLAG_PATH']` - Successfully triggers the unlock condition.

### 4. Remote Exploitation
```bash
# Connect to server
nc 94.237.123.198 59858

# Server expects comma-separated 5D float vector
echo "0.0,-5000003.0,0.0,0.0,0.0" | nc 94.237.123.198 59858
```

**Server Response:**
```
Submit a 5D float vector (comma-separated) to unlock the flag.
Input vector: Correct! Flag: HTB{tr33_p4th_tr4c3d_and_tr1gg3r3d}
```

## Key Technical Concepts

### Decision Tree Analysis
- Decision trees make binary decisions based on feature thresholds
- The `export_text()` function reveals the complete decision logic
- Hidden execution paths can be identified through tree structure analysis

### Machine Learning Model Security
- Serialized models can be reverse-engineered to reveal logic
- Unusual threshold values often indicate intentional backdoors
- Feature manipulation can trigger specific classification paths

### Model Inspection Techniques
- Use `joblib.load()` to deserialize scikit-learn models
- Examine `model.classes_` to understand possible outputs
- Analyze decision tree structure with `export_text()`

## Mitigation Recommendations

### Model Security
- Implement model integrity verification using cryptographic signatures
- Use model encryption to prevent unauthorized analysis
- Apply obfuscation techniques to decision tree structures

### Access Control
- Restrict access to serialized model files
- Implement authentication before model inference
- Monitor for unusual input patterns that might trigger backdoors

### Input Validation
- Implement bounds checking on feature inputs
- Use anomaly detection to identify suspicious input vectors
- Log all model predictions for audit purposes

## Tools Used

- Python libraries: joblib, numpy, scikit-learn
- Network tools: netcat (nc)
- Decision tree visualization and inspection tools

## Write-Up Credit: [binchickens69](https://ctf.hackthebox.com/user/profile/605069)