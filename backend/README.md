# Mental Health Chatbot — Production NLP Layer

A Flask-based NLP backend for a mental health support chatbot. Uses a fine-tuned
**DistilBERT** model to detect emotions (anxiety, depression, anger, sadness, confusion,
suicidal ideation, neutral) from user messages and generates empathetic,
CBT-guided replies with crisis detection.

---

## Project Structure

```
final-year-project/
├── app.py                        # Flask app (Gunicorn-ready)
├── convert_to_onnx.py            # One-time ONNX export script
├── requirements.txt
├── .env.example                  # Copy to .env and fill in values
│
├── config/
│   ├── __init__.py
│   └── settings.py               # All env-based config (no hardcoded secrets)
│
├── models/
│   ├── __init__.py
│   └── emotion_classifier.py     # DistilBERT loader, ONNX inference, confidence gating
│
├── services/
│   ├── __init__.py
│   ├── preprocessor.py           # Text cleaning before model
│   ├── dialogue_manager.py       # Redis-backed session + CBT flow
│   └── nlp_pipeline.py           # Orchestrates preprocessor → model → DM
│
├── middleware/
│   ├── __init__.py
│   ├── rate_limiter.py           # Redis-based per-user rate limiting
│   └── input_validator.py        # Input sanitization + length checks
│
├── utils/
│   ├── __init__.py
│   ├── logger.py                 # Structured JSON logging with latency
│   └── redis_pool.py             # Shared connection pool
│
├── templates/
│   └── index.html                # Chat UI (served by Flask)
│
└── tests/
    ├── __init__.py
    └── test_nlp_pipeline.py      # Unit + integration tests (no heavy deps needed)
```

---

## Pipeline

```
User message
    │
    ▼
InputValidator      → rejects bad input (XSS, empty, too long)
    │
    ▼
RateLimiter         → blocks abuse (Redis sliding window)
    │
    ▼
Preprocessor        → strips URLs, normalises whitespace & repeated chars
    │
    ▼
EmotionClassifier   → DistilBERT / ONNX → emotion label + confidence score
    │
    ▼
DialogueManager     → CBT flow, crisis detection, empathetic reply (Redis-backed)
    │
    ▼
MongoDB logger      → persists conversation record (optional)
    │
    ▼
JSON response       → { reply, emotion, confidence }
```

---

## Quick Start

### 1. Install dependencies
```bash
pip install -r requirements.txt
python -m spacy download en_core_web_sm
```

### 2. Configure environment
```bash
cp .env.example .env
# Edit .env and fill in MONGO_URI, REDIS_URL, SECRET_KEY, MODEL_DIR, etc.
```

### 3. Provide the model

**Option A — Use a pre-trained fine-tuned model:**
Place your fine-tuned DistilBERT checkpoint in `./distilbert_finetuned/`
(must contain `config.json`, `pytorch_model.bin`, `tokenizer_config.json`, `vocab.txt`).

**Option B — Train from scratch:**
```bash
# (Once train.py is written) — see Next Steps
python train.py
```

### 4. Convert to ONNX (recommended for production)
```bash
python convert_to_onnx.py --model-dir ./distilbert_finetuned --output-dir ./onnx_model
# Update .env: ONNX_MODEL_PATH=./onnx_model/model_quantized.onnx
```

### 5. Run locally (development)
```bash
python app.py
# → http://localhost:5000
```

### 6. Run in production
```bash
gunicorn -w 4 -b 0.0.0.0:5000 --timeout 60 app:app
```

---

## Running Tests

All tests are self-contained — heavy dependencies (torch, redis, spacy) are mocked.
No model or Redis instance required.

```bash
# From the project root:
python -m pytest tests/ -v

# Or with unittest:
python -m unittest tests/test_nlp_pipeline.py -v
```

---

## API Endpoints

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/` | Chat UI |
| POST | `/send` | Send a message → `{ reply, emotion, confidence }` |
| POST | `/webhook` | Dialogflow fulfillment webhook |
| GET | `/health` | Health check (Redis + model status) |

### Example `/send` request
```bash
curl -X POST http://localhost:5000/send \
  -H "Content-Type: application/json" \
  -d '{"message": "I feel really anxious and cannot sleep"}'
```

### Example response
```json
{
  "reply": "I can hear that you're feeling anxious. That takes a lot to share. 💙\n\nWould you like to try a calming breathing exercise that might help calm your nervous system?",
  "emotion": "anxiety",
  "confidence": 0.8912
}
```

---

## Infrastructure Requirements

| Service | Purpose | Default |
|---------|---------|---------|
| Redis | Session state + prediction cache + rate limiting | `localhost:6379` |
| MongoDB | Conversation persistence (optional) | `localhost:27017` |

---

## Next Steps

- [ ] Write `train.py` — fine-tune DistilBERT on mental health dataset
- [ ] Source / prepare dataset (e.g. mental health Reddit dataset from Kaggle)
- [ ] Add `Dockerfile` + `docker-compose.yml` for one-command local setup
- [ ] Deploy to cloud (Render / Railway / GCP)
- [ ] Evaluate model — report F1, accuracy per emotion class

## Deploy Backend on Render

### Option 1: Use the included `render.yaml` (recommended)

1. Push this repository to GitHub.
2. In Render, create a **Blueprint** and select the repo.
3. Render will detect `render.yaml` and provision:
   - a Python web service for Flask/Gunicorn
   - a managed Redis instance
4. Add the required environment values in Render:
   - `SECRET_KEY`
   - `MONGO_URI` (optional, but recommended for persistence)
   - `MONGO_DB_NAME`
   - `MONGO_COLLECTION`
   - `FRONTEND_ORIGINS` (comma-separated list of allowed frontend URLs)
5. Deploy and verify the health check at `/health`.

### Option 2: Manual Render service setup

- **Root Directory:** `backend`
- **Build Command:** `pip install -r requirements.txt && python -m spacy download en_core_web_sm`
- **Start Command:** `gunicorn -w 2 -k gthread --threads 4 -b 0.0.0.0:$PORT --timeout 120 app:app`
- **Health Check Path:** `/health`
- **Runtime:** Python 3.11+

Then configure environment variables from `.env.example` in the Render dashboard.
