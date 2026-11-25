# SIH26003 — Adaptive Difficulty ML Service

A FastAPI microservice for the **SIH26003** cognitive gaming & memory assistance platform. It serves a trained Random Forest model (`v2.0`) that recommends the next difficulty level (`1`–`5`) and a confidence score based on a player's recent gameplay telemetry.

## Project Structure

```text
adaptive_difficulty/
├── .gitignore
├── README.md
├── api/
│   ├── config.py          # Resolves model path
│   ├── main.py            # FastAPI app and routes (GET /, POST /predict)
│   ├── model_loader.py    # Loads the serialized joblib model
│   ├── predictor.py       # Runs inference, clamps output to [1, 5], computes confidence
│   ├── schemas.py         # Pydantic request/response validation schemas
│   └── test_client.py     # Sample HTTP client script
└── models/
    ├── sih26003_adaptive_difficulty_v2.pkl
    └── sih26003_model_metadata.json
```

## Model Summary

From [`models/sih26003_model_metadata.json`](models/sih26003_model_metadata.json):

| Property | Value |
| --- | --- |
| Model Name | SIH26003 Adaptive Difficulty Model (`v2.0`) |
| Algorithm | Random Forest |
| Target | `next_difficulty` (`1` to `5`) |
| Adaptive Thresholds | `decrease_below: 0.549`, `increase_at_or_above: 0.632` |
| Test Accuracy | `0.891` |
| MAE | `0.109` |
| Group CV Accuracy | `0.8824` |

## Setup & Run

1. Create and activate a Python virtual environment:

   ```bash
   python -m venv venv
   # Windows (PowerShell)
   .\venv\Scripts\Activate.ps1
   # Linux / macOS
   source venv/bin/activate
   ```

2. Install runtime dependencies:

   ```bash
   pip install fastapi uvicorn pydantic pandas scikit-learn==1.6.1 joblib requests
   ```

   > **Note:** The saved model artifact (`sih26003_adaptive_difficulty_v2.pkl`) was serialized with `scikit-learn==1.6.1`; using a matching version avoids unpickling incompatibilities.

3. Start the FastAPI server from the `api/` directory:

   ```bash
   cd api
   uvicorn main:app --host 127.0.0.1 --port 8000
   ```

   Interactive OpenAPI docs will be available at `http://127.0.0.1:8000/docs`.

## API Endpoints

### `GET /`

Health check confirming the service is running and the model is loaded.

**Response:**

```json
{
  "service": "SIH26003 Adaptive Difficulty ML Service",
  "status": "running",
  "model_loaded": true
}
```

### `POST /predict`

Predicts the recommended `next_difficulty` (`1`–`5`) and `confidence` score.

**Request Schema (`PredictionRequest`):**

| Field | Type | Constraints |
| --- | --- | --- |
| `game_type` | `string` | e.g. `"memory"`, `"attention"` |
| `cognitive_domain` | `string` | e.g. `"memory"` |
| `current_difficulty` | `integer` | `1 <= current_difficulty <= 5` |
| `accuracy` | `float` | `0.0 <= accuracy <= 1.0` |
| `response_time` | `float` | `>= 0.0` |
| `attempts` | `integer` | `>= 0` |
| `hints_used` | `integer` | `>= 0` |
| `recent_accuracy` | `float` | `0.0 <= recent_accuracy <= 1.0` |
| `recent_response_time` | `float` | `>= 0.0` |
| `accuracy_trend` | `float` | Any float |
| `response_time_trend` | `float` | Any float |
| `consecutive_successes` | `integer` | `>= 0` |

**Example Request:**

```bash
curl -X POST http://127.0.0.1:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "game_type": "memory",
    "cognitive_domain": "memory",
    "current_difficulty": 3,
    "accuracy": 0.85,
    "response_time": 4.0,
    "attempts": 1,
    "hints_used": 0,
    "recent_accuracy": 0.82,
    "recent_response_time": 4.2,
    "accuracy_trend": 0.03,
    "response_time_trend": -0.1,
    "consecutive_successes": 3
  }'
```

**Example Response:**

```json
{
  "next_difficulty": 4,
  "confidence": 0.685
}
```

> **Note on `api/test_client.py`:** The sample script sends `"accuracy": 85` and `"recent_accuracy": 82` (0–100 scale), which fails Pydantic's `[0.0, 1.0]` validation (`422 Unprocessable Entity`) unless normalized to `0.85` and `0.82`.
