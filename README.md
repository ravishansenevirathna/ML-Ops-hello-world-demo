# Hello-World MLOps

This repository demonstrates a tiny reproducible MLOps flow:
1. Train a small model (`train.py`) — writes `artifacts/model.pkl` and `artifacts/metrics.json`
2. Run predictions from the command line with `run_model.py --input "[5.1,3.5,1.4,0.2]"`
3. Start a minimal Flask app with `python src/app.py` that serves `/predict`
4. Build a Docker image with `docker build -t hello-mlops .`
5. CI trains the model and uploads artifacts

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CI/CD Pipeline                          │
│                     (.github/workflows/ci.yml)                  │
│                                                                 │
│  Triggers: Push/PR to main                                      │
│  Matrix: Python 3.11, 3.12                                      │
│  Steps: Checkout → Setup → Install → Train → Upload Artifacts  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Training Pipeline                          │
│                         (train.py)                              │
│                                                                 │
│  Dataset: Iris (sklearn)                                        │
│  Model: LogisticRegression (max_iter=200)                       │
│  Split: 80/20 train/test                                        │
│  Output:                                                        │
│    • artifacts/model.pkl (serialized model)                     │
│    • artifacts/metrics.json (accuracy)                          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                    ┌───────────────────┐
                    │   Model Artifacts │
                    │   (artifacts/)    │
                    └───────────────────┘
                        │              │
        ┌───────────────┘              └───────────────┐
        ▼                                              ▼
┌──────────────────┐                          ┌─────────────────┐
│   CLI Interface  │                          │   REST API      │
│  (run_model.py)  │                          │   (app.py)      │
│                  │                          │                 │
│  Input: JSON     │                          │  Framework:     │
│  [5.1,3.5,1.4,   │                          │  Flask          │
│   0.2]           │                          │                 │
│                  │                          │  Endpoints:     │
│  Output: JSON    │                          │  • GET /health  │
│  {"prediction":  │                          │  • POST /predict│
│   [0]}           │                          │                 │
│                  │                          │  Port: 5001     │
│  Model: joblib   │                          │  Host: 0.0.0.0  │
│  load            │                          │                 │
└──────────────────┘                          │  Auto-train if  │
                                              │  model missing  │
                                              └─────────────────┘
                                                      │
                                                      ▼
                                              ┌─────────────────┐
                                              │ Containerization│
                                              │  (Dockerfile)   │
                                              │                 │
                                              │  Base: python:  │
                                              │   3.12-slim     │
                                              │                 │
                                              │  Expose: 5001   │
                                              │                 │
                                              │  CMD: python    │
                                              │   app.py        │
                                              └─────────────────┘
```

### Component Details

#### 1. Training Pipeline (`train.py`)
- Loads Iris dataset from scikit-learn
- Trains LogisticRegression model with 80/20 split
- Saves serialized model using joblib
- Generates metrics.json with test accuracy

#### 2. CLI Interface (`run_model.py`)
- Command-line prediction tool
- Loads model from artifacts/model.pkl
- Accepts JSON array of features
- Returns predictions in JSON format

#### 3. REST API (`app.py`)
- Flask-based prediction service
- Health check endpoint for monitoring
- POST /predict endpoint for inference
- Auto-trains model if missing (convenience feature)
- Production-ready with error handling

#### 4. Containerization (`Dockerfile`)
- Lightweight Python 3.12-slim base image
- Multi-stage setup: dependencies → code copy
- Exposes port 5001
- Runs Flask app on container start

#### 5. CI/CD Pipeline (`.github/workflows/ci.yml`)
- Automated training on push/PR to main
- Matrix testing across Python 3.11 and 3.12
- Artifact upload for model versioning
- Ensures reproducibility across environments

### Technology Stack
- **ML Framework**: scikit-learn 1.3.2
- **Web Framework**: Flask 2.3.2
- **Model Serialization**: joblib 1.4.2
- **Data Processing**: pandas 2.2.2, numpy
- **Container**: Docker (Python 3.12-slim)
- **CI/CD**: GitHub Actions

## Quick start (local)
1. Create and activate a venv (example using python 3.13 or 3.11):
    python -m venv .venv
    source .venv/bin/activate

2. Install dependencies:
    pip install --upgrade pip setuptools wheel
    pip install -r requirements.txt

3. Train the model:
    python train.py

4. Run a single prediction from CLI:
    python run_model.py --input "[5.1, 3.5, 1.4, 0.2]"

5. Start the API:
    python src/app.py
   Then test:
    curl -X POST "http://127.0.0.1:5000/predict" -H "Content-Type: application/json" -d '{"features":[5.1,3.5,1.4,0.2]}'
