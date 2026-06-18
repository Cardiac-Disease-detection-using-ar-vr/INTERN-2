# Echo Branch (Primary Contribution)

The primary contribution of this work is the design and implementation of the **Echo Branch**, which predicts Left Ventricular Ejection Fraction (LVEF) directly from echocardiography frames and converts the predicted EF into cardiac severity probabilities for downstream fusion.

The complete pipeline consists of data preprocessing, feature extraction, EF regression, explainability, and standardized output generation.

---

## Data Processing Pipeline

Input Dataset:

- EchoNet-Dynamic

For every echocardiography video:

1. Read video metadata from `FileList.csv`.
2. Locate End-Diastolic (ED) and End-Systolic (ES) frame indices.
3. Extract ED and ES frames using OpenCV.
4. Resize frames to **224 × 224**.
5. Normalize using ImageNet statistics.
6. Construct paired `(ED, ES)` samples for training and inference.

---

## Echo Encoder Architecture

Backbone:

- EfficientNet-B3 (ImageNet pretrained)

Training strategy:

- Entire backbone frozen.
- Final EfficientNet block unfrozen for fine-tuning.
- Adaptive Average Pooling.
- Projection Head:

```
1536
 ↓
Linear
 ↓
ReLU
 ↓
Dropout (0.3)
 ↓
512-dimensional feature vector
```

The encoder converts each echocardiography frame into a compact 512-dimensional feature representation while preserving clinically relevant structural information.

---

## Echo Regression Model

The regression network processes both ED and ES frames.

```
ED Frame
        \
         \
          EchoEncoder
         /
ES Frame

↓

512-d ED Feature

+

512-d ES Feature

↓

Concatenation (1024-d)

↓

Linear Regression Head

↓

Predicted Ejection Fraction (0–100%)
```

A final clamp operation constrains predictions to the physiologically valid EF range of **0–100%**.

---

## Severity Prediction

The predicted EF is converted into cardiac severity probabilities using clinically defined thresholds.

| Predicted EF | Severity |
|--------------|----------|
| ≥ 50% | Normal |
| 40–49% | Mild Dysfunction |
| < 40% | Severe Dysfunction |

These probabilities are later consumed by the Late Fusion module.

---

## Explainability

Three explainability techniques were implemented.

### Integrated Gradients

- Captum
- ECG embedding attribution

### Grad-CAM

Applied to:

- EfficientNet-B3 final convolution block

Visualizations generated for:

- End-Diastolic frame
- End-Systolic frame

Outputs:

```
outputs/

heatmaps/

patientID_ed.png

patientID_es.png
```

---

### SHAP

DeepExplainer applied to:

- ECG classifier head

Produces:

- Global embedding importance
- Top contributing embedding dimensions

---

## Model Outputs

For every patient, the Echo branch produces:

- Predicted EF
- Severity probabilities
- Predicted severity class
- Grad-CAM heatmaps
- Explainability metadata

These outputs are exported as standardized JSON files for downstream processing.

---

## Exported Artifacts

### Trained Model

```
models/

Frozen_e1.pth
```

### Heatmaps

```
outputs/

heatmaps/

patientID_ed.png

patientID_es.png
```

### Base Score Files

```
outputs/

base_scores/

patientID_base.json
```

### JSON Schema

```
schemas/

base_score.schema.json
```

---

## Current Performance

Current validation performance:

- EF Regression
- MAE ≈ 6–7
- R² ≈ 0.30

The current implementation satisfies the complete inference and explainability pipeline but requires additional model optimization to reach the target performance specified in the project quality bar.

---

## Integration with ECG Branch

The Echo branch is completely modular.

It exposes:

- Predicted EF
- Severity probabilities
- Explainability outputs

These are integrated with the frozen ECG branch through a Late Fusion strategy.

No architectural changes to the Echo model are required once paired ECG–Echo datasets become available.

Only the fusion module requires modification.