# 💼 Resume RAG Career Assistant

A retrieval-augmented Q&A web app that lets you upload your resume and ask career questions grounded in that document - powered by **AWS Bedrock** (Llama 3 + Titan Embeddings) and **FAISS**.

Upload a resume → the backend chunks and embeds it into a per-user FAISS vector index → ask a career question → the app retrieves the most relevant chunks and asks an LLM to answer using only that context.

![Preview](Screenshots/Preview.png)

---

## ✨ Features

- **Resume upload & indexing** - accepts `.pdf`, `.docx`, and `.txt` resumes, extracts text, and chunks it with `RecursiveCharacterTextSplitter`.
- **Per-user isolated vector stores** - each `user_id` gets its own FAISS index on disk (keyed by a SHA-256 hash of the user ID), so one user's resume data can never leak into another's search results.
- **Retrieval-augmented answers** - questions are answered by Meta's **Llama 3 70B Instruct** on Bedrock, grounded strictly in the retrieved resume chunks (Titan Text Embeddings for the vector search).
- **FastAPI backend** with a clean service-layer architecture (routers → services → Bedrock/FAISS), a custom exception hierarchy, structured logging, and configurable CORS.
- **Streamlit frontend** - a simple two-pane UI: upload your resume on the left, chat about your career on the right.
- **Optional API-key auth**, health-check endpoint, and Docker healthchecks for both services.
- **Dockerized** end-to-end via `docker-compose`, with a CI pipeline (pytest + ruff + Docker image builds) on GitHub Actions.

---

## 🏗️ Architecture

```
┌─────────────┐      HTTP       ┌───────────────────┐      boto3       ┌────────────────┐
│  Streamlit   │ ──────────────▶ │   FastAPI backend  │ ───────────────▶ │  AWS Bedrock    │
│  frontend    │ ◀────────────── │  (routers/services) │ ◀─────────────── │ (Llama 3, Titan)│
└─────────────┘                 └─────────┬──────────┘                  └────────────────┘
                                           │
                                           ▼
                                 ┌───────────────────┐
                                 │ Per-user FAISS index│
                                 │  data/vector_store/ │
                                 └───────────────────┘
```

---

**Backend layout:**
```
backend/app/
├── main.py                # FastAPI app, CORS, exception handlers
├── core/
│   ├── config.py           # Centralized, typed settings (.env driven)
│   ├── dependencies.py     # API-key auth dependency
│   ├── exceptions.py       # Domain exception hierarchy
│   └── logging_config.py
├── models/schemas.py       # Pydantic request/response contracts
├── routers/
│   ├── health.py           # GET /health
│   └── resume.py           # POST /api/v1/upload_resume, /ask_question
└── services/
    ├── file_parser.py      # PDF/DOCX/TXT text extraction + validation
    ├── vector_store.py     # Per-user FAISS index create/query
    └── bedrock_client.py   # Bedrock invoke_model + prompt template
```
---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Streamlit |
| Backend API | FastAPI, Pydantic, Uvicorn |
| LLM | AWS Bedrock - `meta.llama3-70b-instruct-v1:0` |
| Embeddings | AWS Bedrock - `amazon.titan-embed-text-v1` |
| Vector store | FAISS (`faiss-cpu`), via LangChain |
| File parsing | PyPDF2, python-docx |
| Infra | Docker, Docker Compose, GitHub Actions CI |
| Testing | pytest, httpx |

---

## 🚀 Getting Started

### Prerequisites
- Python 3.11+
- Docker & Docker Compose (for containerized setup)
- An AWS account with Bedrock access enabled for the Llama 3 and Titan Embeddings models, and credentials available to boto3 (e.g. via `.env` or your normal AWS credential chain)

### 1. Configure environment
Create a `.env` file in the project root:
```env
AWS_DEFAULT_REGION=us-east-1
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key

ALLOWED_ORIGINS=http://localhost:8501
REQUIRE_API_KEY=false
APP_API_KEY=

API_URL=http://127.0.0.1:8000
```

### 2. Run with Docker Compose (recommended)
```bash
docker-compose up --build
```
- Backend → http://localhost:8000 (docs at `/docs`)
- Frontend → http://localhost:8501

### 3. Or run locally without Docker
```bash
# Backend
cd backend
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000

# Frontend (in a separate terminal)
cd frontend
pip install -r requirements.txt
streamlit run app.py
```

---

## 📡 API Reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/health` | Liveness/readiness check |
| `POST` | `/api/v1/upload_resume` | Multipart form: `user_id`, `file` (.pdf/.docx/.txt) → chunks, embeds, and indexes the resume |
| `POST` | `/api/v1/ask_question` | JSON: `{ "user_id": "...", "question": "..." }` → answer grounded in that user's resume |

Interactive Swagger docs are available at `/docs` once the backend is running.

## 🧪 Testing
```bash
cd backend
pytest -v
ruff check app
```
CI runs the test suite, lint, and Docker image builds automatically on every push/PR to `main` (see `.github/workflows/ci.yml`).

---

## 📸 Screenshots
| Upload & Indexing | Results |
|---|---|
| ![Upload](Screenshots/Upload_Resume&Indexed.png) | ![Results](Screenshots/Results.png) |

---

## ⚠️ Notes
- API-key auth (`REQUIRE_API_KEY`) is a minimal shared-secret check meant for keeping a demo deployment from being wide open - swap it for real OAuth2/JWT auth for multi-tenant production use.
- Scanned/image-only PDFs won't yield extractable text; use a text-based export instead.
- The FAISS store here is local-disk and per-instance; for a multi-instance production deployment, swap in a managed vector DB (OpenSearch, pgvector, Pinecone) - `vector_store.py`'s `add_resume`/`query` functions are the only integration surface the rest of the app depends on.

---

## 👤 Author

**Chowdri Furkhan** - [GitHub](https://github.com/Chowdri-Furkhan07)

---

## 📄 License

This project is licensed under the MIT License - free to use, modify, and distribute with attribution.
